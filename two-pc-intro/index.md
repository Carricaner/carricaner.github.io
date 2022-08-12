# 跨伺服器交易處理 — 認識二階段提交 ( 2PC )


<!--more-->

## 前言 <a id="preface"></a>
由於現代方便的網路，使用者透過手機或電腦就能夠進行個人化的操作。

當一位使用者進行轉帳，目的是從 A 帳戶轉到 B 帳戶，開發者在背後是如何處理的呢？

這篇文章主要會列舉並討論這種情境下會遇到的一些狀況。

<br>

進入本文前的幾點事項：

- 本文只列舉幾個種狀況來做『 概括性 』的討論，不涉及實作。
  
- 針對跨伺服器 ( 或分散式 ) 的事務處理還有其他種方式，本文主要針對二階段提交的部分討論。
  
- 文中每個節點 ( 伺服器 ) 都只會使用一個資料庫存放資料。

<br>

{{< figure src="images/unplash-atm-spot.jpeg" caption="*Photo by Eduardo Soares in Unsplash*">}}

<br>

## 本文 <a id="main-content"></a>

再討論情境問題之前，我們先來了解下面兩個概念:
- 交易 ( Transaction )
  
- 二階段提交 ( two-phase commit, 2PC )

<br>

### 交易 ( Transaction )  <a id="transaction"></a>

交易是指複數個對資料庫的操作包成的一個邏輯單位，並且同時需要具備以下兩點特徵：

- 包含的複數個資料庫操作必須是一起成功執行 ( commit ) 或是一起不執行 ( rollback )。
  
- 每個事務之間不會互相干擾 ( Isolation )。

<br>

### 二階段提交 ( two-phase commit, 2PC ) <a id="2pc-intro"></a>

二階段提交是實現跨服務間交易的一種演算法 ( 接下來都以 2PC 代稱 )。

設計 2PC 需要一個事務協調者 ( transaction coordinator, 簡稱 TC )，其負責與各個參與者 ( cohort ) 溝通來滿足交易的基本條件。

整個過程間分為兩個步驟：**詢問** & **提交**。

<br>

流程：

1. 使用者向 TC 提出交易需求
   
2. TC 詢問各個 cohorts
   
3. cohorts 鎖定各自的資源 & 回傳 Yes
   
4. TC 下達 commit ＆回覆使用者 OK
   
5. cohorts 執行交易邏輯並回傳 done

<br>

示意圖：
{{< mermaid >}}
graph LR;
   A([Client])-->|1. Require| B{TC}
   B-->|2. OK?| C[(cohort A)]
   B-->|2. OK?| D[(cohort B)]
   C-->|3. Yes!| B
   D-->|3. Yes!| B
   B-->|4. Commit| C
   B-->|4. Commit| D
   B-->|5. Done!| A
{{< /mermaid >}}

<br>

舉一個有趣例子，西方教堂舉辦婚禮的時候：
1. 牧師分別問新郎以及新娘：你是否願意... 不管生老病死都... ? 
   
   :(fas fa-arrow-right): 協調者詢問參與者們
   
2. 新郎新娘分別回答：願意！
   
   :(fas fa-arrow-right): 參與者們回覆確認並且鎖定一生資源！？
   
3. 牧師：在此宣佈.. 正式成為夫妻！
   
   :(fas fa-arrow-right): 協調者提交

<br>

### 情境 <a id="scenarios"></a>
使用者想要從 A 銀行帳戶 ( 簡稱 A 帳戶 ) 轉帳 100 元到 B 銀行帳戶 ( 簡稱 B 帳戶 )。

有幾點注意 & 假設：
- 只考慮餘額充足的情況 ( A 帳戶餘額 > 100 元 )
  
- 假設 A, B 銀行都各自只用一個資料庫儲存資料。
  
- 必須滿足交易的基本原則。
  
- 假設有一個協調者管理這筆交易 ( 稱為 TC )
  
- A 帳戶 :(fas fa-arrow-right): cohort A
  
  B 帳戶 :(fas fa-arrow-right): cohort B

