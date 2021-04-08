# single-spa

[v5.9.1](https://github.com/single-spa/single-spa/releases/tag/v5.9.1)

将多个 spa 集成到一个 spa 中。原理上很简单，就是提供一个 dom 节点，把 spa 挂载到这个节点上。但是实际应用时会有很多问题，比如说如何防止`window`被污染，如何隔离样式，某些全局的资源(style，script)如何引入。

`single-spa`更像是一个大的框架，它为每个 spa 提供了激活路由、生命周期的钩子，但把如何引入 spa 留给了使用者决定。

## Usage

```js
// single-spa-config.js
import { registerApplication, start } from "single-spa";

// Simple usage
registerApplication(
  "app2",
  () => import("src/app2/main.js"),
  (location) => location.pathname.startsWith("/app2"),
  { some: "value" }
);

// Config with more expressive API
registerApplication({
  name: "app1",
  app: () => import("src/app1/main.js"),
  activeWhen: "/app1",
  customProps: {
    some: "value",
  },
});

start();
```

主要就是先调用`registerApplication`来注册`app`，然后调用`start`来启动应用。`registerApplication`有两种调用方法，主要体现在传入的参数。我们先看第一种，第二种方法同理。第一个参数是用来区分不同的`app`，第二个用来加载`app`，这里说的加载是拉取`app`的代码，然后从中获取生命周期方法。第三个参数可以是字符串，也可以是函数，用来根据路由判断何时激活`app`。最后一个是一个 Object，也可以是返回一个 Object 的函数，最后会传入生命周期方法中。

## Source code

首先来看`registerApplication`方法

```js
const apps = [];
export function registerApplication(
  appNameOrConfig,
  appOrLoadApp,
  activeWhen,
  customProps
) {
  const registration = sanitizeArguments(
    appNameOrConfig,
    appOrLoadApp,
    activeWhen,
    customProps
  );
  // ...
  apps.push(
    assign(
      {
        loadErrorTime: null,
        status: NOT_LOADED,
        parcels: {},
        devtools: {
          overlays: {
            options: {},
            selectors: [],
          },
        },
      },
      registration
    )
  );
  if (isInBrowser) {
    ensureJQuerySupport();
    reroute();
  }
}
```

调用后，会创建一个`app`，并放入数组中保存。接着调用`reroute`。注意`app`的内部结构，以及当前状态`NOT_LOADED`。

核心的方法`reroute`根据当前框架是否启动(`isStarted()`，在调用过`start()`后，返回 `true`)来决定加载`app`，还是切换`app`。

一般，我们会先`registerApplication`，后`start`。所以，会先加载`app`。

```js
function loadApps() {
  return Promise.resolve().then(() => {
    const loadPromises = appsToLoad.map(toLoadPromise);

    return (
      Promise.all(loadPromises)
        .then(callAllEventListeners)
        // there are no mounted apps, before start() is called, so we always return []
        .then(() => [])
        .catch((err) => {
          callAllEventListeners();
          throw err;
        })
    );
  });
}
```

`appsToLoad`是一个数组，存着需要加载的`app`。加载的逻辑藏在`toLoadPromise`函数中。

首先，将`app`的状态从`NOT_LOADED`或`LOAD_ERROR`，变为`LOADING_SOURCE_CODE`。接着调用上面我们传入`registerApplication`函数中加载`app`的函数。如果成功了，就将生命周期的回调存入`app`中，并将状态改为`NOT_BOOTSTRAPPED`。如果失败了，将状态改为`LOAD_ERROR`或者`SKIP_BECAUSE_BROKEN`。然后将`app`返回。最后不管成功失败，都调用`callAllEventListeners`。我们后面再提为什么。

在我们注册完`app`后，调用`start`

```js
let started = false;

export function start(opts) {
  started = true;
  if (opts && opts.urlRerouteOnly) {
    setUrlRerouteOnly(opts.urlRerouteOnly);
  }
  if (isInBrowser) {
    reroute();
  }
}
```

可以发现，这里又调用了`reroute`。这次的调用时为了激活或者切换`app`。

`reroute`函数中的片段

```js
if (isStarted()) {
  appChangeUnderway = true;
  appsThatChanged = appsToUnload.concat(appsToLoad, appsToUnmount, appsToMount);
  return performAppChanges();
}
```

`appsToUnload`,`appsToLoad`,`appsToUnmount`,`appsToMount`分别表示对应状态的`app`的数组。`performAppChanges`函数中，要处理这四个数组，还是有些繁杂的。我们把注意力放在这四个数组的处理上，

```js
function performAppChanges() {
  return Promise.resolve().then(() => {
    // ...
    const unloadPromises = appsToUnload.map(toUnloadPromise);
    const unmountUnloadPromises = appsToUnmount
      .map(toUnmountPromise)
      .map((unmountPromise) => unmountPromise.then(toUnloadPromise));
    const allUnmountPromises = unmountUnloadPromises.concat(unloadPromises);
    const unmountAllPromise = Promise.all(allUnmountPromises);
    // ...
    /* We load and bootstrap apps while other apps are unmounting, but we
     * wait to mount the app until all apps are finishing unmounting
     */
    const loadThenMountPromises = appsToLoad.map((app) => {
      return toLoadPromise(app).then((app) =>
        tryToBootstrapAndMount(app, unmountAllPromise)
      );
    });
    /* These are the apps that are already bootstrapped and just need
     * to be mounted. They each wait for all unmounting apps to finish up
     * before they mount.
     */
    const mountPromises = appsToMount
      .filter((appToMount) => appsToLoad.indexOf(appToMount) < 0)
      .map((appToMount) => {
        return tryToBootstrapAndMount(appToMount, unmountAllPromise);
      });
    return unmountAllPromise
      .catch((err) => {
        callAllEventListeners();
        throw err;
      })
      .then(() => {
        /* Now that the apps that needed to be unmounted are unmounted, their DOM navigation
         * events (like hashchange or popstate) should have been cleaned up. So it's safe
         * to let the remaining captured event listeners to handle about the DOM event.
         */
        callAllEventListeners();
        return Promise.all(loadThenMountPromises.concat(mountPromises))
          .catch((err) => {
            pendingPromises.forEach((promise) => promise.reject(err));
            throw err;
          })
          .then(finishUpAndReturn);
      });
  });
}
```

首先是处理`appsToUnload`，由`toUnloadPromise`函数负责。这里会先将`app`的状态变为`UNLOADING`,然后调用`app.unload`生命周期方法，如果成功了，就将状态置为`NOT_LOADED`或`SKIP_BECAUSE_BROKEN`。

`appsToUnmount`，由`toUnmountPromise`负责。将状态置为`UNMOUNTING`，调用`app.unmount`,如果成功，将状态置为`NOT_MOUNTED`，否则`SKIP_BECAUSE_BROKEN`。这里首先还会先处理`app`的子`parcels`。我们后面会提到。

当所有需要`unmount`，`unload`的`app`处理完后，接着处理需要`bootstrap`，`mount`的`app`。

`toBootstrapPromise`函数，将状态置为`BOOTSTRAPPING`，然后执行`app.bootstrap`，如果成功，将状态变为`NOT_MOUNTED`。失败后变为`SKIP_BECAUSE_BROKEN`。

`toMountPromise`， 执行`app.mount`，成功后将状态置为`MOUNTED`，否则将状态置为`SKIP_BECAUSE_BROKENSKIP_BECAUSE_BROKEN`。

最后调用`finishUpAndReturn`，将所有处于`MOUNTED`状态的`app`的`name`返回。

再回过头来看看`reroute`函数入参中的`pendingPromises`，理解它还要结合`peopleWaitingOnAppChange`这个数组。

```js
let appChangeUnderway = false,
  peopleWaitingOnAppChange = [];
export function reroute(pendingPromises = [], eventArguments) {
  if (appChangeUnderway) {
    return new Promise((resolve, reject) => {
      peopleWaitingOnAppChange.push({
        resolve,
        reject,
        eventArguments,
      });
    });
  }
  // ...
  function performAppChanges() {
    // ...
    return (
      unmountAllPromise
        // ...
        .then(() => {
          // ...
          return Promise.all(loadThenMountPromises.concat(mountPromises))
            .catch((err) => {
              pendingPromises.forEach((promise) => promise.reject(err));
              throw err;
            })
            .then(finishUpAndReturn);
        })
    );
  }
  function finishUpAndReturn() {
    const returnValue = getMountedApps();
    pendingPromises.forEach((promise) => promise.resolve(returnValue));
    // ...
    appChangeUnderway = false;

    if (peopleWaitingOnAppChange.length > 0) {
      // ...
      const nextPendingPromises = peopleWaitingOnAppChange;
      peopleWaitingOnAppChange = [];
      reroute(nextPendingPromises);
    }
    return returnValue;
  }
}
```

去掉其他代码后，可以发现`pendingPromises`这个入参是用来存储被延迟执行的`reroute`调用。当执行`app`切换时，`appChangeUnderway=true`，后续的调用会压入到`peopleWaitingOnAppChange`数组中。当本次`reroute`执行完成时，会将`peopleWaitingOnAppChange`传入`reroute`再次进行调用。而在本次的执行过程中，`pendingPromises`会根据情况进行`resolve`或`reject`。

### Monkey Patch

`reroute`这个方法会在`registerApplication`和`start`的时候调用。但只在这两个地方调用，并不能满足实际需要。当我们在浏览器中前进，后退，跳转时，`reroute`都应该被调用，来根据需要切换`app`。

所以，`single-spa`做了以下处理，首先

```js
/* We capture navigation event listeners so that we can make sure
 * that application navigation listeners are not called until
 * single-spa has ensured that the correct applications are
 * unmounted and mounted.
 */
const capturedEventListeners = {
  hashchange: [],
  popstate: [],
};
export const routingEventsListeningTo = ["hashchange", "popstate"];
// Monkeypatch addEventListener so that we can ensure correct timing
const originalAddEventListener = window.addEventListener;
// ...
window.addEventListener = function (eventName, fn) {
  if (typeof fn === "function") {
    if (
      routingEventsListeningTo.indexOf(eventName) >= 0 &&
      !find(capturedEventListeners[eventName], (listener) => listener === fn)
    ) {
      capturedEventListeners[eventName].push(fn);
      return;
    }
  }
  return originalAddEventListener.apply(this, arguments);
};
```

对于监听`hashchange`，`popstate`事件都拦截下来，存储到本地。

接着，监听路由变化，

```js
// We will trigger an app change for any routing events.
window.addEventListener("hashchange", urlReroute);
window.addEventListener("popstate", urlReroute);

function urlReroute() {
  reroute([], arguments);
}
```

注意这个监听函数，内部调用了`reroute([], arguments)`。马上我们就可以前后串联在一起。

对于`history.pushState`，`history.replaceState`，也做 patch

```js
window.history.pushState = patchedUpdateState(
  window.history.pushState,
  "pushState"
);
window.history.replaceState = patchedUpdateState(
  window.history.replaceState,
  "replaceState"
);
function patchedUpdateState(updateState, methodName) {
  return function () {
    const urlBefore = window.location.href;
    const result = updateState.apply(this, arguments);
    const urlAfter = window.location.href;
    if (!urlRerouteOnly || urlBefore !== urlAfter) {
      if (isStarted()) {
        // fire an artificial popstate event once single-spa is started,
        // so that single-spa applications know about routing that
        // occurs in a different application
        window.dispatchEvent(
          createPopStateEvent(window.history.state, methodName)
        );
      } else {
        // do not fire an artificial popstate event before single-spa is started,
        // since no single-spa applications need to know about routing events
        // outside of their own router.
        reroute([]);
      }
    }
    return result;
  };
}
```

当调用`pushState`，`replaceState`时，如果`location.href`发生改变，会发起一个`PopStateEvent`事件，我们知道，原本这两个操作时不会发起这个事件的。现在这样处理，是为了`spa`路由变化时，让`single-spa`也知道，因为上面的代码中，也监听了`popstate`事件，触发后会调用`reroute`。

现在，我们再把这些`patch`，监听函数和`reroute`联系在一起。`single-spa`将所有监听路由变化的函数保存在内部。当有路由变化时，无论是哪种形式，`single-spa`都会知道，并调用`reroute`。在`app`加载完毕，后者是完成切换时，`reroute`内部会调用`callAllEventListeners`。

```js
/* We need to call all event listeners that have been delayed because they were
 * waiting on single-spa. This includes haschange and popstate events for both
 * the current run of performAppChanges(), but also all of the queued event listeners.
 * We want to call the listeners in the same order as if they had not been delayed by
 * single-spa, which means queued ones first and then the most recent one.
 */
function callAllEventListeners() {
  pendingPromises.forEach((pendingPromise) => {
    callCapturedEventListeners(pendingPromise.eventArguments);
  });

  callCapturedEventListeners(eventArguments);
}
```

接着`callAllEventListeners`内部调用`callCapturedEventListeners`，执行所有储存起来的监听路由变化的函数。

至此，`single-spa`完成了它的路由功能。

### unregisterApplication

既然可以注册`app`，那也可以取消注册。只需要调用`unregisterApplication`。

```js
export function unregisterApplication(appName) {
  if (apps.filter((app) => toName(app) === appName).length === 0) {
    // ...
  }
  return unloadApplication(appName).then(() => {
    const appIndex = apps.map(toName).indexOf(appName);
    apps.splice(appIndex, 1);
  });
}
```

### Parcel

`parcel`和上面说的`app`很像，都有自己的生命周期，也需要一个挂载的 dom 节点。主要的不同是，`parcel`不会响应路由变化，只是作为一个组件存在。通常用于不同技术栈的`spa`之间共享组件。

挂载`parcel`有两种方法，`mountRootParcel`，`mountParcel`。唯一不同的是存储的位置。

```js
const rootParcels = { parcels: {} };
// This is a public api, exported to users of single-spa
export function mountRootParcel() {
  return mountParcel.apply(rootParcels, arguments);
}

export function mountParcel(config, customProps) {
  const owningAppOrParcel = this;
  // ...
  // Add to owning app or parcel
  owningAppOrParcel.parcels[id] = parcel;
  // ...
}
```

`mountRootParcel`把自己存在`rootParcels`对象上，而`mountParcel`则是存在`app.parcels`上。好处是可以随着`app`的`unmount`一起卸载，无需手动。

`parcel`的`load`，`bootstrap`，`mount`，`unmount`和`app`的逻辑都大同小异。多出的一个生命周期回调是`update`。

```js
if (config.update) {
  parcel.update = flattenFnArray(config, "update");
  externalRepresentation.update = function (customProps) {
    parcel.customProps = customProps;
    return promiseWithoutReturnValue(toUpdatePromise(parcel));
  };
}
```

`toUpdatePromise`的逻辑也就是把`parcel`的状态置为`UPDATING`，然后调用`parcel.update`。成功后再变回`MOUNTED`，否则变为`SKIP_BECAUSE_BROKEN`。

值得注意的是，一旦调用`mountParcel`，生成的`parcel`会立即`load`，`bootstrap`，`mount`。

## Conclusion

`single-spa`提供了一种方式，将多个独立的`spa`连接在一起。但若要实际使用，还有许多事情要解决。

1. 如何引入 css
2. 如何引入 spa 外部的 scripts，styles，又不互相干扰
3. 如何解决`window`可能被污染的问题
4. 如何接入已经存在的 spa
