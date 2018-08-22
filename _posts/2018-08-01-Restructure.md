---
layout: post
title: Restructure 思考脈絡 (1)
--- 

## 重新思考程式架構

> 要想很多？還是足夠就好？

最近看了 clean architecture 還蠻多省思，有空再分享一下，先來做一下我一直想要來處理的事情。

過去我常常會為了專案，「未來」「可能」會發生的功能，對元件保留功能，或設計擴充的可能性。但往往時間過去了，當時的設想，往往不會如願，這些保留的擴充性，反而成為了阻礙。

要如何保有彈性，但又不會設計的太遠離現實？

> 重新一步一步把目標貼近現實，
> 過程中將「商業(現實)邏輯」和「功能」區分
> - 對「商業邏輯」貼緊目標做設計
> - 對「功能」保留未來擴張的權利

## 目標
先用一個廣泛、抽象的命題開始，之後會慢慢地收斂回現實

> 將多個來源的資料，合併成一份資料後匯入系統

對多個來源 `sources`，透過 `mergedData()` 產生一個合併過後的資料

{% highlight swift linenos %}
protocol DataType {
    func adding(_ data: DataType) -> DataType
}

class DataLoader<T: DataType> {
    
    var sources: [T] = []
    
    func mergedData() -> T? {
        
        let result: T? = sources.reduce(nil) { (last, item) -> T? in
            if let last = last {
                let merged: T = last.adding(item) as! T
                return merged
            } else {
                return item
            }
        }
        
        return result
    }
}
{% endhighlight %}

------

> 將多個來源的資料，合併成一份資料後匯入系統
>
> 資料內有多筆細目，細目數量可能很大，且 process 的 memory 大小有限制

增加了一個限制，所以原先程式碼已經無法符合現在個規範。

當資料量很大時 ex: 20 萬、100 萬，之類，直接回傳一個合併好的陣列，似乎在記憶體有限的狀況下，不是個好方法

也可以想像當 source 資料很多時，也不見得會是 memory，可能會是一個 file，或是其他 DB 來源

因此透過讀檔案的方式，將多筆資料讀出後，再依序交給呼叫者處理

{% highlight swift linenos %}
protocol DataType {
    associatedtype RowData
    
    func open()
    func next() -> Bool
    func getRowData() -> RowData
    func close()
}

class DataLoader<T: DataType> {
    
    var sources: [T] = []
    
    func readMergedData(listMaxSize: Int, _ handler: ([T.RowData]) -> Void) {
        
        var temp: [T.RowData] = []
        
        sources.forEach { (source) in
            source.open()
            while source.next() {
                temp.append(source.getRowData())
                
                if temp.count == listMaxSize {
                    autoreleasepool {
                        handler(temp)
                        temp = []
                    }
                }
            }
            source.close()
        }
        
        if temp.count > 0 {
            autoreleasepool {
                handler(temp)
                temp = []
            }
        }
    }
}
{% endhighlight %}

------

> 將多個來源的資料，合併成一份資料後匯入系統
>
> 資料內有多筆細目，細目數量可能很大，且 process 的 memory 大小有限制
>
> 資料使用其中一欄做排序，由小到大，且唯一，
>  - 如果遇到值一樣的欄位，則取用在 array 中比較前面的 source 

繼續增加條件。資料的樣貌也有初次的描述。

資料需要能夠被排序，因此每筆資料要能夠 `Comparable`

sort and merge 的演算法則是先定義一個框架 `getMinRowData() -> T.RowData` & `hasNext() -> Bool`

主要是 `readMergedData` 能夠符合最外層邏輯的期待

{% highlight swift linenos %}
protocol RowDataType: Comparable {
    
}

protocol DataType {
    associatedtype RowData: RowDataType
    
    func open()
    func next() -> Bool
    func getRowData() -> RowData
    func close()
}

class DataLoader<T: DataType> {
    
    var sources: [T] = []
    
    func readMergedData(listMaxSize: Int, _ handler: ([T.RowData]) -> Void) {
        
        sources.forEach { (source) in
            source.open()
        }
        
        var temp: [T.RowData] = []
        while hasNext() {
            temp.append(getMinRowData())
            
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
    
    private func getMinRowData() -> T.RowData {
        /// TODO: sort and get
        return (sources.first?.getRowData())!
    }
    
    private func hasNext() -> Bool {
        /// TODO: sort and get
        return false
    }
}
{% endhighlight %}


下一篇： [Restructure 思考脈絡 2](2018-08-22-Restructure-2.md)