<br>

直觀的做法就是：
1. 先去 A 帳戶進行扣款 100 元
   
2. 再去 B 帳戶進行存款 100 元
   
3. Done!

<br>

等等... 事情好像進行得過於順利...？ 

那接下來討論幾個狀況

<br>

#### 狀況一：帳戶 B 在交易過程中存款失敗 <a id="conditio-1"></a>

情境會演變成 A 帳戶已經扣除了 100 元，但是 B 帳戶卻沒有存到款的情形...

<br>

套用 2PC 的解決程序：
1. 使用者向 TC 提出交易需求
   
2. TC 向帳戶 A, B 詢問可否進行交易 
   
3. TC 收到帳戶 B 的失敗回覆
   
4. TC 向帳戶 A, B 傳送放棄的命令 ( abort ) 
   
5. TC 回傳轉帳失敗的訊息給使用者。
   
<br>

示意圖如下：
{{< mermaid >}}
graph LR;
   A([Client])-->|1. Require| B{TC}
   B-->|2. OK?| C[(帳戶 A)]
   B-->|2. OK?| D[(帳戶 B)]
   C-->|3. Yes!| B
   D-.->|3. No ...| B
   B-->|4. Abort| C
   B-->|4. Abort| D
   B-->|5. Fail!| A
{{< /mermaid >}}

<br>

雖然交易失敗，但是確保了跨服務間資料的一致性。

<br>

#### 狀況二：TC 遲遲沒有收到帳戶 A 的確認回覆 <a id="conditio-2"></a>

會演變成整筆交易停滯，遲遲沒有反應...

<br>

套用 2PC 的解決程序：
1. 使用者向 TC 提出交易需求
   
2. TC 向帳戶 A, B 詢問可否進行交易 

3. TC 在一定時間時間後沒有收到帳戶 A 的回覆
   
4. TC 向帳戶 A, B 傳送放棄的命令 ( abort ) 
   

5. TC 回傳轉帳失敗的訊息給使用者。

<br>

示意圖如下：
{{< mermaid >}}
graph LR;
   A([Client])-->|1. Require| B{TC}
   B-->|2. OK?| C[(帳戶 A)]
   B-->|2. OK?| D[(帳戶 B)]
   C-.->|3. No response| B
   D-->|3. Yes!| B
   B-->|4. Abort| C
   B-->|4. Abort| D
   B-->|5. Fail!| A
{{< /mermaid >}}

<br>
   
這樣一來，也可以確保後台的資料一致性。

<br>

#### 狀況三：帳戶 A, B 皆向 TC 回傳確認訊息，但遲遲沒有收到 TC 的最終決定 <a id="conditio-3"></a>

我們以帳戶 B 的角度來看這種狀況：
1. 可以單方面執行 commit 指令 ？不行 :x: 
   
   :(fas fa-arrow-right): 因為帳戶 A 可能傳給 TC 的確認指令是 No。
   
   :(fas fa-arrow-right): 會演變成帳戶 A 沒有執行但是帳戶 B 有執行的狀況...
   
2. 可以單方面執行 abort 指令 ？不行 :x: 

   :(fas fa-arrow-right): 因為帳戶 A 可能傳給 TC 的確認指令是 Yes。
   
   :(fas fa-arrow-right): 會演變成帳戶 A 有執行但是帳戶 B 沒有執行的狀況...

<br>

那兩種行動都不能做，該怎麼辦？
1. 繼續等... ( 期盼奇蹟會出現 :sloth: ...)
   
2. 實施 **超時終止協議** 
   
   :(fas fa-arrow-right): 簡單來說就是各個 cohort 之間互相確認狀況，因為涉及的狀況複雜，本文不討論

<br>

一般 cohorts 回覆 TC 之後就會鎖定各自的資源不要讓其他的邏輯干擾，但是現在 TC 卻沒有下達執行的命令，所以造成可怕的現象： **整個交易停滯 ...**

<br>

