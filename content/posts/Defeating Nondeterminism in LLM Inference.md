---
title: "Defeating Nondeterminism in LLM Inference：讓大語言模型的推論真的「一模一樣」"
date: 2025-09-22T10:30:00+08:00
draft: false
description: "Count-Min Sketch algorithm"
tags: ["LLM", "深度學習"]
categories: ["技術分享"]
showToc: true
TocOpen: true
showComments: true
---

## 目錄

- [目錄](#目錄)
- [什麼是推論中的非決定性（Nondeterminism）](#什麼是推論中的非決定性nondeterminism)
- [為什麼「temperature = 0」＋同一 prompt 還是得不到固定結果](#為什麼temperature--0同一-prompt-還是得不到固定結果)
- [非決定性的真正來源（不是你以為的浮點 + GPU）](#非決定性的真正來源不是你以為的浮點--gpu)
- [解法：batch-invariant kernel + 固定 reduction 路徑](#解法batch-invariant-kernel--固定-reduction-路徑)
- [實驗觀察：效果與 trade-off](#實驗觀察效果與-trade-off)
- [優缺點分析](#優缺點分析)
- [結語](#結語)

---

## 什麼是推論中的非決定性（Nondeterminism）

當我們使用大型語言模型（LLM）做 inference 時，希望「同一個模型 + 同一個 prompt + 同樣設定」每次產生的輸出都是 **完全一樣**。

但實際上，不少情況下即便把隨機性關掉（例如把 temperature 設為 0，即 greedy decoding），輸出還是可能不同。

這叫做推論的非決定性：結果因為某些底層細節而改變，而這些細節通常在使用者／API 層看不到。

---

## 為什麼「temperature = 0」＋同一 prompt 還是得不到固定結果

我們直覺認爲，把 sampling 關掉，直接選最高機率 token，就該 deterministic，但現實中常常不是這樣。

[Thinking Machines Lab][1] 的文章指出：

* 即使在開放原始碼的 inference 工具（像 vLLM）上，在同一硬體、同一權重、同一 prompt，greedy decoding 有時候也會有不同的輸出。
* 常見解釋是：「浮點運算不結合 (non-associativity)」 +「多線程／GPU 核心執行順序不一定」導致加法、累加的順序每次不一樣。這導致小數誤差累積，不同的 token 的 logit 就會有微小差異。

---

## 非決定性的真正來源（不是你以為的浮點 + GPU）

Thinking Machines Lab 認為，雖然浮點 non-associativity 和 GPU 的 concurrency 是部分原因，但它們並不是整體 story 的核心。

他們指出：

1. **大部分 kernel 在 Forward Pass 中其實沒有 atomic add**。atomic add 是那種多個 thread 同時加到某個記憶體位置，順序不一定的操作。雖然 atomic add 是不可決的，但在 LLM 的主要推論流程中，這樣的操作用得並不多。

2. **batch size / 請求併發量的變化** 是關鍵因素。當 inference server 接收請求時，為了效率，會把幾個請求打包（batch）一起處理；載入量／併發情況不一樣，batch size 就不一樣；這個 batch size 的不同會改變內部 kernel 的運算順序或使用哪一種 reduction strategy（例如分割或 chunking、split-K、block size 等），進而造成浮點累加順序的改變。 

3. **batch-invariance 的缺失**：也就是說，如果同一個樣本在不同 batch 裡被處理時，其內部 reduction（加總、RMSNorm normalization、attention 裡的 sequence + feature dimension reduction 等）運算的順序／切分 (splitting) 方法不同，那麼結果就會不同。要達到真正 deterministic，就得保證這些 operation 不管 batch size 或 shape 怎麼變，對單一樣本（或 token）內部的運算過程是同樣的。

---

## 解法：batch-invariant kernel + 固定 reduction 路徑

Thinking Machines Lab 提出了具體的設計和實作改進方案：

| 對象                                | 改進方法／策略                                                                                                                                                                                                                                   |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **RMSNorm**                       | 確保 normalization 的 reduction 在不同 batch size 下採用相同的 reduction 順序，不會因為 batch 很小就換用不同策略。
| **Matrix Multiplication（MatMul）** | 不論 batch size 或 shape 是什麼，都使用統一的 kernel 設定，不用動態切分策略 (split-K) 或因 shape 小省略某些最佳化。這樣確保每次同樣的輸入，accumulation、tile 切分／block 分配等步驟是一樣的。)                                                                            |
| **Attention + KV Cache**          | Attention 的 prefill + decode 階段、sequence 長度與 KV cache layout 的變化會導致 blocked split 或 masked block 的數量不同，進而影響 reduction 順序。他們提出固定 split-size 策略：不要固定「要分多少塊」而是固定「每塊多大」，這樣不管你有幾個 tokens、cache 有多少，都保持同樣的分塊／運算順序。 |

這些都是為了達成 **batch-invariant**：意思是「對單個請求來說，其輸出不應該因為別的請求是否一起 batch 處理，或因為 batch size 怎麼變動而變」。只有保證這樣，才能在整體系統中看起來 deterministic。

---

## 實驗觀察：效果與 trade-off

以下是他們做的一些實驗以及觀察到的效果／代價：

| 測試項目                                                            | 預設情況（未修正）                          | 啟用 batch-invariant kernel 後 | 備註／觀察                   |
| --------------------------------------------------------------- | ---------------------------------- | --------------------------- | ----------------------- |
| **Qwen-3-235B，temperature=0，重複 1000 次同一 prompt（約 1000 tokens）** | 產生 **80 種不同** completion 結果        | 所有輸出 **完全一致**               | 非決定性差異完全消除              |
| **Qwen-3-8B，API 伺服器情境，輸出長度約 90–110 tokens，1000 sequences**      | 較快（使用 vLLM 預設最佳化 kernel）           | 有 **一定 slowdown**，但仍可接受     | 效能下降幅度依硬體／batch size 而異 |
| **Attention kernel**                                            | 分塊／masking 隨序列長度改變，reduction 順序不一致 | 固定 split-size，順序保持一致        | 效能比預設稍慢，但輸出穩定性大幅提升      |


---

## 優缺點分析

**✅ 優點**

* 可重複性（Reproducibility）大幅提升：在科研、調試、RLHF、log-prob 比較這類事情上，結果更穩定，不會因為 batch load 或請求併發而莫名其妙跑掉。
* 系統／工具的可信度更高：如果使用者知道每次 call 是 deterministic 的，那麼 debugging、追踪錯誤、保證一致性會簡單很多。
* 測試、benchmark、比較模型／方法時，更公平／準確，不會被隱藏的變因干擾。

**❌ 缺點**

* 效能／速度折扣：為了保 batch-invariance，有些 kernel 必須省略或限制某些 shape／batch size 的最佳化，或不能動態切分／block 分配策略，這會讓在小 batch 或特殊形狀時比較慢。
* 實作複雜性提升：要把所有可能影響 reduction 順序的地方都審查（包括 KV cache layout、split block、mask 處理等）；也要確保未來版本／硬體／軟體更新不會重新引入 batch size 對 reduction 路徑的依賴。
* 資源成本與工程投入：kernel 的開發、測試、維護成本，以及可能的硬體或指令集限制。


---

## 結語

推論的非決定性長久以來被當成「無可避免」的技術細節，常被簡化為「GPU 多線程 + 浮點誤差」那樣。

但透過 Thinking Machines Lab 的研究，我們可以看到：只要把 batch size / 請求併發負載等這些看似外在的變因納入考量，設計 batch-invariant 的 kernel，確保 reduction 路徑固定，就是真的可以把這些非決定性問題「打敗」。

當然，要完全沒落差、速度依然像以前一樣快，目前還有挑戰，但在很多研究／產品場景下，穩定一致的輸出價值遠大過那段延遲或性能折衷。對於一個講究精準、可靠、科研可重複性的 ML 工程師來說，這是一個值得且必須的過程。

[1]: https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/?utm_source=chatgpt.com "Defeating Nondeterminism in LLM Inference"
