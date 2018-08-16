---
layout: post
title: 變數資料與檔案儲存
--- 

思考一個常常遇到的問題：

> 物件中有一個變數資料，當下次再開啟 app 時，需要是上次的數值

先用很直覺的方式處理它

{% highlight swift linenos %}
struct Student: Codable {
    var id: String
    var name: String
    var age: Int
}

class Foo {
    let filePath = "path/to/file"
    var currentStudent: Student? {
        get {
            let decoder = JSONDecoder()
            let data = Data(contentsOfFile: filePath)
            return = try? decoder.decode(Student.self, from: data)

        }
        set {
            let encoder = JSONEncoder()
            let data = try? encoder.encode(newValue)
            data.write(to: URL(fileURLWithPath: filePath))
        }
    }
}
{% endhighlight %}

Cool!

但是一般來說，這個如果資料常常被讀取，被修改的機會比較少，我們通常不會這樣寫。加上如果只考慮單一 process 使用時，可能只要考慮開始結束儲存就好了


```swift
struct Student: Codable {
    var id: String
    var name: String
    var age: Int
}

class Foo {
    let filePath = "path/to/file"

    var currentStudent: Student?

    func load() {
        let decoder = JSONDecoder()
        let data = Data(contentsOfFile: filePath)
        currentStudent = try? decoder.decode(Student.self, from: data)
    }

    func save() {
        let encoder = JSONEncoder()
        let data = try? encoder.encode(currentStudent)
        data.write(to: URL(fileURLWithPath: filePath))
    }
}
```

`save()` `load()` 分別在 launch 和 terminate 的時候呼叫就好了

---------------------------------------

## 分開「資料被使用的邏輯」和「提供資料的方法」

回過頭來，上面的問題換個方式說

> app 內有一個 memory cache，當下層的 file 資料沒有變化時，都會拿取 cache，當 file 資料變化時，就會通知 memory 更新; 如果 memory cache 資料有變化，也會寫道 file 中。

對最上層的物件 Foo 來說，其實他根本不關心資料是怎麼來的，他只要能存取到一個 student 的資料就行了。因此，我們可以用這個概念，將功能先分開。

```swift
struct Student: Codable {
    var id: String
    var name: String
    var age: Int
}

class Foo {
    let filePath = "path/to/file"

    var studentData = DiskCacheProvider<Student>()
    var currentStudent: Student? {
        get {
            return studentData.getValue()
        }
        set {
            studentData.setValue(newValue)
        }
    }
}

class DiskCacheProvider<T> {
    func getValue() -> T {

    }

    func setValue(_ newValue: T) {

    }
}
```

這樣 `DiskCacheProvider` 只要負責「資料提供的邏輯和方法」。

---------------------------------------

## 再細分「資料提供的邏輯和方法」

仔細想想 `DiskCacheProvider` 要做什麼事情
 - 何時存取 Cache / File 的邏輯
 - 存取 Cache 資料的方法
 - 存取 File 資料的方法

至少需要上面的幾個事情才能做到整個功能。

`存取 Cache / File 資料的方法` 可以分別獨立來實作。在實作前，先將概念再抽取出來：

> 一個有拿取數值和儲存數值的抽象介面

可以設計成：

```swift
public protocol SynchronizedDataProviding: class {
    associatedtype Value
        
    func getValue() throws -> Value
    
    func setValue(_ value: Value) throws
}
```

這裡用 `Synchronized` 是先考量 memory or file 之類的實作體，網路需要等待的介面先不考慮

`何時取用 Cache / File 的邏輯` 可以看作是：

> 何時存取 [資料來源1] or [資料來源2] 的邏輯

所以可以設計成下面的狀況：

```swift
public class SynchronizedStorageChain<MainProvider: SynchronizedDataProviding, UnderlyingProvider: SynchronizedDataProviding> where MainProvider.Value == UnderlyingProvider.Value {
    
    public let provider: MainProvider
    public let underlyingProvider: UnderlyingProvider?
    
    private var isUnderlyingProviderBeUpdated = false
    
    public func getValue() throws -> MainProvider.Value {
        if isUnderlyingProviderBeUpdated, let underlyingProvider = underlyingProvider {
            let newValue = try underlyingProvider.getValue()
            valueBeModifiedHandler?(newValue)
            try provider.setValue(newValue)
        }
        
        return try provider.getValue()
    }
    
    public func setValue(_ value: MainProvider.Value) throws {
        try provider.setValue(value)
        try underlyingProvider?.setValue(value)
    }
    
    public init(_ provider: MainProvider, underlyingProvider: UnderlyingProvider) {
        self.provider = provider
        self.underlyingProvider = underlyingProvider
    }
}
```

可以看到 `setValue` 描述了當要設定過數值，除了儲存到 Main Provider，也會儲存到 Underlying Provider。

`getValue` 描述：如果 Underlying Provider 資料有變化，則會讀取 underlying 的資料，更新 main provider。

接著就要處理 `isUnderlyingProviderBeUpdated` 何時會被變動的問題了。

意思是 underlyingProvider 所屬的型態 SynchronizedDataProviding 應該要能告知物件變數該改變了。

所以最後可以設計成：

```swift
public protocol SynchronizedDataProviding: class {
    associatedtype Value
    
    var valueBeModifiedHandler: ((_ newValue: Value) -> Void)? { get set }
    
    func getValue() throws -> Value

    func setValue(_ value: Value) throws
}
```

而流程面實作可以寫成：

```swift
public class SynchronizedStorageChain<MainProvider: SynchronizedDataProviding, UnderlyingProvider: SynchronizedDataProviding> where MainProvider.Value == UnderlyingProvider.Value {
    
    ...
    
    public init(_ provider: MainProvider, underlyingProvider: UnderlyingProvider) {
        self.provider = provider
        self.underlyingProvider = underlyingProvider
        self.underlyingProvider?.valueBeModifiedHandler = { [unowned self] (_) -> Void in
            self.isUnderlyingProviderBeUpdated = true
        }
    }
}
```

最後實際應用會長成下面這個樣子：

```swift
class StudentMemoryProvider: SynchronizedDataProviding {
    typealias Value = Student
    private var value: Student

    ...
}

class StudentFileProvider: SynchronizedDataProviding {
    typealias Value = Student
    private var fileManager: FileManager
    let filePath = "path/to/file"

    ...
}

class Foo {
    let filePath = "path/to/file"

    var memoryProvider = StudentMemoryProvider
    var fileProvider = StudentFileProvider()
    var studentData
     = SynchronizedStorageChain<StudentMemoryProvider, StudentFileProvider>(memoryProvider, underlyingProvider: fileProvider)

    var currentStudent: Student {
        get {
            return try! studentData.getValue()
        }
        set {
            try! studentData.setValue(newValue)
        }
    }
}

```

---------------------------------------

## 最後

這樣的思考流程，主要是希望能盡量符合單一職責原則 (SRP)，提升內聚力，降低耦合性。對 `Foo` 來說他知道下面三個物件 

 - `StudentMemoryProvider` 
 - `StudentFileProvider` 
 - `SynchronizedStorageChain`

把相依性放在這最上層，將他們設定、結合起來。

而 `SynchronizedStorageChain` 只知道介面，所以並不相依於 memory or file system

這樣在測試會有比較簡單的切入點，並且在未來擴充功能時，可以分別考量每個介面要對應的物件改變，減低大規模的改變風險。

目前這個架構放在 [SynchronizedStorageChain](https://github.com/willsbor/SynchronizedStorageChain) 有興趣可以去看看！

