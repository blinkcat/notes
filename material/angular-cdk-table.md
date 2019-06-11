# CdkTable

[介绍以及用法](https://material.angular.io/cdk/table/overview)  
version: 8.0.0

## IterableDiffers

用来判断数据具体哪些地方发生改变

```typescript
// ngOnInit 中创建一个
this._dataDiffer = this._differs.find([]).create((_i: number, dataRow: RenderRow<T>) => {
  return this.trackBy ? this.trackBy(dataRow.dataIndex, dataRow.data) : dataRow;
});
// renderRows 中用来查找数据中的变化
const changes = this._dataDiffer.diff(this._renderRows);

changes.forEachOperation(
  (record: IterableChangeRecord<RenderRow<T>>, prevIndex: number | null, currentIndex: number | null) => {
    if (record.previousIndex == null) {
      this._insertRow(record.item, currentIndex!);
    } else if (currentIndex == null) {
      viewContainer.remove(prevIndex!);
    } else {
      const view = <RowViewRef<T>>viewContainer.get(prevIndex!);
      viewContainer.move(view!, currentIndex);
    }
  }
);

changes.forEachIdentityChange((record: IterableChangeRecord<RenderRow<T>>) => {
  const rowView = <RowViewRef<T>>viewContainer.get(record.currentIndex!);
  rowView.context.$implicit = record.item.data;
});
```

[关于 IterableDiffer 的一篇文章](https://netbasal.com/getting-to-know-angular-differs-60cd68f4bd8f)
