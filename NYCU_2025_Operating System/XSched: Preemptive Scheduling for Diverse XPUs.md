# XSched: Preemptive Scheduling for Diverse XPUs
## Abstract

### 像 GPU、NPU、ASIC 與 FPGA 這類 異質加速器（XPUs），缺乏彈性的排程能力，因此在多任務環境下，無法滿足豐富的應用需求（例如任務的優先權與公平性）。

* XPU 是廣義用語，指各類異質運算加速器：
    * GPU (Graphics Processing Unit)：圖形與通用運算。
    * NPU (Neural Processing Unit)：針對 AI inference/訓練優化的專用晶片。
    * ASIC (Application-Specific IC)：特定任務定製晶片。
    * FPGA (Field-Programmable Gate Array)：可程式化邏輯陣列。
  
* 傳統的這些 XPU 以 throughput 為導向，設計目標是讓單一任務快速完成。
* 因此，其內部的 命令排程（Command Scheduling） 多為簡單的 FIFO 或 RR（Round Robin），無法做到像 CPU 那樣：
    * 搶佔式（preemptive）執行
    * 依任務重要度（priority）分配資源
    * 針對多個使用者/程序進行公平分時（fair scheduling）
    
* 這導致在多應用同時使用 GPU/NPU 時：
    * 低優先任務會阻塞高優先任務，沒有辦法動態切換資源
    * 延遲（latency）與公平性（fairness）失控
 
### 本文提出了一個名為 XSched 的排程框架，使各類 XPU 都能支援可搶佔式（preemptive）的排程
* XSched 提出了一種多層次的硬體模型
    * Level 1: 可中斷「尚未送出（pending）」的命令；
    * Level 2: 可中斷「已送出但尚未完成（in-flight）」的命令；
    * Level 3: 可中斷「正在執行（running）」的命令。
 
---

## 1 Introduction


### 背景：XPU 在雲端與邊緣的普及

* XPUs（GPU、NPU、ASIC、FPGA）如今無處不在：
    * 雲端：一張 GPU 被多租戶共享以降低成本。
    * 邊緣裝置：同一顆 NPU 同時跑多個 AI 模型（如視覺 + 語音）。
    * 共享 XPU 是趨勢，但也造成 多任務並行 的需求激增

### 問題：XPU 的排程機制過於原始
* 目前多數 XPU 沒有搶佔式（preemptive）排程：多採用 FCFS（先來先服務） 或 RR（Round Robin）
* 結果：
    * 高優先任務被卡住（priority inversion）
    * 公平性問題（fairness issue）
    * S即時性失效（例如自駕車 deadline miss）
 
### 既有解法：軟體層的 Preemptive Scheduler
* 以往研究（如 EffiSha、REEF、TimeGraph）嘗試在 host 端做軟體層排程。
* 但這些方法只能在特定 GPU 或 driver 上運作，缺乏通用性。
* 目前沒有任何 preemptive scheduling 可用

| 挑戰               | 含義   | 問題說明                     |
| ---------------- | ---- | ------------------------ |
| **Portability**  | 可攜性  | 各廠 XPU 架構差異大，軟體方案無法重用。   |
| **Uniformity**   | 統一性  | 沒有跨 XPU 的共用抽象層，難以制定通用策略。 |
| **Evolvability** | 可演化性 | 現有方案與硬體緊耦合，難以支援新架構。      |

---

## Key ideal
* 我們提出一個結合 "unified abstraction" 與 "multi-level hardware model" 的可搶佔式 XPU 排程方法。這種做法能有效遮蔽不同 XPU 在硬體能力與軟體堆疊上的差異與複雜度，從而確保方案的通用性
    * abstraction: 為不同類型的 XPU 提供一致的任務排程介面，從而支持與硬體無關的排程策略，並促成不同 XPU 之間的協作
    * multi-level hardware model: 把「可搶佔」這件事分成幾個等級（Level 1～3）
     
## Our approach
### XQueue: 是一個能被搶佔的 command queue，專門拿來當作 XPU 的排程單位
* 類似於 CPU thread abstraction
* 每個 XQueue 承載一個 XPU 任務，該任務由一系列 command 組成
    
 <img width="708" height="343" alt="image" src="https://github.com/user-attachments/assets/46902924-d7ae-4060-a20d-d25bdb3acf32" />

      
### multi-level hardware model: 把「可搶佔」這件事分成幾個等級（Level 1～3）

| Level   | 可搶佔對象            | 精確定義（對應原文）                       | 典型含義                                |
| ------- | ---------------- | -------------------------------- | ----------------------------------- |
| **Lv1** | **pending** 命令   | 已經 **submitted** 但 **尚未 launch** | 最基本：擋住後續要送去裝置的命令，換別的 XQueue 先跑      |
| **Lv2** | **in-flight** 命令 | 已經 **launched** 但 **尚未執行**       | 能撤回/掛起已下發到裝置但還沒進核心執行單元的命令           |
| **Lv3** | **running** 命令   | 正在執行 的命令                     | 最進階：能中斷正在跑的 kernel / operator，之後再恢復 |


### XSched: 基於我們的統一抽象層與多層硬體模型，我們實作了 XSched —— 一個可搶佔式的 XPU 排程框架。
* 是前面提出的 XQueue + 三層硬體模型 的落地版本。
* 它是一個通用中介層，可以部署在不同硬體（GPU、NPU、ASIC、FPGA）上，負責協調任務執行順序。
* 目的：讓所有 XPU 都能像 CPU 一樣有 preemption（可中斷與恢復）

---
