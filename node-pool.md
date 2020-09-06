# node-pool

[master](https://github.com/coopernurse/node-pool) 分支

33349aea4085ee632fe8610c3dde0ea240a14c7a

> Generic resource pool with Promise based API. Can be used to reuse or throttle usage of expensive resources such as database connections.

简单地说就是用来管理昂贵的资源，比如说数据库连接，puppeteer 创建的浏览器引用等等。这些资源每次创建都是有代价的，要么是占用 io，要么是占用 cpu。一种常用的策略是，创建一个资源池，或者叫做连接池。池中通常有固定数量的资源可供复用。超过资源数量的消费者必须排队等待资源释放，才能使用。这个库就是实现了这种资源池。

## 用法

api 不多，首先是 createPool，调用这个工厂方法会返回一个资源池对象。该方法接收一个 factory 对象，里面包含创建资源，销毁资源等方法。还接收一个配置对象。

在返回的资源池对象上调用 acquire，返回一个 promise，等这个 promise resolved 以后，在回调中可以拿到一个资源对象。

release 方法，将获取到的资源对象传入，放回资源池中。这个方法也返回一个 promise。

比较常用的就是这些。

## 设计思路

首先是管理资源，然后是管理对资源的请求。

先看 node-pool 如果管理资源

```js
/**
 * A queue/stack of pooledResources awaiting acquisition
 * TODO: replace with LinkedList backed array
 * @type {Deque}
 */
this._availableObjects = new Deque();

// ...

_addPooledResourceToAvailableObjects(pooledResource) {
	pooledResource.idle();
	if (this._config.fifo === true) {
		this._availableObjects.push(pooledResource);
	} else {
		this._availableObjects.unshift(pooledResource);
	}
}
// ...
  /**
   * Attempt to move an available resource to a waiting client
   * @return {Boolean} [description]
   */
_dispatchResource() {
    if (this._availableObjects.length < 1) {
      return false;
    }

    const pooledResource = this._availableObjects.shift();
    this._dispatchPooledResourceToNextWaitingClient(pooledResource);
    return false;
}
```

创建的资源会被放到一个双端队列中，之所以使用双端队列，有两个原因。一是，这个双端队列底层使用的是双向链表，插值的效率优于数组。二是，用户可以通过配置决定资源是否为 fifo（先进先出），从代码中可以看到每次取资源都是从头部取，而插入则是根据用户配置来决定插入到头部还是尾部。

资源池里保存的资源对象使用 PooledResource 类封装过。

```js
const wrappedFactoryPromise = this._Promise
  .resolve(factoryPromise)
  .then((resource) => {
    const pooledResource = new PooledResource(resource);
    this._allObjects.add(pooledResource);
    this._addPooledResourceToAvailableObjects(pooledResource);
  });
```

这个对象里保存了和资源的各种相关时间，资源的状态。

```js
class PooledResource {
  constructor(resource) {
    this.creationTime = Date.now();
    this.lastReturnTime = null;
    this.lastBorrowTime = null;
    this.lastIdleTime = null;
    this.obj = resource;
    this.state = PooledResourceStateEnum.IDLE;
  }
  // ...
}
```

资源的状态总共有 5 种，在使用时，会根据条件切换状态。

```js
const PooledResourceStateEnum = {
  ALLOCATED: "ALLOCATED", // In use
  IDLE: "IDLE", // In the queue, not in use.
  INVALID: "INVALID", // Failed validation
  RETURNING: "RETURNING", // Resource is in process of returning
  VALIDATION: "VALIDATION", // Currently being tested
};
```

那么资源是如何创建的呢？node-pool 在初始化的时候，会调用\_ensureMinimum 方法，来确保 pool 中存在一定数量的资源。这个数量可以通过配置修改。

```js
  _ensureMinimum() {
    if (this._draining === true) {
      return;
    }
    const minShortfall = this._config.min - this._count;
    for (let i = 0; i < minShortfall; i++) {
      this._createResource();
    }
  }
```

在具体创建资源时，会调用创建 pool 对象时，传入的 factory 对象中的 create 方法。在创建成功后，会将资源对象放入队列中，然后调用\_dispense，来分配资源。

```js
  _createResource() {
    // An attempt to create a resource
    const factoryPromise = this._factory.create();
    const wrappedFactoryPromise = this._Promise
      .resolve(factoryPromise)
      .then(resource => {
        const pooledResource = new PooledResource(resource);
        this._allObjects.add(pooledResource);
        this._addPooledResourceToAvailableObjects(pooledResource);
      });

    this._trackOperation(wrappedFactoryPromise, this._factoryCreateOperations)
      .then(() => {
        this._dispense();
        // Stop bluebird complaining about this side-effect only handler
        // - a promise was created in a handler but was not returned from it
        // https://goo.gl/rRqMUw
        return null;
      })
      .catch(reason => {
        this.emit(FACTORY_CREATE_ERROR, reason);
        this._dispense();
      });
  }
```

接下来详细说明 node-pool 是如何管理对资源的请求的。

在调用 acquire 方法时，会将请求用 ResourceRequest 类封装后，装入\_waitingClientsQueue 队列。然后也调用\_dispense 分配资源。这个\_waitingClientsQueue 是一个优先队列，配合 acquire 方法的入参 priority，可以对排队的请求设置优先级。这里要注意 acquire 方法的返回值，resourceRequest.promise，这个 promise 会在\_dispense 方法中于合适的时间被触发。现在先来看看 ResourceRequest 这个类。

```js
	/**
		* Holds waiting clients
		* @type {PriorityQueue}
		*/
	this._waitingClientsQueue = new PriorityQueue(this._config.priorityRange);
	// ...
	acquire(priority) {
		// ...
		const resourceRequest = new ResourceRequest(
		this._config.acquireTimeoutMillis,
		this._Promise
		);
		this._waitingClientsQueue.enqueue(resourceRequest, priority);
		this._dispense();

		return resourceRequest.promise;
	}
```

ResourceRequest 这个类继承了 Deferred 类。顾名思义，Deferred 表示一个延迟对象。从实现上来看，只是对原本的 promise 进行了改造，保留其状态，将 resolve 和 reject 方法传到了外部。这样就可以在外部来触发 promise。因此，上面的 acquire 方法返回的这个 resourceRequest.promise，就可以在\_dispense 方法中触发了。ResourceRequest 除了这些，还额外带有一个定时器，用以实现请求资源超时。在用户设置了 acquireTimeoutMillis 属性的情况下，会对超过这个时限的请求做 reject 处理。

```js
/**
 * This is apparently a bit like a Jquery deferred, hence the name
 */

class Deferred {
  constructor(Promise) {
    this._state = Deferred.PENDING;
    this._resolve = undefined;
    this._reject = undefined;

    this._promise = new Promise((resolve, reject) => {
      this._resolve = resolve;
      this._reject = reject;
    });
  }
  // ...
}

/**
 * Wraps a users request for a resource
 * Basically a promise mashed in with a timeout
 * @private
 */
class ResourceRequest extends Deferred {
  /**
   * [constructor description]
   * @param  {Number} ttl     timeout
   */
  constructor(ttl, Promise) {
    super(Promise);
    this._creationTimestamp = Date.now();
    this._timeout = null;

    if (ttl !== undefined) {
      this.setTimeout(ttl);
    }
  }
  // ...
}
```

接着就是\_dispense 方法了，这个方法连接了资源的创建和请求。创建资源的方法\_createResource，和获取资源的方法 acquire，两者都会调用\_dispense。由这个方法来分配资源。注意如果已有的资源不足，并且还可以新创建资源的情况下，会调用\_createResource 方法。这样形成了一个递归调用。如果用户设置了 testOnBorrow，那么先对资源做 test，然后分配。否则直接分配。

```js
/**
   * Attempt to resolve an outstanding resource request using an available resource from
   * the pool, or creating new ones
   *
   * @private
   */
  _dispense() {
	// ...
    for (let i = 0; actualNumberOfResourcesToCreate > i; i++) {
      this._createResource();
    }

    // If we are doing test-on-borrow see how many more resources need to be moved into test
    // to help satisfy waitingClients
    if (this._config.testOnBorrow === true) {
      // ...
      for (let i = 0; actualNumberOfResourcesToMoveIntoTest > i; i++) {
        this._testOnBorrow();
      }
    }

    // if we aren't testing-on-borrow then lets try to allocate what we can
    if (this._config.testOnBorrow === false) {
	  // ...
      for (let i = 0; actualNumberOfResourcesToDispatch > i; i++) {
        this._dispatchResource();
      }
    }
  }
```

二者最终都会调用\_dispatchPooledResourceToNextWaitingClient 方法。

```js
  /**
   * Dispatches a pooledResource to the next waiting client (if any) else
   * puts the PooledResource back on the available list
   * @param  {PooledResource} pooledResource [description]
   * @return {Boolean}                [description]
   */
  _dispatchPooledResourceToNextWaitingClient(pooledResource) {
    const clientResourceRequest = this._waitingClientsQueue.dequeue();
	// ...
    pooledResource.allocate();
    clientResourceRequest.resolve(pooledResource.obj);
    return true;
  }
```

从等待队列中获取一个请求，再将资源传入。clientResourceRequest.resolve(pooledResource.obj)这个调用，就和 acquire 方法中返回的那个 promise 联系了起来。

接下来还有两个和 acquire 方法对应的两个方法 release 释放资源，destroy 销毁资源。

```js
  /**
   * Return the resource to the pool when it is no longer required.
   *
   * @param {Object} resource
   *   The acquired object to be put back to the pool.
   */
  release(resource) {
    // check for an outstanding loan
    const loan = this._resourceLoans.get(resource);
	// ...
    loan.resolve();
    const pooledResource = loan.pooledResource;

    pooledResource.deallocate();
    this._addPooledResourceToAvailableObjects(pooledResource);

    this._dispense();
    return this._Promise.resolve();
  }

    /**
   * Request the resource to be destroyed. The factory's destroy handler
   * will also be called.
   *
   * This should be called within an acquire() block as an alternative to release().
   *
   * @param {Object} resource
   *   The acquired resource to be destoyed.
   */
  destroy(resource) {
    // check for an outstanding loan
    const loan = this._resourceLoans.get(resource);
	// ...
    const pooledResource = loan.pooledResource;

    pooledResource.deallocate();
    this._destroy(pooledResource);

    this._dispense();
    return this._Promise.resolve();
  }
```

这两个方法的实现很相似，它们最后都调用了\_dispense，来重新处理分配资源。

node-pool 还有一个有趣的功能就是 Idle Object Eviction。

> The pool has an evictor (off by default) which will inspect idle items in the pool and destroy them if they are too old.

如果配置了 evictionRunIntervalMillis，那么内部会启动一个定时器，来检测长时间没有被使用的资源，然后将其回收。

```js
_scheduleEvictorRun() {
	// Start eviction if set
	if (this._config.evictionRunIntervalMillis > 0) {
	// @ts-ignore
	this._scheduledEviction = setTimeout(() => {
		this._evict();
		this._scheduleEvictorRun();
	}, this._config.evictionRunIntervalMillis);
	}
}

_evict() {
	// ...
	for (let testsHaveRun = 0; testsHaveRun < testsToRun; ) {
		// ...
		const shouldEvict = this._evictor.evict(
			evictionConfig,
			resource,
			this._availableObjects.length
		);
		testsHaveRun++;

		if (shouldEvict === true) {
			// take it out of the _availableObjects list
			this._evictionIterator.remove();
			this._destroy(resource);
		}
	}
}
```

最后就是清理资源了，node-pool 提供了两个方法，drain 和 clear 配合起来使用。

```js
/**
* Disallow any new acquire calls and let the request backlog dissapate.
* The Pool will no longer attempt to maintain a "min" number of resources
* and will only make new resources on demand.
* Resolves once all resource requests are fulfilled and all resources are returned to pool and available...
* Should probably be called "drain work"
* @returns {Promise}
*/
drain() {
	this._draining = true;
	return this.__allResourceRequestsSettled()
		.then(() => {
			return this.__allResourcesReturned();
		})
		.then(() => {
			this._descheduleEvictorRun();
		});
}
```

drain 调用后，acquire 就不能调用了，否则会报错。\_ensureMinimum 方法调用后也没有效果。这就使得 pool 不会再自动维持一个最小的资源数。并且，drain 方法会等到所有的资源请求都完成并返回资源池后，终止 Eviction 的定时任务。

在 drain 方法完成后，就可以调用 clear 方法，清除所有的资源。

```js
  /**
   * Forcibly destroys all available resources regardless of timeout.  Intended to be
   * invoked as part of a drain.  Does not prevent the creation of new
   * resources as a result of subsequent calls to acquire.
   *
   * Note that if factory.min > 0 and the pool isn't "draining", the pool will destroy all idle resources
   * in the pool, but replace them with newly created resources up to the
   * specified factory.min value.  If this is not desired, set factory.min
   * to zero before calling clear()
   *
   */
  clear() {
    const reflectedCreatePromises = Array.from(
      this._factoryCreateOperations
    ).map(reflector);

    // wait for outstanding factory.create to complete
    return this._Promise.all(reflectedCreatePromises).then(() => {
      // Destroy existing resources
      // @ts-ignore
      for (const resource of this._availableObjects) {
        this._destroy(resource);
      }
      const reflectedDestroyPromises = Array.from(
        this._factoryDestroyOperations
      ).map(reflector);
      return reflector(this._Promise.all(reflectedDestroyPromises));
    });
  }
```

这两个方法最好连在一起使用

```js
const p = pool.drain().then(function () {
  return pool.clear();
});
```
