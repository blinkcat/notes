# immer

版本[v9.0.6](https://github.com/immerjs/immer/tree/v9.0.6)

以更新 mutable 数据的方式更新 immutable 数据。

## usage

```js
import produce from "immer";

const baseState = [
  {
    title: "Learn TypeScript",
    done: true,
  },
  {
    title: "Try Immer",
    done: false,
  },
];

const nextState = produce(baseState, (draftState) => {
  draftState.push({ title: "Tweet about it" });
  draftState[1].done = true;
});
```

使用方式非常简单，极大地降低了更新复杂的 immutable 对象的复杂度。

## How it works

`immer`使用起来很简单，但是其内部实现却是一点都不简单。经过九个版本的迭代，兼容了不支持`Proxy`特性的 es5 环境、支持`Map`、`Set`、`Class`等对象，又处理了大量边界情况，使得内部逻辑分支众多，一下不好理出头绪来。再加上 immer 还支持`patch`功能，使得代码更加复杂。

所以我们先从基本的功能开始看代码，当前只需要关心支持`Proxy`的情况，只考虑 Object 和 Array，暂时先忽略 patch 功能。

immer 的大致原理是根据我们传入`produce`函数的第一个参数，一个对象或者是数组，利用 es6 的 Proxy 生成这个参数的代理对象，作为第二个参数，一个函数的入参 draftState。这时候，如果我们读取 draftState 上的属性，如果这个属性也是对象或者数组，那么做个属性值也会被 immer 生成一个 Proxy 对象代理。这样当我们做 add 操作、set 操作、delete 操作的时候，可以知道用户具体做了什么操作。并且，将这些操作作用在原对象的一个浅拷贝中，而不是原对象本身。最后遍历这些 Proxy 对象，找到这浅拷贝对象，替换原始对象，生成一个新的对象。

我们常用的`produce`函数，其实是`Immer`类中的方法，

```ts
const immer = new Immer();
// ...
export const produce: IProduce = immer.produce;
export default produce;
```

这个类中实现了很多常用的功能，这里我们先只看`produce`函数的逻辑。

```ts
export class Immer implements ProducersFns {
  produce: IProduce = (base: any, recipe?: any, patchListener?: any) => {
    // ...
    let result;

    // Only plain objects, arrays, and "immerable classes" are drafted.
    if (isDraftable(base)) {
      const scope = enterScope(this);
      const proxy = createProxy(this, base, undefined);
      let hasError = true;
      try {
        result = recipe(proxy);
        hasError = false;
      } finally {
        // finally instead of catch + rethrow better preserves original stack
        if (hasError) revokeScope(scope);
        else leaveScope(scope);
      }
      // ...
      usePatchesInScope(scope, patchListener);
      return processResult(result, scope);
    } else if (!base || typeof base !== "object") {
      // ...
    } else die(21, base);
  };
}
```

乍看一下很复杂，实际也是。

首先，immer 只会只对几种数据生成代理，这里是通过`isDraftable`函数判断，传入的 base 对象是否是 immer 可以代理的。

```ts
export const DRAFTABLE: unique symbol = hasSymbol
  ? Symbol.for("immer-draftable")
  : ("__$immer_draftable" as any);
// ...
export function isDraftable(value: any): boolean {
  if (!value) return false;
  return (
    isPlainObject(value) ||
    Array.isArray(value) ||
    !!value[DRAFTABLE] ||
    !!value.constructor[DRAFTABLE] ||
    isMap(value) ||
    isSet(value)
  );
}
```

也就是说，只有 Object、Array、Map、Set、以及自身或者是构造函数上具有`DRAFTABLE`属性的对象，才可以被 immer 代理。这些几乎涵盖了所有我们在状态管理中能用到的数据类型。

因为`produce`函数可以嵌套使用，每个 produce 函数中，只处理自己生成的代理对象，所以这里引入了`Scope`概念。一个 scope 对应一个 produce 函数，所以在 produce 函数中需要调用`enterScope`函数，创建本次 produce 过程的 scope 对象。

```ts
function createScope(
  parent_: ImmerScope | undefined,
  immer_: Immer
): ImmerScope {
  return {
    drafts_: [],
    parent_,
    immer_,
    // Whenever the modified draft contains a draft from another scope, we
    // need to prevent auto-freezing so the unowned draft can be finalized.
    canAutoFreeze_: true,
    unfinalizedDrafts_: 0,
  };
}
// ...
let currentScope: ImmerScope | undefined;
// ...
export function enterScope(immer: Immer) {
  return (currentScope = createScope(currentScope, immer));
}
```

这里可以看出，scope 对象是一个反向的链表。

在创建完 scope 之后，就开始调用`createProxy`创建代理对象。

```ts
export function createProxy<T extends Objectish>(
  immer: Immer,
  value: T,
  parent?: ImmerState
): Drafted<T, ImmerState> {
  // precondition: createProxy should be guarded by isDraftable, so we know we can safely draft
  const draft: Drafted = isMap(value)
    ? getPlugin("MapSet").proxyMap_(value, parent)
    : isSet(value)
    ? getPlugin("MapSet").proxySet_(value, parent)
    : immer.useProxies_
    ? createProxyProxy(value, parent)
    : getPlugin("ES5").createES5Proxy_(value, parent);

  const scope = parent ? parent.scope_ : getCurrentScope();
  scope.drafts_.push(draft);
  return draft;
}
```

这个代理对象代理的是 produce 函数的第一个参数，通常就是我们状态管理中的 state 对象。在创建完成后会将它放入当前 scope 的 draft\_数组中，也就是将这两者相关联，留着后面遍历。

下面我们就深入`createProxyProxy`函数，看看代理的具体过程，

```ts
export function createProxyProxy<T extends Objectish>(
  base: T,
  parent?: ImmerState
): Drafted<T, ProxyState> {
  const isArray = Array.isArray(base);
  const state: ProxyState = {
    type_: isArray ? ProxyType.ProxyArray : (ProxyType.ProxyObject as any),
    // Track which produce call this is associated with.
    scope_: parent ? parent.scope_ : getCurrentScope()!,
    // True for both shallow and deep changes.
    modified_: false,
    // Used during finalization.
    finalized_: false,
    // Track which properties have been assigned (true) or deleted (false).
    assigned_: {},
    // The parent draft state.
    parent_: parent,
    // The base state.
    base_: base,
    // The base proxy.
    draft_: null as any, // set below
    // The base copy with any updated values.
    copy_: null,
    // Called by the `produce` function.
    revoke_: null as any,
    isManual_: false,
  };

  let target: T = state as any;
  let traps: ProxyHandler<object | Array<any>> = objectTraps;
  if (isArray) {
    target = [state] as any;
    traps = arrayTraps;
  }

  const { revoke, proxy } = Proxy.revocable(target, traps);
  state.draft_ = proxy as any;
  state.revoke_ = revoke;
  return proxy as any;
}
```

这里利用 es6 的 Proxy 语法，Object 类型和 Array 类型的代理方式大同小异。

首先是创建一个 state 对象，这个对象记录了很多信息，对之后的操作很有帮助。比如说`state.base_`，表示当前需要代理的对象、`state.modified_`，表示当前的 state 中的 base\_对象自身，或孙子属性有没有被修改过、而`state.copy_`这是 state.base\_对象的浅拷贝，所有对 base\_的操作，都转移到了它的身上。所以说最后生成的代理对象代理的是 state 对象，而不是原本的 base 对象。这是为了方便记录 base 对象被修改时的信息。和 scope 对象一样，state 中也记录了 parent\_，也是一个反向的链表。

接下来看`objectTraps`中定义的方法，也就是代理对象会拦截的一些基础操作。我们先来看`get`操作，

```ts
export const objectTraps: ProxyHandler<ProxyState> = {
  get(state, prop) {
    if (prop === DRAFT_STATE) return state;

    const source = latest(state);
    if (!has(source, prop)) {
      // non-existing or non-own property...
      return readPropFromProto(state, source, prop);
    }
    const value = source[prop];
    if (state.finalized_ || !isDraftable(value)) {
      return value;
    }
    // Check for existing draft in modified state.
    // Assigned values are never drafted. This catches any drafts we created, too.
    if (value === peek(state.base_, prop)) {
      prepareCopy(state);
      return (state.copy_![prop as any] = createProxy(
        state.scope_.immer_,
        value,
        state
      ));
    }
    return value;
  },
  // ...
};
```

这里的 state 入参就是上面生成的 proxy 对象，而 get 函数会在访问 proxy 对象属性的时候被触发。如果访问的属性是`DRAFT_STATE`，这是一个特殊的属性，那么就会返回 proxy 对象所代理的 state 对象。

```ts
export const DRAFT_STATE: unique symbol = hasSymbol
  ? Symbol.for("immer-state")
  : ("__$immer_state" as any);
```

get 函数的主要逻辑是，如果你访问的是一个不可代理的属性值，或者是这个属性不是对象自身拥有的属性，再或者是这个对象已经被代理过了，那么直接返回。否则，再次调用`createProxy`函数，生成这个对象的代理，并记录下来，最后返回这个新的代理对象。

需要提到的是，在生成子属性的代理之前，会先对当前的 state.base\_对象浅拷贝到 state.copy\_对象。

```ts
export function prepareCopy(state: { base_: any; copy_: any }) {
  if (!state.copy_) {
    state.copy_ = shallowCopy(state.base_);
  }
}
```

而且这个子属性的代理也是会赋值到 state.copy\_[prop]中，之后的修改操作，都会反馈到这个 copy\_对象中。

接着再来看看`set`操作，

```ts
export const objectTraps: ProxyHandler<ProxyState> = {
  // ...
  set(
    state: ProxyObjectState,
    prop: string /* strictly not, but helps TS */,
    value
  ) {
    const desc = getDescriptorFromProto(latest(state), prop);
    if (desc?.set) {
      // special case: if this write is captured by a setter, we have
      // to trigger it with the correct context
      desc.set.call(state.draft_, value);
      return true;
    }
    if (!state.modified_) {
      // the last check is because we need to be able to distinguish setting a non-existing to undefined (which is a change)
      // from setting an existing property with value undefined to undefined (which is not a change)
      const current = peek(latest(state), prop);
      // special case, if we assigning the original value to a draft, we can ignore the assignment
      const currentState: ProxyObjectState = current?.[DRAFT_STATE];
      if (currentState && currentState.base_ === value) {
        state.copy_![prop] = value;
        state.assigned_[prop] = false;
        return true;
      }
      if (is(value, current) && (value !== undefined || has(state.base_, prop)))
        return true;
      prepareCopy(state);
      markChanged(state);
    }

    if (
      state.copy_![prop] === value &&
      // special case: NaN
      typeof value !== "number" &&
      // special case: handle new props with value 'undefined'
      (value !== undefined || prop in state.copy_)
    )
      return true;

    // @ts-ignore
    state.copy_![prop] = value;
    state.assigned_[prop] = true;
    return true;
  },
};
```

稍微有些复杂，我们只看关键的部分。set 操作会在 proxy 的属性被赋值的时候触发，例如 proxy[prop]=xxx。你可能会好奇，如果是这样呢，proxy[aprop][bprop]=xxx。这个就要回到上面我们说的 get 操作，首先 proxy[aprop]会创建一个新的 proxy 对象，相当于 newProxy[bprop]=xxx，执行的是 newProxy 代理对象中的 set 操作。

`state.modified_`是一个 boolean 类型，表示这个 state 所代表的 base 对象有没有被修改过，包括子孙属性。那么第一个修改时，需要调用`prepareCopy`函数和`markChanged`函数。这两个函数看代码中的实现，都是幂等的。

```ts
export function markChanged(state: ImmerState) {
  if (!state.modified_) {
    state.modified_ = true;
    if (state.parent_) {
      markChanged(state.parent_);
    }
  }
}
```

之前说过 state 对象在创建的时候，存了父 state 的引用，所以这里会一直向上递归做标记。

但是，只有在确定属性被更改的情况下，才需要执行这两个函数。如果是新的值和当前的值相同，或者说新的值是 undefined，而当前的属性存在，且值也为 undefined，那么这时候不应该认为发生了`change`。还有一种特殊的情况，如果这个属性值已经被代理过了，但是你又把它所代理的值赋值给它，这时候也不认为是发生了`change`。除此之外，都会更新`state.copy_`，并且更新`state.assigned_`。

接着我们再来看 delete 操作，

```ts
export const objectTraps: ProxyHandler<ProxyState> = {
  // ...
  deleteProperty(state, prop: string) {
    // The `undefined` check is a fast path for pre-existing keys.
    if (peek(state.base_, prop) !== undefined || prop in state.base_) {
      state.assigned_[prop] = false;
      prepareCopy(state);
      markChanged(state);
    } else {
      // if an originally not assigned property was deleted
      delete state.assigned_[prop];
    }
    // @ts-ignore
    if (state.copy_) delete state.copy_[prop];
    return true;
  },
  // ...
};
```

这里首先判断了属性是否在`state.base_`上存在，如果存在，先将操作记录在`state.assigned_`中，然后执行`prepareCopy`，`markChanged`这两个函数。如果不存在，清除`state.assigned_`中的记录。最后更新 state.copy\_。

再接着是 getOwnPropertyDescriptor 操作，获取对象自身的属性描述符，

```ts
export const objectTraps: ProxyHandler<ProxyState> = {
  // ...
  getOwnPropertyDescriptor(state, prop) {
    const owner = latest(state);
    const desc = Reflect.getOwnPropertyDescriptor(owner, prop);
    if (!desc) return desc;
    return {
      writable: true,
      configurable: state.type_ !== ProxyType.ProxyArray || prop !== "length",
      enumerable: desc.enumerable,
      value: owner[prop],
    };
  },
  // ...
};
```

这里会将原始的描述符做一些改动再返回。

其他的操作代理都很简单，就不在赘述了。

我们再回到 produce 函数执行的流程中，在创建完 proxy 以后，在 try finally 块中，调用 produce 函数的第二个参数，proxy 作为参数传进去。我们假设执行很顺利，忽略 patch 功能。这时候，proxy 中已经记录了很多对原始对象的改动，我们需要在`processResult`函数中收集这些改动，返回一个修改过的对象。

`processResult`函数有两个入参，第一个入参`result`，是上面提到的 produce 函数第二个入参函数执行后的返回值。当然一般是没有返回值的。第二个就是我们上面创建的`scope`对象，在它的 drafts\_数组中，记录了本次 produce 函数执行过程中创建的 proxy 对象。接着，便需要递归遍历操作，生成新的对象。

```ts
export function processResult(result: any, scope: ImmerScope) {
  scope.unfinalizedDrafts_ = scope.drafts_.length;
  const baseDraft = scope.drafts_![0];
  const isReplaced = result !== undefined && result !== baseDraft;
  // ...
  if (isReplaced) {
    if (baseDraft[DRAFT_STATE].modified_) {
      revokeScope(scope);
      die(4);
    }
    if (isDraftable(result)) {
      // Finalize the result in case it contains (or is) a subset of the draft.
      result = finalize(scope, result);
      if (!scope.parent_) maybeFreeze(scope, result);
    }
    // ...
  } else {
    // Finalize the base draft.
    result = finalize(scope, baseDraft, []);
  }
  revokeScope(scope);
  // ...
  return result !== NOTHING ? result : undefined;
}
```

如果 result 不为 undefined，也不等于我们的代理对象数组中的第一个，也就是第一个被代理的对象 base，那么`isReplaced`变量为 true。这里有个限制，你不能即返回新值，又修改代理对象。再判断如果这个 result 是可代理的，那么就要调用`finalize`函数，检查它的内部是否有代理对象。接着，如果当前的 scope 不再有父 scope，也就是说本次 produce 调用不在其他的 produce 调用里面，那就要尝试冻结这个 result 对象。这是为了让我们的 state 不可在 produce 函数外面被修改，真正具有 immutable 的特性。

在大多数业务情况下，result 会是 undefined，所以我们会走第二个流程，将首个代理对象放入`finalize`函数中处理。

在 finalize 函数中，会酌情地递归遍历 proxy 对象中的属性，使用 copy\_替代 base\_。

```ts
function finalize(rootScope: ImmerScope, value: any, path?: PatchPath) {
  // Don't recurse in tho recursive data structures
  if (isFrozen(value)) return value;

  const state: ImmerState = value[DRAFT_STATE];
  // A plain object, might need freezing, might contain drafts
  if (!state) {
    each(
      value,
      (key, childValue) =>
        finalizeProperty(rootScope, state, value, key, childValue, path),
      true // See #590, don't recurse into non-enumarable of non drafted objects
    );
    return value;
  }
  // Never finalize drafts owned by another scope.
  if (state.scope_ !== rootScope) return value;
  // Unmodified draft, return the (frozen) original
  if (!state.modified_) {
    maybeFreeze(rootScope, state.base_, true);
    return state.base_;
  }
  // Not finalized yet, let's do that now
  if (!state.finalized_) {
    state.finalized_ = true;
    state.scope_.unfinalizedDrafts_--;
    const result =
      // For ES5, create a good copy from the draft first, with added keys and without deleted keys.
      state.type_ === ProxyType.ES5Object || state.type_ === ProxyType.ES5Array
        ? (state.copy_ = shallowCopy(state.draft_))
        : state.copy_;
    // Finalize all children of the copy
    // For sets we clone before iterating, otherwise we can get in endless loop due to modifying during iteration, see #628
    // Although the original test case doesn't seem valid anyway, so if this in the way we can turn the next line
    // back to each(result, ....)
    each(
      state.type_ === ProxyType.Set ? new Set(result) : result,
      (key, childValue) =>
        finalizeProperty(rootScope, state, result, key, childValue, path)
    );
    // everything inside is frozen, we can freeze here
    maybeFreeze(rootScope, result, false);
    // ...
  }
  return state.copy_;
}
```

我们可以看到 finalize 函数中总是会有返回值的。如果参数 value 已经是被冻结过的，或者说是原始类型，就说明已经处理过了或者不需要处理，直接返回。

```ts
export function isFrozen(obj: any): boolean {
  if (obj == null || typeof obj !== "object") return true;
  // See #600, IE dies on non-objects in Object.isFrozen
  return Object.isFrozen(obj);
}
```

如果 value 不是代理对象，这是很有可能的，因为我们是递归遍历代理对象子属性，而代理对象的属性值，并不一定也是代理对象。这个时候再对它的子属性进行遍历，调用`finalizeProperty`函数，然后返回 value。这个函数我们后面会讲到。

接下来，如果这个 value 是父 scope 的，那么也不会处理，直接返回。

如果这个 value 没有被修改过，modified\_值为 false，那么尝试冻结 state.base\_,然后再返回 state.base\_。

最后的部分就是对 state.copy\_进行处理了，依旧是调用`finalizeProperty`函数处理。处理完成后返回 state.copy\_。

在 finalize 函数中只有遍历，没有值的修改和替换，这部分工作都在 finalizeProperty 函数中完成，

```ts
function finalizeProperty(
  rootScope: ImmerScope,
  parentState: undefined | ImmerState,
  targetObject: any,
  prop: string | number,
  childValue: any,
  rootPath?: PatchPath
) {
  if (__DEV__ && childValue === targetObject) die(5);
  if (isDraft(childValue)) {
    const path =
      rootPath &&
      parentState &&
      parentState!.type_ !== ProxyType.Set && // Set objects are atomic since they have no keys.
      !has((parentState as Exclude<ImmerState, SetState>).assigned_!, prop) // Skip deep patches for assigned keys.
        ? rootPath!.concat(prop)
        : undefined;
    // Drafts owned by `scope` are finalized here.
    const res = finalize(rootScope, childValue, path);
    set(targetObject, prop, res);
    // Drafts from another scope must prevented to be frozen
    // if we got a draft back from finalize, we're in a nested produce and shouldn't freeze
    if (isDraft(res)) {
      rootScope.canAutoFreeze_ = false;
    } else return;
  }
  // Search new objects for unfinalized drafts. Frozen objects should never contain drafts.
  if (isDraftable(childValue) && !isFrozen(childValue)) {
    if (!rootScope.immer_.autoFreeze_ && rootScope.unfinalizedDrafts_ < 1) {
      // optimization: if an object is not a draft, and we don't have to
      // deepfreeze everything, and we are sure that no drafts are left in the remaining object
      // cause we saw and finalized all drafts already; we can stop visiting the rest of the tree.
      // This benefits especially adding large data tree's without further processing.
      // See add-data.js perf test
      return;
    }
    finalize(rootScope, childValue);
    // immer deep freezes plain objects, so if there is no parent state, we freeze as well
    if (!parentState || !parentState.scope_.parent_)
      maybeFreeze(rootScope, childValue);
  }
}
```

如果传入的 childValue 是代理对象，我们还需要通过 finalize 去遍历子属性。而 finalize 函数执行后的返回值就是对应这个 childValue 的新对象。我们将它赋值给 targetObject\[prop\]。如果不是，但是这个 childValue 是可以被代理的，那么它的内部可能会存在代理对象，所有还需要调用 finalize 对象遍历。

最后我们得到了一个基于原对象修改过后的新对象，在返回它之前，还要调用`revokeScope`函数清理 draft\_数组，

```ts
export function revokeScope(scope: ImmerScope) {
  leaveScope(scope);
  scope.drafts_.forEach(revokeDraft);
  // @ts-ignore
  scope.drafts_ = null;
}

export function leaveScope(scope: ImmerScope) {
  if (scope === currentScope) {
    currentScope = scope.parent_;
  }
}

function revokeDraft(draft: Drafted) {
  const state: ImmerState = draft[DRAFT_STATE];
  if (
    state.type_ === ProxyType.ProxyObject ||
    state.type_ === ProxyType.ProxyArray
  )
    state.revoke_();
  else state.revoked_ = true;
}
```

`leaveScope`会将当前的 scope 退出，这样当前的 scope 就变成了原 scope 的父级。`revokeDraft`函数用来清理代理对象，要么调用 revoke*，要么将 revoked*标志为 true。这取决于当前的代理对象是哪种类型。

到此，这就是 immer 的一个基础工作流程。

除此之外，还有相当一部分代码是兼容不支持 es6 Proxy 特性的 es5 环境，以及支持 Map，Set 的代理和 patch 等功能。

接下来我们来看看 patch 功能是如何实现的。

## Patch

```ts
import { produceWithPatches } from "immer";

const [nextState, patches, inversePatches] = produceWithPatches(
  {
    age: 33,
  },
  (draft) => {
    draft.age++;
  }
);
```

结果如下：

```ts
[
  {
    age: 34,
  },
  [
    {
      op: "replace",
      path: ["age"],
      value: 34,
    },
  ],
  [
    {
      op: "replace",
      path: ["age"],
      value: 33,
    },
  ],
];
```

不得不说，这是个很有趣的功能。

一个 patch 的结构如下，还是很好理解的。

```ts
export interface Patch {
  op: "replace" | "remove" | "add";
  path: (string | number)[];
  value?: any;
}
```

`produceWithPatches`函数内部调用了`produce`函数，因为 produce 函数还有第三个参数，专门用来生成 patches。

```ts
produce: IProduce = (base: any, recipe?: any, patchListener?: any) => {
  // ...
  if (patchListener !== undefined && typeof patchListener !== "function")
    die(7);
  // ...
  if (isDraftable(base)) {
    const scope = enterScope(this);
    // ...
    usePatchesInScope(scope, patchListener);
    return processResult(result, scope);
  }
};
```

scope 对象的结构我们上面提到过，调用`usePatchesInScope`会在 scope 中设置初始 patches，inversePatches\_数组，还会存入 patchListener 函数。

```ts
export function usePatchesInScope(
  scope: ImmerScope,
  patchListener?: PatchListener
) {
  if (patchListener) {
    getPlugin("Patches"); // assert we have the plugin
    scope.patches_ = [];
    scope.inversePatches_ = [];
    scope.patchListener_ = patchListener;
  }
}
```

patch 会在 produce 函数运行的过程中，每次修改 draft 对象时记录，最后处理代理对象时生成。所以，在`state={assigned_:{[prop]: boolean}}`中记录了赋值和删除操作，用 true 和 false 表示。在调用`processResult`函数中通过这些记录，生成对应的 patches。

```ts
export function processResult(result: any, scope: ImmerScope) {
  // ...
  const baseDraft = scope.drafts_![0];
  const isReplaced = result !== undefined && result !== baseDraft;
  if (isReplaced) {
    // ...
    if (scope.patches_) {
      getPlugin("Patches").generateReplacementPatches_(
        baseDraft[DRAFT_STATE],
        result,
        scope.patches_,
        scope.inversePatches_!
      );
    }
  } else {
    // Finalize the base draft.
    result = finalize(scope, baseDraft, []);
  }
  // ...
  if (scope.patches_) {
    scope.patchListener_!(scope.patches_, scope.inversePatches_!);
  }
  return result !== NOTHING ? result : undefined;
}
```

如果`isReplaced`为 true，最后会调用`generateReplacementPatches_`方法生成 patches。

```ts
function generateReplacementPatches_(
  rootState: ImmerState,
  replacement: any,
  patches: Patch[],
  inversePatches: Patch[]
): void {
  patches.push({
    op: REPLACE,
    path: [],
    value: replacement === NOTHING ? undefined : replacement,
  });
  inversePatches.push({
    op: REPLACE,
    path: [],
    value: rootState.base_,
  });
}
```

这个函数很简单，因为这里的含义是使用新值替换以前的值，所以 patches 和 inversePatches 只包含一个 patch。

在 processResult 函数中最后返回 result 之前，会调用`patchListener_`回调，传入 patches\_和 inversePatches\_。

在 finalize 函数中，也有生成 patch 的地方，

```ts
function finalize(rootScope: ImmerScope, value: any, path?: PatchPath) {
  // ...
  const state: ImmerState = value[DRAFT_STATE];
  // ...
  // Not finalized yet, let's do that now
  if (!state.finalized_) {
    // ...
    each(
      state.type_ === ProxyType.Set ? new Set(result) : result,
      (key, childValue) =>
        finalizeProperty(rootScope, state, result, key, childValue, path)
    );
    // ...
    if (path && rootScope.patches_) {
      getPlugin("Patches").generatePatches_(
        state,
        path,
        rootScope.patches_,
        rootScope.inversePatches_!
      );
    }
  }
  return state.copy_;
}
```

只有处理代理对象，才会生成 patch，并且每个代理对象只会拿来生成一次。注意 path 变量，它是会在每次调用 finalizeProperty 函数的过程中做拼接的。

```ts
function finalizeProperty(
  rootScope: ImmerScope,
  parentState: undefined | ImmerState,
  targetObject: any,
  prop: string | number,
  childValue: any,
  rootPath?: PatchPath
) {
  // ...
  if (isDraft(childValue)) {
    const path =
      rootPath &&
      parentState &&
      parentState!.type_ !== ProxyType.Set && // Set objects are atomic since they have no keys.
      !has((parentState as Exclude<ImmerState, SetState>).assigned_!, prop) // Skip deep patches for assigned keys.
        ? rootPath!.concat(prop)
        : undefined;
    // Drafts owned by `scope` are finalized here.
    const res = finalize(rootScope, childValue, path);
    // ...
  }
  // ...
}
```

这里的拼接条件有点复杂，前三个要求还好理解，但是第四个要求有点奇怪，

```ts
!has((parentState as Exclude<ImmerState, SetState>).assigned_!, prop); // Skip deep patches for assigned keys.
```

要求在 parentState.assigned\_中不能有这个 prop 的记录。这就表示这个 prop 不能有过 set 操作，delete 操作。也就是说如果这个 prop 代表的值被替换过或删除过, 那就不需要再生成后面的 patch。举个例子，

```ts
const [_, patches] = produceWithPatches(
  { a: { b: 1, c: { b: 2 } } },
  (draft) => {
    draft.a.c = 4;
    draft.a.c = { b: 2 };
    draft.a.c.b = 3;
  }
);

console.log(patches);
// 输出
// [ { op: 'replace', path: [ 'a', 'c' ], value: { b: 3 } } ]
```

这个时候`draft.a`触发了 get 操作，已经是代理对象了，它的 assigned\_分别记录了'c'。如果再修改这个属性值自身，或其孙子属性值也就不再多做记录了，而是只采用最后一次改变的值。这时候上面提到的第四个要求不满足，所以 path 变成了 undefined，再调用`finalize`时，因为 path 为空，就不会调用`generatePatches_`函数了。

在这之后，我们来看看`generatePatches_`函数，

```ts
function generatePatches_(
  state: ImmerState,
  basePath: PatchPath,
  patches: Patch[],
  inversePatches: Patch[]
): void {
  switch (state.type_) {
    case ProxyType.ProxyObject:
    case ProxyType.ES5Object:
    case ProxyType.Map:
      return generatePatchesFromAssigned(
        state,
        basePath,
        patches,
        inversePatches
      );
    case ProxyType.ES5Array:
    case ProxyType.ProxyArray:
      return generateArrayPatches(state, basePath, patches, inversePatches);
    case ProxyType.Set:
      return generateSetPatches(
        state as any as SetState,
        basePath,
        patches,
        inversePatches
      );
  }
}
```

它会根据代理的类型来生成 patches。我们着重看两种类型，`ProxyType.ProxyObject`，`ProxyType.ProxyArray`。

如果是`ProxyType.ProxyObject`类型，

```ts
// This is used for both Map objects and normal objects.
function generatePatchesFromAssigned(
  state: MapState | ES5ObjectState | ProxyObjectState,
  basePath: PatchPath,
  patches: Patch[],
  inversePatches: Patch[]
) {
  const { base_, copy_ } = state;
  each(state.assigned_!, (key, assignedValue) => {
    const origValue = get(base_, key);
    const value = get(copy_!, key);
    const op = !assignedValue ? REMOVE : has(base_, key) ? REPLACE : ADD;
    if (origValue === value && op === REPLACE) return;
    const path = basePath.concat(key as any);
    patches.push(op === REMOVE ? { op, path } : { op, path, value });
    inversePatches.push(
      op === ADD
        ? { op: REMOVE, path }
        : op === REMOVE
        ? { op: ADD, path, value: clonePatchValueIfNeeded(origValue) }
        : { op: REPLACE, path, value: clonePatchValueIfNeeded(origValue) }
    );
  });
}
```

通过 assigned\_中的值来判断哪些属性发生了改变，通过 base\_，和 copy\_中相同 key 值来判断是替换还是新增。basePath 拼接 key 来生成 path。patches 很容易得到，而在 inversePatches 中，还需要通过特殊的方式获取 value。

```ts
function clonePatchValueIfNeeded<T>(obj: T): T {
  if (isDraft(obj)) {
    return deepClonePatchValue(obj);
  } else return obj;
}
function deepClonePatchValue(obj: any) {
  if (!isDraftable(obj)) return obj;
  if (Array.isArray(obj)) return obj.map(deepClonePatchValue);
  if (isMap(obj))
    return new Map(
      Array.from(obj.entries()).map(([k, v]) => [k, deepClonePatchValue(v)])
    );
  if (isSet(obj)) return new Set(Array.from(obj).map(deepClonePatchValue));
  const cloned = Object.create(Object.getPrototypeOf(obj));
  for (const key in obj) cloned[key] = deepClonePatchValue(obj[key]);
  if (has(obj, immerable)) cloned[immerable] = obj[immerable];
  return cloned;
}
```

如果 origValue 是个代理对象，那么说明当前存在 produce 函数嵌套。还得对它进行深拷贝，因为它可能会在后续的遍历过程中被改变。

接着我们再看下 `ProxyType.ProxyArray` 类型的处理过程，

```ts
function generateArrayPatches(
  state: ES5ArrayState | ProxyArrayState,
  basePath: PatchPath,
  patches: Patch[],
  inversePatches: Patch[]
) {
  let { base_, assigned_ } = state;
  let copy_ = state.copy_!;

  // Reduce complexity by ensuring `base` is never longer.
  if (copy_.length < base_.length) {
    // @ts-ignore
    [base_, copy_] = [copy_, base_];
    [patches, inversePatches] = [inversePatches, patches];
  }

  // Process replaced indices.
  for (let i = 0; i < base_.length; i++) {
    if (assigned_[i] && copy_[i] !== base_[i]) {
      const path = basePath.concat([i]);
      patches.push({
        op: REPLACE,
        path,
        // Need to maybe clone it, as it can in fact be the original value
        // due to the base/copy inversion at the start of this function
        value: clonePatchValueIfNeeded(copy_[i]),
      });
      inversePatches.push({
        op: REPLACE,
        path,
        value: clonePatchValueIfNeeded(base_[i]),
      });
    }
  }

  // Process added indices.
  for (let i = base_.length; i < copy_.length; i++) {
    const path = basePath.concat([i]);
    patches.push({
      op: ADD,
      path,
      // Need to maybe clone it, as it can in fact be the original value
      // due to the base/copy inversion at the start of this function
      value: clonePatchValueIfNeeded(copy_[i]),
    });
  }
  if (base_.length < copy_.length) {
    inversePatches.push({
      op: REPLACE,
      path: basePath.concat(["length"]),
      value: base_.length,
    });
  }
}
```

首先是为了简化对比，将确保 base\_始终代表长度小的数组，并且 patches 和 inversePatches 也做相应替换。

接着将根据 base\_的长度和 assigned\_对象中的记录，找出 replace 操作。

最后，长的那个数组剩下的值作为 add 操作，注意这时候的 inversePatches，它是直接操作数组的 length 属性，将多个 add 归纳为一步。不过这里似乎没有考虑 remove 的情况，应该是 immer 的 bug，所以提了一个[issue](https://github.com/immerjs/immer/issues/879)

之前提到在`finalize`函数的递归调用中，会生成 patches。最终在`processResult`函数中，会将这些生成的 patches 交给`scope.patchListener_`回调。

在我们得到了这些 patches 以后，可以使用`applyPatches`函数将这些 patches 应用到 state 上，例如，

```ts
const state = [1, 2, 3];
const [_, patches] = produceWithPatches([1, 2, 3], (draft) => {
  draft.push(4);
  draft.push(5);
});

const state2 = applyPatches(state, patches);

console.log(state, state2);
// [ 1, 2, 3 ] [ 1, 2, 3, 4, 5 ]
```

接下来我们来看下`applyPatches`函数的实现，

```ts
export class Immer implements ProducersFns {
  // ...
  applyPatches<T extends Objectish>(base: T, patches: Patch[]): T {
    // If a patch replaces the entire state, take that replacement as base
    // before applying patches
    let i: number;
    for (i = patches.length - 1; i >= 0; i--) {
      const patch = patches[i];
      if (patch.path.length === 0 && patch.op === "replace") {
        base = patch.value;
        break;
      }
    }

    const applyPatchesImpl = getPlugin("Patches").applyPatches_;
    if (isDraft(base)) {
      // N.B: never hits if some patch a replacement, patches are never drafts
      return applyPatchesImpl(base, patches) as any;
    }
    // Otherwise, produce a copy of the base state.
    return this.produce(base, (draft: Drafted) =>
      applyPatchesImpl(draft, patches.slice(i + 1))
    ) as any;
  }
  // ...
}
```

首先是处理一种特殊情况，整个 base 被替换为另一个值。这种情况一般是在 produce 函数中，返回了一个新的对象。这时候 patches 中只会存在一条记录。这里选择倒序遍历 patches 是为了确定这个特殊的 replace patch 的位置，记为 i。下方有两个处理分支，如果 base 是代理对象，那么直接将所有的 patches 作用于这个对象上，这段逻辑不是很明白。在其他情况下，利用 produce 函数来处理。注意这里的 patches 被截取过了。

最后，我们再来看下`applyPatchesImpl`也就是`applyPatches_`函数的实现，

```ts
function applyPatches_<T>(draft: T, patches: Patch[]): T {
  patches.forEach((patch) => {
    const { path, op } = patch;

    let base: any = draft;
    for (let i = 0; i < path.length - 1; i++) {
      const parentType = getArchtype(base);
      const p = "" + path[i];
      // See #738, avoid prototype pollution
      if (
        (parentType === Archtype.Object || parentType === Archtype.Array) &&
        (p === "__proto__" || p === "constructor")
      )
        die(24);
      if (typeof base === "function" && p === "prototype") die(24);
      base = get(base, p);
      if (typeof base !== "object") die(15, path.join("/"));
    }

    const type = getArchtype(base);
    const value = deepClonePatchValue(patch.value); // used to clone patch to ensure original patch is not modified, see #411
    const key = path[path.length - 1];
    switch (op) {
      case REPLACE:
        switch (type) {
          case Archtype.Map:
            return base.set(key, value);
          /* istanbul ignore next */
          case Archtype.Set:
            die(16);
          default:
            // if value is an object, then it's assigned by reference
            // in the following add or remove ops, the value field inside the patch will also be modifyed
            // so we use value from the cloned patch
            // @ts-ignore
            return (base[key] = value);
        }
      case ADD:
        switch (type) {
          case Archtype.Array:
            return base.splice(key as any, 0, value);
          case Archtype.Map:
            return base.set(key, value);
          case Archtype.Set:
            return base.add(value);
          default:
            return (base[key] = value);
        }
      case REMOVE:
        switch (type) {
          case Archtype.Array:
            return base.splice(key as any, 1);
          case Archtype.Map:
            return base.delete(key);
          case Archtype.Set:
            return base.delete(patch.value);
          default:
            return delete base[key];
        }
      default:
        die(17, op);
    }
  });

  return draft;
}
```

在这个函数中，会遍历执行所有的 patch。首先根据 path 获取到最终要更改的对象，所以这里的 for 循环终止条件是`i < path.length-1`。然后还要判断这个路径是否合法。接着将 patch 中的 value 值做个深拷贝，因为我们要确保 patch 中的 value 以后不会被改动。最后根据 patch 的类型和对象的类型进行修改操作。

以上就是 immer 主要功能的源码解析。