遇到這樣的問題，我們可以試試 **三階段提交**

<br>

### 三階段提交 ( two-phase commit, 3PC ) <a id="3pc-intro"></a>
簡單來說，就是將二階段提交的第一個步驟 ( 詢問 ) 拆成：詢問 + 鎖定資源，最後才是下達執行的命令，所以總共有三個階段組成。

三階段提交的核心思想是詢問時不鎖定資源，除非所有人都同意才鎖定資源。

整個流程會是：

1. 使用者向 TC 提出轉帳的需求
   
2. TC 詢問各個 cohorts
   
3. cohorts 都返回 Yes
   
4. TC 下達 pre-commit
   
5. cohorts 鎖定各自資源並且都回傳 Yes
   
6. TC 下達最終 commit ＆ 回覆使用者 OK
   
<br>

示意圖：

{{< mermaid >}}
graph LR;
   A([Client])-->|1. Require| B{TC}
   B-->|2. OK?| C[(帳戶 A)]
   B-->|2. OK?| D[(帳戶 B)]
   C-->|3. Yes!| B
   D-->|3. Yes!| B
   B-->|4. Pre-commit| C
   B-->|4. Pre-commit| D
   C-->|5. Lock, Ready!| B
   D-->|5. Lock, Ready!| B
   B-->|6. Commit| C
   B-->|6. Commit| D
   B-->|6. Done!| A
{{< /mermaid >}}

再回來看狀況三的情形，我們可以設計成當帳戶 A,B 一段時內間沒有收到來自 TC 最終的 commit 訊息時就釋放資源並且取消操作，這樣一來也可以保持資料的一致性。

<br>

### 兩軍問題 ( Two General's Problem ) <a id="two-general-problem-intro"></a>

兩軍問題是一個思維性實驗問題，狀況如下：

- 有兩支軍隊，分別由一位將軍領導，共同目的是要擊潰某個敵方城池。
  
- 兩軍派出的信使必須要通過敵方領地，並且信使有可能被敵方俘虜。
  
- 一定要兩軍同時發起進攻才會成功，只有一支軍隊進攻會導致失敗。

<br>

發現不管設計怎麼樣的方法都無法保證兩軍一定可以在同一時間進攻，所以這個問題基本上是無解的。

兩軍問題就是對現代每個節點使用網際網路彼此進行溝通的影射。

我們沒有完美的辦法，但是可以思索出可接受的辦法。

<br>

{{< youtube s8Wbt0b8bwY>}}

<br>

## 總結 <a id="conclusion"></a>
- 二階段提交是一種用來達成分散式交易的設計方式。
  
- 二階段提交分為「 詢問 」以及「 提交 」兩個階段。
  
- 二階段提交需要有協調者 ( Transaction Coordinator ) 以及參與者 ( Cohort )。
  
- 協調者只是一個抽象的角色，實際上可以把它放在某一個參與者之中來實作也是 OK。
  
- 分散式事務最常遇到的狀況分類為：重啟 & 響應超時 ( timeout )，其中後者最為複雜麻煩。
  
- 三階段提交分為「 詢問 」,「 鎖定資源 」&「 提交 」三個階段。
  
- 三階段提交可以用來彌補二階段參與者確認回覆即鎖定資源造成等待的窘境。
  
- 建立在不可靠的溝通渠道的兩軍問題根本上是無解，只能設計較能接受的處理方式。
  
- 現實中實作 2PC 或是 3PC 都將會是很大的挑戰。

<br>

## 參考 <a id="references"></a>
- [二階段提交](https://zh.wikipedia.org/wiki/%E4%BA%8C%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4)
   
- [兩軍問題](https://zh.wikipedia.org/wiki/%E4%B8%A4%E5%86%9B%E9%97%AE%E9%A2%98)
   
- [正确理解二阶段提交（Two-Phase Commit）](https://blog.csdn.net/lengxiao1993/article/details/88290514)

- [分布式系统的事务处理](https://coolshell.cn/articles/10910.html)


