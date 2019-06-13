# DefaultIterableDiffer

[source](https://github.com/blinkcat/angular/blob/master/packages/core/src/change_detection/differs/default_iterable_differ.ts)

另外，[这篇文章](https://segmentfault.com/a/1190000013516380)也有精彩的讲解。

用来分辨可遍历对象(数组或实现[Iterator](http://es6.ruanyifeng.com/#docs/iterator)接口的对象)中，有哪些数据项发生变化。即，remove、add、move。

## usage

```typescript
// trackByFn 实现 TrackByFunction 接口，用来返回数据项的标识。类似于 react 中的 key。
// interface TrackByFunction<T> { (index: number, item: T): any; }
const dataDiffer = new DefaultIterableDiffer<V>(trackByFn);
// 返回 DefaultIterableDiffer 自身
dataDiffer.diff([1, 2, 3, 4]);
// 输出每种变化 Order: remove, add, move
dataDiffer
  .diff([2, 4, 3, 5])
  .forEachOperation((item: IterableChangeRecord, previousIndex: number | null, currentIndex: number | null) => {
    console.log(item, previousIndex, currentIndex);
  });
```

## 解析

DefaultIterableDiffer 实现了 IterableDiffer, IterableChanges 两个接口。

```typescript
interface IterableDiffer<V> {
  /**
   * Compute a difference between the previous state and the new `object` state.
   *
   * @param object containing the new value.
   * @returns an object describing the difference. The return value is only valid until the next
   * `diff()` invocation.
   */
  diff(object: NgIterable<V>): IterableChanges<V> | null;
}

export interface IterableChanges<V> {
  /**
   * Iterate over all changes. `IterableChangeRecord` will contain information about changes
   * to each item.
   */
  forEachItem(fn: (record: IterableChangeRecord<V>) => void): void;

  /**
   * Iterate over a set of operations which when applied to the original `Iterable` will produce the
   * new `Iterable`.
   *
   * NOTE: These are not necessarily the actual operations which were applied to the original
   * `Iterable`, rather these are a set of computed operations which may not be the same as the
   * ones applied.
   *
   * @param record A change which needs to be applied
   * @param previousIndex The `IterableChangeRecord#previousIndex` of the `record` refers to the
   *        original `Iterable` location, where as `previousIndex` refers to the transient location
   *        of the item, after applying the operations up to this point.
   * @param currentIndex The `IterableChangeRecord#currentIndex` of the `record` refers to the
   *        original `Iterable` location, where as `currentIndex` refers to the transient location
   *        of the item, after applying the operations up to this point.
   */
  forEachOperation(
    fn: (record: IterableChangeRecord<V>, previousIndex: number | null, currentIndex: number | null) => void
  ): void;

  /**
   * Iterate over changes in the order of original `Iterable` showing where the original items
   * have moved.
   */
  forEachPreviousItem(fn: (record: IterableChangeRecord<V>) => void): void;

  /** Iterate over all added items. */
  forEachAddedItem(fn: (record: IterableChangeRecord<V>) => void): void;

  /** Iterate over all moved items. */
  forEachMovedItem(fn: (record: IterableChangeRecord<V>) => void): void;

  /** Iterate over all removed items. */
  forEachRemovedItem(fn: (record: IterableChangeRecord<V>) => void): void;

  /** Iterate over all items which had their identity (as computed by the `TrackByFunction`)
   * changed. */
  forEachIdentityChange(fn: (record: IterableChangeRecord<V>) => void): void;
}
```

DefaultIterableDiffer 在内部维护了两个 Map，和几个 linkedList。之所以使用 LinkedList 是因为内部算法中有大量的在当前列表中进行插入和删除的操作。

内部的 Map 使用 trackId 作为 key，也就是上面构造函数中传入的 trackByFn 执行后的返回值。这个函数默认是

```typescript
const trackByIdentity = (index: number, item: any) => item;
```

而 Map 的 value 是一个 LinkedList。这个 list 中存着具有 相同 trackId 的 IterableChangeRecord\_，

```typescript
export class IterableChangeRecord_<V> implements IterableChangeRecord<V> {
  currentIndex: number | null = null;
  previousIndex: number | null = null;

  /** @internal */
  _nextPrevious: IterableChangeRecord_<V> | null = null;
  /** @internal */
  _prev: IterableChangeRecord_<V> | null = null;
  /** @internal */
  _next: IterableChangeRecord_<V> | null = null;
  /** @internal */
  _prevDup: IterableChangeRecord_<V> | null = null;
  /** @internal */
  _nextDup: IterableChangeRecord_<V> | null = null;
  /** @internal */
  _prevRemoved: IterableChangeRecord_<V> | null = null;
  /** @internal */
  _nextRemoved: IterableChangeRecord_<V> | null = null;
  /** @internal */
  _nextAdded: IterableChangeRecord_<V> | null = null;
  /** @internal */
  _nextMoved: IterableChangeRecord_<V> | null = null;
  /** @internal */
  _nextIdentityChange: IterableChangeRecord_<V> | null = null;

  constructor(public item: V, public trackById: any) {}
}
```

没错，IterableChangeRecord\_ 是内部对数据项 item 进行封装过的对象，简称为 record。内部使用的两个 Map 是 \_linkedRecords，\_unlinkedRecords。前者记录着当前在使用的 record，后者记录已经删除的 record。  
除此之外，内部还维护了 5 个 LinkedList。其中

1.  \_itHead，\_itTail 记录当前数组数据项 list 的头尾指针。
2.  \_additionsHead，\_additionsTail 记录了当前数组相对于前一个数组增加数据项的头尾指针。
3.  \_movesHead，\_movesTail 记录了当前数组相对于上一个数组中发生移动的数据项。
4.  \_removalsHead，\_removalsTail 同理，记录的是删除的数据项。
5.  \_identityChangesHead，\_identityChangesTail 记录的是 item 引用发生改变的数据项。

接着，在调用 diff 的过程中，DefaultIterableDiffer 会逐项比较传入的数组和当前维护的，也就是上面说的第一个 linkedList，存在着哪些变化，删除了，添加了，还是移动了某些数据项，又或者是某些数据项的引用发生了变化。然后，根据这些变化构造新的 Map 和 LinkedList。  
上面提到的 IterableChanges 接口中的许多方法，就是根据这些 list 来进行迭代，访问。例如

```typescript
  forEachAddedItem(fn: (record: IterableChangeRecord_<V>) => void) {
    let record: IterableChangeRecord_<V>|null;
    for (record = this._additionsHead; record !== null; record = record._nextAdded) {
      fn(record);
    }
  }

  forEachMovedItem(fn: (record: IterableChangeRecord_<V>) => void) {
    let record: IterableChangeRecord_<V>|null;
    for (record = this._movesHead; record !== null; record = record._nextMoved) {
      fn(record);
    }
  }
```

其中最常用的一个是 forEachOperation，也是源码中最难理解的一个。大意是遍历每一个操作，给出当前操作的对象 item: IterableChangeRecord<V>。previousIndex: number|null 是经过上次操作后，这个 item 在当前数组中所处的位置，如果是 null，表示是新加的 item。currentIndex: number|null 是当前 item 在本次操作后，将会在当前数组中的位置，如果是 null，表示这个 item 是需要删除的。

```typescript
function forEachOperation(
  fn: (item: IterableChangeRecord<V>, previousIndex: number | null, currentIndex: number | null) => void
) {
  let nextIt = this._itHead;
  let nextRemove = this._removalsHead;
  // 如果是删除了一个item，这个值减一，表示之后的item位置会相对往前挪一。
  // 如果是增加一个item，则加一，表示之后的item的位置往后挪一。
  let addRemoveOffset = 0;
  // 表示移动一个item，对其他item位置的影响
  let moveOffsets: number[] | null = null;
  while (nextIt || nextRemove) {
    // Figure out which is the next record to process
    // Order: remove, add, move
    // 这块是为了尽量保证遍历的顺序。
    const record: IterableChangeRecord<V> =
      !nextRemove || (nextIt && nextIt.currentIndex! < getPreviousIndex(nextRemove, addRemoveOffset, moveOffsets))
        ? nextIt!
        : nextRemove;
    const adjPreviousIndex = getPreviousIndex(record, addRemoveOffset, moveOffsets);
    const currentIndex = record.currentIndex;

    // consume the item, and adjust the addRemoveOffset and update moveDistance if necessary
    if (record === nextRemove) {
      addRemoveOffset--;
      nextRemove = nextRemove._nextRemoved;
    } else {
      nextIt = nextIt!._next;
      if (record.previousIndex == null) {
        addRemoveOffset++;
      } else {
        /**
         * 这块是关于计算移动的，这里实现的算法比较难理解，类似于动态规划。举个例子：
         * [1,2,3,4,5] -> [2,5,4,3,1]
         * angular对这块的理解是，2先从原来的位置1向前移动到位置0得到 [2,1,3,4,5]。
         * 然后5从位置4向前移动到位置1，得到 [2,5,1,3,4]，
         * 接着4从位置4向前移动到位置2，得到 [2,5,4,1,3]，
         * 最后3从位置4向前移动到位置3，得到最终结果 [2,5,4,3,1]。
         * 所以说下面源码注释中才会有 currentIndex < previousIndex 恒成立。
         */
        // INVARIANT:  currentIndex < previousIndex
        if (!moveOffsets) moveOffsets = [];
        // 忽略增加和删除的影响，因为它们对计算移动不会带来影响，比如说 1,2,3
        // 后边的3移动到2的前边去，不管1有没有被删除，moveOffsets中2这个值所在位置的 offset 都需要加1。增加同理。
        const localMovePreviousIndex = adjPreviousIndex - addRemoveOffset;
        const localCurrentIndex = currentIndex! - addRemoveOffset;
        // 如果是相等，说明已经移动到位了。参考上面例子中的 1。
        if (localMovePreviousIndex != localCurrentIndex) {
          // 从移动起点开始计算，先算出起点之前的位置在本次移动中实际所处的位置，如果这个位置在起点和目标点之间，
          // 则需要加1
          for (let i = 0; i < localMovePreviousIndex; i++) {
            const offset = i < moveOffsets.length ? moveOffsets[i] : (moveOffsets[i] = 0);
            const index = offset + i;
            if (localCurrentIndex <= index && index < localMovePreviousIndex) {
              moveOffsets[i] = offset + 1;
            }
          }
          // 这里用到了previousIndex，因为之前忽略了 addRemoveOffset
          const previousIndex = record.previousIndex;
          moveOffsets[previousIndex] = localCurrentIndex - localMovePreviousIndex;
        }
      }
    }

    if (adjPreviousIndex !== currentIndex) {
      fn(record, adjPreviousIndex, currentIndex);
    }
  }
}
// 当前变化中的 previousIndex，应该是原始的 previousIndex 加上之前添加删除item的影响和移动item的影响
function getPreviousIndex(item: any, addRemoveOffset: number, moveOffsets: number[] | null): number {
  const previousIndex = item.previousIndex;
  if (previousIndex === null) return previousIndex;
  let moveOffset = 0;
  if (moveOffsets && previousIndex < moveOffsets.length) {
    moveOffset = moveOffsets[previousIndex];
  }
  return previousIndex + addRemoveOffset + moveOffset;
}
```
