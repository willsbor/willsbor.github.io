---
layout: post
title: iOS App Delegate
--- 

在所有專案中，App Delegate 代表的就是程式的開頭(不考慮系統層，只觀察應用端)。系統所提供的方式只是一個 sync 的 callback，所以原則上你沒辦法阻止系統要你的應用程式在各個可能的狀態間轉移( active / inactive / background / termiate )，這是合理的，但是實務上卻有令人煩惱的地方

- async function 
- 上個狀態的事情還沒做完
- 多次進出

因此，應該要對自己的系統設計自已的狀態轉移機制，來避免 async 的工作造成的影響，這也是確保 UI / Main Thread 不會被阻擋，而讓使用者對頁面感到延遲。

如果從狀態來考慮
應該會長的類似像這樣
![AppDelegateState](https://willsbor.github.io/images/AppDelegateState.png =250x)

大致上的概念是
- 只要目前的狀態還沒有完成，也就不會馬上轉移狀態
- 當下狀態的工作完成後，會繼續檢查目前 app 實際要到達的狀態是否是現在的這個狀態，如果還沒有到達，則會依照規則前往

在配合 protocol 的定義，應該可以將之前亂糟糟的 AppDelegate 整理乾淨....

Finish Date: 2017-07-12