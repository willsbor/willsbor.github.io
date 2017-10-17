---
layout: post
title: NSHash​Table & NSMap​Table
--- 

http://nshipster.com/nshashtable-and-nsmaptable/

http://www.cocoachina.com/ios/20150519/11809.html

[weakObjects()](https://developer.apple.com/documentation/foundation/nshashtable/1412241-weakobjects)

看了一下別人的 [code](https://jobandtalent.engineering/ios-architecture-an-state-container-based-approach-4f1a9b00b82e) 看到了這個關鍵至 NSHash​Table

```
NSSet(NSMutableSet)持有其元素的強引用，同時這些元素是使用hash值及isEqual:方法來做hash檢測及判斷是否相等的。
NSHashTable是可變的，它沒有不可變版本。
它可以持有元素的弱引用，而且在對像被銷毀後能正確地將其移除。而這一點在NSSet是做不到的。
它的成員可以在添加時被拷貝。
它的成員可以使用指針來標識是否相等及做hash檢測。
它可以包含任意指針，其成員沒有限制為對象。我們可以配置一個NSHashTable實例來操作任意的指針，而不僅僅是對象。
```

突然想到之前

有時候未遇到類似的問題，當我有一個 superviser 要關注多個物件時，物件會像這個 superviser 註冊，

之前都常常是用 NSSet 之類的 strong 連結物件，所以該物件消失時，還得向這個 superviser 說我要消失了，

如果使用了 NSHashTable 可以使用 weak reference，當被專注的自行 dealloc 時，直接被 nullable 而移除，

似乎這層關係就簡單多了
