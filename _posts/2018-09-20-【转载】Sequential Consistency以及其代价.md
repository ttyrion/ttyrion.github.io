---
layout:     page
title:      【转载】Sequential Consistency以及其代价
subtitle:   
date:       2018-09-20
author:     翼
header-img: image/bg.jpg
catalog: true
tags:
---

## 转载说明
这篇文章是转载的，作者博客地址[在这里](http://opass.logdown.com/pages/about-me)，原文链接[在这里](http://opass.logdown.com/posts/809449-concurrency-series-5-the-price-of-sequential-consistency) 。

## 备注
此篇文章讲内存顺序一致性模型，讲的不错。不过文中提到的 **Dekker's Alogrithm** 是错的，并不是真正的 Dekker's Alogrithm。此文里的代码有潜在的 **starvation** 问题。真正的 Dekker's Alogrithm 是能够保证互斥访问的同时，还避免 deadlock 和 starvation 问题的。算法伪代码如下，详情可参考wiki：[Dekker's algorithm](https://en.wikipedia.org/wiki/Dekker%27s_algorithm) 。  
```
variables
    wants_to_enter : array of 2 booleans
    turn : integer

wants_to_enter[0] ← false
wants_to_enter[1] ← false
turn ← 0   // or 1


p0:                                          p1:
   wants_to_enter[0] ← true                     wants_to_enter[1] ← true
   while wants_to_enter[1] {                    while wants_to_enter[0] {
      if turn ≠ 0 {                                if turn ≠ 1 {
         wants_to_enter[0] ← false                    wants_to_enter[1] ← false
         while turn ≠ 0 {                             while turn ≠ 1 {
           // busy wait                                 // busy wait
           
         }                                            }
         wants_to_enter[0] ← true                     wants_to_enter[1] ← true
      }                                            }
   }                                            }
                                              
   // critical section                          // critical section
   
   ...                                          ...
   turn ← 1                                     turn ← 0
   wants_to_enter[0] ← false                    wants_to_enter[1] ← false
   // remainder section                         // remainder section
   
    
```

## Memory Consistency Models
我們前面系列提及到，實際上程式在編譯與執行的時候，不一定會真的照你所寫的順序發生。而是可能改變順序、盡量最佳化，同時營造出彷彿一行一行執行下來的幻象，只要實際的結果和照順序執行沒有差別就好。這樣的幻象要成立，在於程式設計師和該系統（硬體、編譯器等產生、執行程式的平台）達成了一致的協定，系統保證程式設計師只要照著規則走，程式執行結果會是正確的。但什樣叫做正確？正確的意思不是保證只會發生一種執行結果，而是定義在所有可能發生的執行結果中，哪些是允許的。我們把這樣的約定稱為Memory Consistency Models，系統要想辦法在保證正確的情況下，盡可能的最佳化，讓程式跑的又快又好。

Memory Consistency Models存在於許多不同的層次中，像是組合語言跑在硬體上時，因為處理器可以做指令重排和最佳化，雙方得確保執行結果和預期相同。或者，在將高階語言轉換成組語時，因為編譯器能夠將組合語言重排，雙方也得確保產生的結果和預期一致。換言之，從原始碼到最後實際執行的硬體上，大家都必須做好約定，才會跑出預期的結果。

## 最直覺的約定，Sequential Consistency
在1970年代，Lamport大大就在思考這個問題了。他提出一個如今最常見的Memory Consistency Model: Sequential Consistency，並且定義如下：  
<
A multiprocessor system is sequentially consistent if the result of any execution is the same as if the operations of all the processors were executed in some sequential order, and the operations of each individual processor appear in this sequence in the order specified by its program.

我們可以分成兩個觀點來看Sequential Consistency的定義:  
1. 對於每個獨立的處理單元，執行時都維持程式的順序(Program Order)
2. 整個程式以某種順序在所有處理器上執行

Lamport的定義濃縮的很精煉，對於第一次看到的人會抓不太他想表達的重點，因為這實在是太蠢、太顯而易見了。第一點講你的程式在處理器內會照順序跑，第二個講所有處理器會以某種順序執行你的程式。你一定覺得，幹這不是廢話嗎XD。之所以會這樣覺得，是因為你一直以來都活在這樣的世界，就像活在牛頓時代以前的人，覺得拿手上的東西放開就會掉下來一樣自然。接下來我們會告訴你，想要保證這樣的現象，在現代的處理器上會限制很多最佳化的手段，讓程式執行的沒那麼快。如果你同意放棄一些約定，例如不保證每個處理單元維持程式執行的順序，我們還能榨出更多效能出來。

讓我再額外補充一點，Memory Consistency Model只是一個幻象的約定，程式執行的結果必須看起來是這樣，但是實際程式編譯完、跑在硬體上，想怎麼改變執行順序都可以，只要結果和約好的定義相同就好。

[sc1](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/sc/sc1.png)   

## 確保執行順序
我們將用這張圖來闡述Sequential Consistency的兩個要點。上圖左邊是Dekker's Alogrithm，一個關於critical section的演算法。如果我們確保Sequential Consistency，在每個處理器核心內維持program order，那麼這個程式就能確保同一時間只有一個處理器進入critical section。因為你一定是先立Flag，再檢查對方Flag是否立起，如果沒有才進入Critical Section。  
但想像另一個情況，同一個處理器他可能直覺的認為Flag1, Flag2兩個變數沒有相依性，因此就算違反SC調換執行順序也沒差。如果P1和P2都先執行第二行，才執行第一行，那麼就會發生同時進入critical section的窘境。

## 確保對所有處理器都是一致的
上圖右邊，當P1執行A=1的效果發生後，P2進行if判斷為真，於是執行B=1，P3執行if判斷B等於1為真，最後把register1寫入A的值。這樣的程式要保證register1讀到A的值是1，前提是P1寫入共享變數A=1後，P2和P3都能保證讀到A=1，也就是確保整個程式以某種順序在所有處理器上執行，對每個處理器而言，都能看到其他先執行的指令所發生的效果。  
我們接下來看三個典型的範例，就算是沒有cache的硬體架構，稍不注意也可能會違反Sequential Consistency。  


## Write Bufers with Bypassing Capability
這個範例會告訴我們維持Write->Read順序的重要性。  
[sc2](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/sc/sc2.png)   
如上圖左，每個處理器都有自己的write buffer，程式在執行時處理器可以先寫到Write Buffer，晚點再寫到Memory上。

我們使用最佳化的手法，當處理器寫入write buffer時，不等待寫入到記憶體完成，直接繼續執行下面的程式。而接下來若發生read，只要讀的位址不是write buffer內等待寫入memory的位址，就允許讀取。這在單核心處理器上是個很常見的最佳化手法，不用等待耗時的寫入就繼續執行，可以縮短等待的時間。

但這種做法會導致違反Sequential Consistency，看看上圖右邊的程式，假設程式雖然看起來是一行一行執行下來，但實際上執行write時，是先寫到Buffer上，然後直接允許下一行read從主記憶體讀取。因此實際程式對記憶體的操作，會是上圖左的t1(讀取Flag2)->t2(讀取Flag1)->t3(寫入Flag1)->t4(寫入Flag2)，兩個Flag都讀到0，統統進入critical section，並且違反SC。

## Overlapping Write Operations
這個範例會告訴我們維持Write->Write順序的重要性。

[sc3](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/sc/sc3.png)   
假設在一個有多個記憶體模組，沒有bus且彼此互向連結的系統上，因為沒有bus，所以執行時不需要照順序執行，而是可以同時執行多個操作。我們假設處理器一樣照著程式的順序發出write請求，而且不等待前一個執行完畢，就直接發出下一個請求。

在執行右邊的程式時，如果遵守SC，應該可以看到Data會是最新的值2000。但這個架構上並不保證發生，因為P1寫入Data和Head時，可能會發生Head先抵達記憶體，Data後抵達記憶體的情況。因此實際的操作可能變成t1(寫入Head成功)-->t2(讀取Head為1)-->t3(讀取Data讀到舊的值)-->t4(寫入Data成功)，變得完全違反SC了。

在單處理器上，對於寫入不同的位址，修改寫入的順序是不會有大問題的，只要維持data的相依性就好。但在此處的範例就可能出狀況，想要解決這樣的問題，必須等待上一個write完成之後，也就是等待acknowledgement response，才發出下一個寫入的請求。

## Non-Blocking Read Operations
這個範例會告訴我們維持Read->Read, Read->Write順序的重要性。  
[sc4](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/sc/sc4.png)   
這也是一個常見的最佳化，允許我們更改讀取的順序。假設P1很乖，程式都照順序寫入，但P2不等待read Head完成就繼續發出read Data的請求。就有可能發生t1(Read Data先回傳結果為0)-->t2(寫入Data 2000)-->t3(寫入Head 1)-->t4(讀取Head為1)，產生違反SC的結果。

## Cache Architechture 與 Sequential Consistency
上述三個是很典型的案例，要是程式存取記憶體的順序發生改變，可能會違反Sequentical Consistency。在有設計cache的系統內，也會遭遇上述的問題。

對於有cache的系統架構，同一份資料可能會存在於多個處理器的cache上。如果想要維持Sequential Consistency，系統要確保同一份資料在不同處理器的cache上保持一致，不然有的處理器讀到比較新的資料，有的讀到比較舊的，每顆處理器所見到的行為不一致，很容易就違反SC。就算是發現要讀取的資料剛好在cache內，也不能立刻讀出來，必須要確保前一個操作完成，才能進行讀取。

## Cache Coherence and Sequential Consistency
我們需要一個機制確保不同處理器上的cache在系統中保持一致，稱之為cache coherence protocol。如果cache coherent protocal夠嚴格，那麼我們就可以保證這個系統上的程式不會出現違反Sequential Consistency的結果。

我們可以把cache coherent protocal想像成 "所有的寫入最終都會被所有的處理器看見", 以及"寫入相同的位址的順序對於所有處理器而言都是一致的"（因此寫入相同位址不會出現交換順序的情況），如果想要遵守Sequential Consistency，還要確保"寫入不同位址的順序對於所有處理器而言都是一致的"（因此所有寫入不會出現交換順序的情況）。

## Detecting the Completion of Write Operations
想要維持Program Order，代表我們需要確保上個寫入完成了，才能執行下一個指令。因此我們需要從記憶體模組收到一個完成的信號代表該次寫入完成。對於沒有cache的系統來說很簡單，就從主記憶體回傳信號即可。

但對於有cache的系統，所謂的寫入完成，真正的意思是，對所有處理器而言都能看到新的寫入值，因此必需要確保每份cache都被正確的更新或是無效化(必須重新從記憶體抓正確的值出來)

## Maintaining the Illusion of Atomicity for Writes
在把新的值更新到每個cache上時，要知道這樣的操作並不是atomic的，並不是一瞬間，所有的cache統統都更新完成。可能有的cache會先被更新，有的之後才更新。

[sc5](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/sc/sc5.png)  
如上圖，假設P1和P2都照著Program Order執行，但要是寫入A=1和A=2用不同的順序抵達P3和P4，就會發生register1和register2讀到兩個不同的值的情形。例如P3看到的是A=1 A=2 B=1 C=1, P4看到的是A=2 A=1 B=1 C=1，使得P3, P4明明是讀取相同的A值，卻出現不一致的情形。避免這種狀況的方式是確保"寫入相同的位址的順序對於所有處理器而言都是一致的"。

[sc6](https://raw.githubusercontent.com/ttyrion/ttyrion.github.io/master/image/sc/sc6.png)  
但只保證寫入相同位址時所有處理器都看到同樣的更新順序是不夠的。回到一開始的圖右邊的程式碼範例，P1寫入A=1，假設P2已經可見A=1，於是執行B=1，但是對P3來說，還沒收到A=1的修改，但是已經看到B=1的修改。於是便讀出A的值為0。
想要避免這種情況發生，在讀取一個剛寫入的值之前，必須要確保所有的處理器的cache都正確的更新，如此一來對所有處理器來說，整個程式的順序就一致了，就能夠滿足Sequential Consistency。

## 小結
這些東西說破不值錢，但對於第一次接觸Memory Consistency Model和Sequential Consistency的人而言，一開始要理解並不容易。但不用緊張看久了就有fu了，SC雖然容易理解，但其實限制了很多最佳化的手段，如果我們可以放寬對Sequential Consistency的依賴，就可以讓程式跑得更快，後續我們會往更weak的memory consistency model邁進。
