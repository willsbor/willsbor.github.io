---
layout: post
title: Restructure 思考脈絡 (2)
--- 

上一篇： [Restructure 思考脈絡 1](2018-08-01-Restructure.md)


延續上一篇的題目，在漸進式地增加功能：

------

> 將多個來源的資料，合併成一份資料後匯入系統
>
> 資料內有多筆細目，細目數量可能很大，且 process 的 memory 大小有限制
>
> 資料使用其中一欄做排序，由小到大，且唯一，
>  - 如果遇到值一樣的欄位，則取用在 array 中比較前面的 source 
>
> **資料來源支援另一種輸入形式，輸入資料為與上次最後輸入的資料做差異，標記要新增或刪除的資料**

不考慮在工作流程上的各種意外處理，先以單一化目前物件的工作能力來看，只要處理有關**讀取/匯入資料**的工作事項，因此：

1. 每個資料來源(source)都應該有有提供各自的增減資料
2. 匯入時再合併增加/刪減的資料
3. 增加是__增加沒有的資料__，刪除是__刪除上次存在的資料__，如果對**同一個號碼**在不同 source 中有增加或刪除，那應該要保留增加的指令

先來思考 source 提供的面向，要增加一個描述 `RowDataOperation` 來告訴 `DataLoader` 資料是增還是刪。開啟 source 時，要告訴他這次要拿的資料形式是 `全拿(full)` 還是 `差異資料(diff)`

{% highlight swift linenos %}
enum RowDataOperation {
    case add
    case remove
}

protocol RowDataValue: Comparable {
    var operation: RowDataOperation { get }
}

enum DataProvidingMode {
    case full
    case diff
}

protocol SortedDataSource {
    associatedtype RowData: RowDataValue

    func open(mode: DataProvidingMode)
    func next() -> Bool
    func getRowData() -> RowData
    func close()
}
{% endhighlight %}


接著看 `DataLoader` 如何應用 source

- method `readMergedData(...)` 提供了資供資料的輸入參數。

- 原本的 `getMinRowData()` 也因為 .full 和 .diff 模式的不同，所以分開成兩個動作：
  1. `getMinRowDataSet()`：綜合各個 Source，且依據 source 的排序，拿取目前*所有*數值最小的 RowData
  2. 取用這群 (Set) 資料的策略，則交由 `DataProvidingMode` 來決定。這裡用 `extesion Array where ... `來實作

{% highlight swift linenos %}
class DataLoader<T: SortedDataSource> {
    
    var sources: [T] = []
    
    func readMergedData(mode: DataProvidingMode, listMaxSize: Int, _ handler: ([T.RowData]) -> Void) {
        
        sources.forEach { (source) in
            source.open(mode: mode)
        }
        
        var temp: [T.RowData] = []
        while hasNext() {
            let value =
                getMinRowDataSet()
                .select(by: mode)

            temp.append(value!)
            
            if temp.count == listMaxSize {
                autoreleasepool {
                    handler(temp)
                    temp = []
                }
            }
        }
        
        if temp.count > 0 {
            autoreleasepool {
                handler(temp)
                temp = []
            }
        }
        
        sources.forEach { (source) in
            source.close()
        }
    }
    
    /// 綜合各個 Source，依據 source 的排序，拿取目前數值最小的 RowData
    ///
    /// - Returns: 目前 最小值的數個 RowData
    private func getMinRowDataSet() -> [T.RowData] {
        /// TODO: sort and get
        return [(sources.first?.getRowData())!]
    }
    
    private func hasNext() -> Bool {
        /// TODO: sort and get
        return false
    }
}

extension Array where Element: RowDataValue {
    func select(by mode: DataProvidingMode) -> Element? {
        switch mode {
        case .diff:
            if count >= 2 && contains(where: { $0.operation == .add }) {
                return filter { $0.operation == .add }.first
            } else {
                return first
            }
        case .full:
            return first
        }
    }
}
{% endhighlight %}

## 小結

以 Data Loader 的角度來說，他已經做好他的工作了，將輸入物件 (sources) 轉換成 讀取出來的物件 (系統支援的模式 .full mode / .diff mode)

接下來思考如何控制 Data Loader 在系統限制下的工作流程
