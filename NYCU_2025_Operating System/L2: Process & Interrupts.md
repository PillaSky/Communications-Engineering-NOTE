
# L2: Process & Interrupts

## What is a Process

### Process
* An active program and related resources, such as
   * Processor state
   * Memory address space

### Process Descriptor
* Contains all information about a process
* 名稱用法:
   * In Linux: Process Descriptor ,`struct task_struct`
   * General name: Process Control Block
* `task_struct`包含哪些資料
   * `pid_t pid`: Process ID，唯一識別值
   * `long state`: 程序狀態（running, sleeping, zombie 等）

 ### Task Lists
 * A list of **process descriptors** in a circular doubly linked lists
    * Circular doubly linked list（循環雙向鏈結串列）
       * 雙向：方便往前/往後找其他 process
       * 循環：不需額外處理 list 的結尾
      
<img width="343" height="281" alt="image" src="https://github.com/user-attachments/assets/d2265893-ee81-43e1-b4e1-82d40ad57cf8" />    
* process: 都有一份 task_struct 代表自己
* task_struct: 是一種儲存 process 所有狀態的資料結構，又稱為 Process Descriptor 、 Process Control Block
* task list: 由一堆 `task_struct` 組成，是一個雙向循環串列

***

## How Kernel Accesses a Process Descriptor
* Kernel Stack: Linux kernel use this **per-thread memory aera** to store (每個 thread 在 kernel space 都有一塊 kernel stack)
    * Intermediate information: 執行期間的「中間資訊」，例如函式呼叫的 return address、參數、區ˋ育變數
    * Metadata block(‵thread_info`): 這個結構存放 **指向 process descriptor,task_struct**的指標
* `thread_info`:
    * stored at the bottom of the kernel stack (Easier to Access)
    * Size of kernel stack: static 2 ~ 3 pages
    * Dereferences the task member of `thread_info` to **get the pointer of the `task_struct`**
 
上面白話來說就是:  
1. 每個 thread 在 kernel space 都有一塊 kernel stack
2. stack 的最底部放了一個小結構叫 `thread_info`
3. `thread_info` 裡有個欄位 task，它是一個指向 `task_struct` 的 pointer
4. 所以只要知道目前執行 thread 的 **kernel stack**，就能進一步找到他的 **`thread_info`**，也就得知該 ``task_struct` 的 pointer
因此就能讓 Kernel access the Process Descriptor

<img width="1248" height="544" alt="image" src="https://github.com/user-attachments/assets/4567ea02-8646-4e74-a0b8-1151bec57a4f" />

***

## Process State
<img width="335" height="281" alt="image" src="https://github.com/user-attachments/assets/81370a39-adb6-4d57-9ac5-68188ca53ce6" />  

* The **state field** of the process descriptor **describes the current condition of the process** 
    * `Task_Running`: Runnable, either **running** or **ready but not running**
    * `Task_Interruptible`: Sleeping (i.e., blocked), waiting for some condition
    * `Task_Uninterruptible`: Sleeping, does not wake up even receivesa signal
    * `__Task_Stopped`: Execution has stopped. (SIGCONT to resume)
    * `__Task_Traced`: Processes traced by a Debugger

### Process State Diagram
<img width="1129" height="916" alt="image" src="https://github.com/user-attachments/assets/d10ff4ad-aaeb-4f82-8078-1b2862d1e571" />    

1. `fork()` 產生新的 process
2. `TASK_RUNNING` (ready but not running)
    * 表示這個 process 雖然準備好執行，但目前還沒被分配到 CPU
    * 在這個狀態中，它會被放在一個「run queue」中，等待 scheduler 來挑選
3. 被排程器挑選執行 → `TASK_RUNNING` (running)
4. 搶佔（Preemption）→ 回到 ready queue
5. 自願睡眠 → `TASK_INTERRUPTIBLE` / `TASK_UNINTERRUPTIBLE`
    * 執行中的 process 如果需要等待某個事件（例如 I/O、lock、訊號等），就會呼叫 sleep 相關函式，此時它會進入「等待」狀態
6. 事件發生，喚醒 → 回到 `TASK_RUNNING` (ready)
7. 終止 → `do_exit()` → 結束生命週期

***

## The Process Family Tree

* The first process: All processes are descendants of the **init process (PID 1)**
* How to accesses one’s parent or children: Use the **parent or the children pointer** provided by the process descriptor 
* How to access other processes: By traversing through the **`task_list`**

<img width="353" height="286" alt="image" src="https://github.com/user-attachments/assets/045a7bb0-6ce2-41e5-b1de-a47d79bf87a2" />

### 白話文版本
1. 所有 processes 的始祖是 `init process`（PID 1）
2. 如何找到自己的 parent / children:   
每個 process 都有一個資料結構叫 task_struct（即 Process Descriptor）。裡面會記錄與其他 processes 的關係
3. 如何存取其他 process？  
所有的 process 都會被加入到一個雙向環狀串列：`task_list`。可以巡覽 `task_struct` 中的 tasks 欄位，來遍歷整個系統的所有 process。

***
## Process Creation
這張投影片講的是 Linux 中 Process Creation（程序建立） 的過程，尤其是搭配 Copy-on-Write (CoW) 技術來提升效能的實作方式。

### Process Creation = `fork()` + `exec()`
* `fork()`:
   * Linux 中要產生一個新的程序（process）時，會透過 `fork()`。這個動作會建立一個**與父程序相同的子程序**
   * 採用這種機制是因為效率高、實作簡單
   * 雖然說是「複製父程序」，但有幾個重要資訊是「不同的」
      * `PID`（Process ID): 每個 process 都要有唯一的識別碼。
      * `Page Table`: 會建立新的，但指向同樣的實體頁面
      * `Page Frames`
* `exex()`: 常見用法為先用 `fork()` 建立子程序，再用 `exec()` 載入真正要執行的程式
* Challenge: 一口氣從 parent process 複製所有資源會非常耗時間，於是就有了 CoW 來解決上述問題

### Copy-on-Write (CoW)
* 一開始父子 process 共用一份記憶體（ read-only），節省空間與複製時間。
* 直到其中一方要修改記憶體時，才會真的分配新的 page frame 並複製資料（觸發 page fault）
* 直到真正需要寫入才複製，這也是「延遲複製」的精神
  
### 正常父行程 P1 的記憶體存取行為
<img width="2048" height="620" alt="image" src="https://github.com/user-attachments/assets/d7b7235c-3c4c-4c91-9c25-b04e7f46fe0a" />  

CPU 想要存取一個資料時
1. 需要先去查 page table，透過虛擬位址 VA ，把 VA 映射成對應的 實體位址 PA
2. 再透過 PA 去 RAM (page frame) 存取資料

### `fork()` + write → Copy Page Table
<img width="2048" height="584" alt="image" src="https://github.com/user-attachments/assets/c1d76550-841e-4b9c-8125-0386737e90de" />  

1. 當我們呼叫 `fork()` 時，系統會建立一個新的 process（P2），它會 複製 P1 的 page table，但實體資料還沒複製
2. 這時候兩個行程 P1 和 P2 都指向相同的 page frame ，並且設定為 read-only

### Copy-on-Write (CoW) 實際發生
<img width="2048" height="635" alt="image" src="https://github.com/user-attachments/assets/844bd39f-9181-4d25-ba8a-8e1718581b50" />  

1. 當 P2 想要**寫入記憶體**時，因為該頁是 read-only 的，就會觸發 page fault
2. 作業系統這時候才會真正複製一份 page frame給 P2，這就是 Copy-on-Write

### Linux Implementation: `copy_process()`
1. Allocate a new kernel stack (e.g., `thread_info`) and process descriptor (`task_struct`)
    * 每個 process 在 Linux 核心中都會被表示成一個 `task_struct` 結構
    * 此結構中包含了該 process 的所有必要資訊：狀態、排程資訊、記憶體位置、PID、開啟的檔案、signal handler 等
    * 為了管理 process 的 kernel stack，Linux 還會為每個 process 配一個 `thread_info` 結構，並與 `task_struct` 共用相同的 page
  
2. Copy or set fields in the process descriptor
    * Copy: 有些欄位是直接複製自父行程，例如 program counter, states
    * Set fields: 有些欄位需要新的設定，例如新的 Process ID (PID)
  
3. Set the state field to `TASK_UNINTERRUPTABLE`
    * 因為剛建立的 process 還沒完全初始化完成，不能被排程執行
    * 所以會先被設定為 `TASK_UNINTERRUPTABLE` 狀態，代表它目前不會被中斷或被 scheduler 切走。

4. Duplicate parent’s page table
    * Linux 使用 Copy-on-Write 技術來實作記憶體複製：**複製的是 page table而非內容**
    * 兩個 process 的虛擬記憶體初期會指向相同的 physical page（並設為 read-only），只有在 write 時才觸發 CoW

5. Duplicate (Process) or Share (Thread):

*** 

## Process vs Thread
### Thread
* 又稱為 **Lightweight Process**
* Executes within the same program in a **shared memory address space**, **shared open files**, and **shared page tables**
    * 所以，多個 thread 能夠同時操作共用的資料區（如 global 變數、heap），這就是為什麼 thread 之間需要額外處理 同步問題（race condition）
* Linux implements all threads as standard processes
    * Each thread have their own `task_struct`
       * Program Counter
       * Registers
       * Stack
       * State
    * 換句話說，在 Linux 裡，thread 只是共享資源的 process
* Linux does not provide any special scheduling or data structures to represent threads
    * Linux kernel 並沒有特別區分「這是一個 thread」或「這是一個 process」
    * 對 scheduler來說，它只看到「一堆具有 task_struct 的任務」，不在乎它們是否共享資源
    * 所以，thread 是一種語意上的概念
* Creating Threads: `clone()` is identical to `fork()`
    * `fork()`: 建立一個完整新 process（所有資源都複製，等要寫入記憶體時再 Copy-on-Write）
    * `clone()`: 依據 flag 決定要共享哪些資源
        * `clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND, 0)`
* Kernel Threads (kernel threads 沒有自己的記憶體空間)
    * Like normal processes (schedulable and preemptable): 和一般 process 一樣，kernel thread 會被 scheduler 排程與中斷；也可被 preempt（強制換 context）
    * But, in contrast to normal processes, **kernel threads do not have an address space**
    * Kernel Threads Operate only in kernel-space and do not context switch to user-space.

### 小結論: Linux 中的 Thread 與 Kernel Thread
#### Thread
* Linux 內部將其視為普通的 process，每個 thread 都有獨立的 task_struct
* 本質是共享資源的 process：透過 `clone()` 系統呼叫並指定特定 flags
* Linux 核心 並不特別標記或調度「thread」，對 scheduler 而言所有任務都是 task_struct，thread 只是語意上的概念

#### Kernel Thread
* 和普通 process 一樣，kernel thread 也是 task_struct，同樣可被排程與中斷（schedulable & preemptable）
* 最大不同點：kernel thread 沒有自己的 address space，只在 kernel-space 執行，不會切換到 user-space
* 常用於處理核心層面的背景任務（如 I/O 處理、驅動、排程器…）
* 不會執行使用者程式碼，無法呼叫 exec()

***

## Process Termination（程序終止）
Termination occurs when **the process or C compiler call `exit()`**
* Linux 核心會進一步呼叫 `do_exit()` 來進行實際的資源釋放與後續清理
* 步驟說明（對應 `do_exit()`）
  1. Sets the PF_EXITING flag in the flags member of the task_struct
     * 當一個 process 要終止時， Linux kernel 會把它的 process 狀態做標記，以讓其他 kernel 元件知道這個 process 正在退出中，不能再進行某些操作
  2. `exit_mm()`: 釋放與此 process 關聯的 memory descriptor,讓記憶體能回收並被其他 process 使用
  3. `exit_sem()`: 將此 process 從 IPC semaphore 等候佇列中移除，防止 zombie process 阻塞 semaphore queue
  4. `exit_files()` and `exit_fs()`
     *  `exit_files()`: 釋放 file descriptor table，並將每個 file 的引用計數減 1
     *  `exit_fs(): 釋放與 file system 相關的資訊
  5. `exit_notify()`: 向 parent process 發送 `SIGCHLD`，將此 process 的 子程序重新指定父程序。
     * 目的是維持 parent-child 關係的正確性，允許 parent 呼叫 wait() 來清理資源。
  6. `schedule()`: 呼叫排程器，切換至另一個可執行的 process
  7. After **the parent retrieves the information**
     * the process’s kernel stack (including `task_struct` and `thread_info`) will be deallocated.

***
## Process Scheduling

### 如何達成 Multitasking
#### Single Processor (multiprogramming)
* 雖然只有一個 CPU，但系統快速切換不同的 process 來執行，造成「看起來像是同時在跑多個程式」
* 意思是在只有一個處理器的系統中，要實現 multitasking ，就需要靠 Time Sharing 的方式來模擬出「好像同時執行多個程式」的效果。

#### Multiple Processors
* 每個處理器可以同時跑不同的 process，是真正的硬體層並行（真正的 parallelism）

---

### 好的排程器應該滿足的三個目標
* Fairness（公平性): 每個 process 都應該有機會使用 CPU，不該有某個 process 一直獨佔 CPU
* Response Time（回應時間）: 應該要快速回應
* Throughput（吞吐量）: 整體系統每秒能完成多少工作（例如完成幾個 process），為系統效率的指標之一

### 根據「誰決定讓出 CPU」來分類
1. Cooperative Scheduling（協同式排程）
   * Process **主動釋放 CPU 控制權**
   * 缺點: 如果一個程式不釋放，整個系統會卡住
2. Preemptive Scheduling（搶佔式排程）
   *  Involuntarily suspend a running process (context switch)
   *  作業系統強制中斷正在執行的 process，而非 process 主動釋放
   *  具體作法是透過「context switch（上下文切換）」來轉交 CPU 給其他 process
   *  Linux 採用此機制
3. 設計 Preemptive Scheduling 的核心概念
   * Timeslice: The time a process runs before it is preempted, prevent some processes monopolizing processors

### Scheduling policy challenge
1. How to decide the timeslice
2. Which process shall be run next

***

## Considerations for Designing Policies (設計排成器的考量因素)
### Process Priority
* nice value
    * larger value represents being micer to other process
    * 解釋: nice 越小（甚至是負值）→ 越不 nice → 優先權越高 → 獲得更長的 time slice
* real-time value
    * All real-time processes have **higher priority** than normal processes
    * 解釋: 實時任務（如音訊播放、控制系統）不能被隨意打斷，因此排程器會保證它們的優先執行權

***

### Timeslice
#### Timeslice 定義
* 每個 process 在被排程上 CPU 時，最多可以連續運行的時間
* 若執行超過該時間，就會被 preempt（搶走 CPU）並 context switch
* Longer timeslice → Fewer context switches → Better Resource Utilization

#### Timeslice 的挑戰
#### Challenge 1:
* Triggers context switch frequently if all processes are **low priority**
* 解釋: 若所有 process 的 nice 值都很大（代表 priority 很低），他們的 time slice 會很小，導致頻繁 context switch

#### Challenge 2:
* **Timeslice difference is huge** for processes 
* 解釋: 意味著有些 process 一執行就是幾百毫秒，有些卻只有 5ms，造成Resource unbalance

#### CFS can solve these challenges, but O(1) cannot

***

### Process Characteristics (程序特性)
1. I/O-Bound Process
    * Spends much of time **submitting and waiting on I/O requests**
    * Example: Graphical User Interface (GUI)
    * Policy: Run more frequently and interactive, **shorter timeslice**
2. Processor-Bound Process
    * Spends much of time **executing code**
    * Example: ssh-keygen, MATLAB
    * Policy: Run less frequently, longer timeslice

### Maintenance Overhead
這部分在探討「排程器內部如何維持與選擇下一個 process」  
* How to find the next process
   1. Queue
   2. Tree
   3. Bit-map
* Runqueue 的設計選擇
   1. Global runqueue：全系統只有一個排程佇列，需加 lock 保護
   2. Per-processor runqueue：每個 CPU core 自己有一組 runqueue，減少 lock contention → 提升效能與 CPU 使用率

***

## Linux Process Scheduler
這段是在介紹 Linux 作業系統不同時期的 Process Scheduler 設計演進  

### O(N) Scheduler（before Kernel 2.4）
* All processors **share a global runqueue** (意思是所有 ready 的 process 都會放進這個 queue，並根據優先權排序)
* 缺點 1. Shall lock the runqueue when a process runs out of its timeslice
* 缺點 2. Processes shall be inserted to an **expired queue**
* Lead to scale poorly when there are tons of processes

### O(1) Scheduler（Kernel 2.5）
* 為了解決上面的 scalability 問題，Linux 在 kernel 2.5 中引入了 O(1) scheduler
* O(1) scheduler 的設計目標：快速找到目前最應該執行的 process（也就是擁有最高優先權的那一個）
* 在 O(1) scheduler 中，每一個 priority 值（0～139） 都有對應的一個佇列（queue）
* 也就是一個 priority 對應一個 queue，queue 裡面放的是擁有這個 priority 的 process。
#### bitmap 優先權索引
1. Check bitmap to get the target priority value (leftmost)
   * 查詢 Bitmap，找出最左邊 bit = 1 的位置，所以這步是「**選出目前最高優先級的非空 queue**」
3. Use the priority value to get (dequeue) the next process (FIFO) in the active array
   * 拿到那個 priority 值之後，就知道要去哪一個 queue，我們就「dequeue」：**把排第一的 process 拿出來執行**
3. Recalculate priority and enqueue to the expired bitarray
   * 當一個 process 用完它的 time slice 後，OS 會依據它的使用狀況（執行時間、priority等）重新計算它的 priority
缺點: Low interactive performance (Low priority processes)

<img width="2349" height="386" alt="image" src="https://github.com/user-attachments/assets/9dc68871-5ca2-40e1-aaab-374613b512a9" />  
* 左圖代表我們的  bitmap ，會在這邊找到優先度最高的 queue
* 右圖是各個 queue 裡面蘊含的 process，我們會從中拿出第一個 process 來執行 (dequeue)

### O(log(N) Scheduler (Kernel 2.6)
* Name: Completely Fair Scheduler（CFS）
* Goal: more interactive(像 GUI 需求) and **more Fair than O(1)**
* Concept 1: each process would **receive 1/n** of the processor’s time (fair share)
* Concept 2: Schedule processes for infinitely small durations (意思是如果要做到理想化的多工，應該是每個 process 都能「無限小時間」地不斷被輪流執行)

#### CFS 的運作邏輯
1. 系統會記錄每個 process 的 **virtual runtime**
2. 用一顆 **Red-Black Tree** ，根據這些 process 目前的 runtime 排序
3. 每次都從樹中，拿出 CPU 用量最少的 process 來執行
4. 執行完就更新該 process runtime
這樣能做到近似公平的排程

#### 甚麼是 Virtual Runtime?
- Theory Vruntime += `delta_exec * (NICE_0_Weight/NICE_Weight)`
    - 我們每次都會挑選**最小`vruntime`的process來執行**
    - 這個`vruntime`會越加越大，差別在於每個process上升幅度不同
    - 每個process都會有它"實際執行的時間"以及它的"nice value"
        - 這個nice value會對應到某個實數，nice value越大則權重越小
    - 根據上面的敘述，我們每次執行完某個process之後，都會在vruntime再加上某個值
        - 如果nice value越小(優先度越高；分母的weight越大)，導致vruntime增加的幅度會更小
        - 反之nice value越大(優先度越低；分母的weight越小)，導致vruntime增加的幅度會更大
        - 於是優先度高的process仍然可以先執行，但不會永遠壟斷CPU，因為它的vruntime依然在上升
    - 執行完該process之後，又會再一次重新挑選vruntime最小的process來執行
- Practical Vruntime += (delta_exec * NICE_0_Weight * 2^32/NICE_Weight) >> 32
        - `delta_exec`: 這次執行了多久
        - `NICE_0_Weight`: nice value=0 的權重 (基準值，通常是 1024)
        - `NICE_Weight`這個 process 的 nice value 對應的權重

- Allocated CPU time
    - `Allocated_CPU_Time = __sched_period() * (NICE_Weight /NICE_k_Weight)`
        - `__sched_period()`: CPU週期長度

### 該如何從nice value計算nice weight，進一步求出 CPU time
- 從 CPU time 的觀點: 以 nice value = 0 為基準，每移動一級就會**增加/減少 CPU 時間**，因此稱為**Follow the 10% effect**
- 從 nice weight 的觀點: weight(n) = 1024 x (1.25)^(-n)
以上兩點是可以互推的

<img width="482" height="609" alt="image" src="https://github.com/user-attachments/assets/b1532391-fa31-400c-979c-cdec94337545" />    

---
前面介紹完了 Linux 作業系統不同時期的 Process Scheduler 設計演進，接著要進一步介紹**即時排程政策**    

## Realtime Scheduling Policies
這段是在說明 Linux 的 Realtime Scheduling Policies（即時排程政策） 的行為  

* Soft Realtime
    * Kernel 嘗試在時間內「調度應用程式」(**Linux 僅提供 soft realtime**)
* Hard Realtime
    * Kernel 保證在時間內滿足任何調度需求
 
---

### Linux 的即時排程政策（Linux Realtime Scheduling Policies）

* Priority Range: 0~99 (Realtime Processes)
    * FIFO (`SCHED_FIFO`):
        * have no timeslice and can run indefinitely
        * Only a high priority **FIFO** or **RR Tasks** can preempt
          
    * RR   (`SCHED_RR`)
        * FIFO with timeslice
        * 即使這個 `SCHED_RR` 任務已經把它的 time slice 用完了，也要等它主動讓出，代表低優先權的任務永遠不能中斷它

* Priority Range:100~139 (Normal Processes)

---

## Process Context Switch
這段落是在解釋 Linux 中的「Process Context Switch（程序上下文切換）」，也就是從一個可執行的 process 換到另一個的過程  

* 簡單的說，'context switch' 就是切換  'context'。而所謂的 'context'，指的就是當下 CPU  registers 的內容。
    * 因為當程式在執行時，所做的事情就是從 memory 中讀取指令和依照指令做計算。
    * CPU 在做這兩件事情時，就是用 core registers (CPU 內部的 register) 來紀錄讀取到哪裡了，以及當作計算時的暫存區域。
    * 當 CPU 做事做到一半，如果有事件發生了，而且這個事件的處理所用到的 core registers 會把目前的進度蓋掉的話，CPU 就會先需要把 core registers 目前的值先暫存到 memory 的某個地方，等 CPU 處理完事件之後，再把暫存的值讀回來，就可以從事件發生時中斷的地方在繼續處理。這就是所謂的 'context switch'。

---

### 「Process Context（程序上下文）」指的是一個 process 執行所需的全部狀態資訊:
* User Address Space: 使用者空間資料，例如：Page Table
* Hardware Registers: 處理器暫存器，例如：PC（Program Counter）
* Kernel Data Structures: 作業系統記錄該 process 的內部資料，例如：PID、開啟的檔案

---

### 「Process Context Switch」就是從一個可執行 process 換到另一個的操作，執行 switch 時，主要分成兩部分: `switch_mm()` 與 `switch_to()`

***

#### `switch_mm()`: Switch the virtual memory mapping 
<img width="976" height="145" alt="image" src="https://github.com/user-attachments/assets/99bf9b06-6c66-4e4e-8e8c-6f8d7b836959" />  

* 比喻：你今天是 CPU，一開始在幫 P1 工作。這時你的（記憶體 page table）是 P1 的資料。但現在你要幫 P2 工作，就必須把桌上地圖換成 P2 的那張地圖，不然你根本找不到他要用的資料。
* 所以 `switch_mm()` 做了什麼？
    * Flush Branch Predictor: 擦掉「剛剛猜的東西」（P1 的預測邏輯）
    * Load New Page Table (CR3): 換成 P2 的記憶體地圖
    * Flush TLB: 清掉「記憶體快取」的資料，避免用到 P1 的陳年舊資料
    * Load CR4: 把新的進階設定裝上（P2 可能有不同記憶體特性）
    * Load LDT:（通常不用）要是 P2 有自己的 segment 資訊也要載入
 
* 結果：你（CPU）現在桌上已經是 P2 的記憶體地圖，準備接手他的任務了

***

#### `switch_to()`: Switch the process state

<img width="1218" height="425" alt="image" src="https://github.com/user-attachments/assets/0fad5fb9-604f-4cab-ad51-5c0f8180ab79" />

* 比喻：想像你現在要交接班：需要從 Process A 交接給 Process B，要把工具、身份證、密碼、暫存器、文件全部換成 B 的。
* 所以 `switch_to()` 做了什麼？
    * Push registers to save: CPU 把目前暫存器的值壓到記憶體，保存 A 的現狀
    * Swap Stacks: 把「堆疊資料」也一起交換，因為不同 process 用的是不同的 stack，這邊也要切換 stack 指標
    * Save FPU, Update TS: 儲存數學運算器狀態
    * Update Ring 0 Stack: 切換內核模式下使用的堆疊
    * Save/Load ES & DS: 保存「區段暫存器」，用來指到 memory segment（記憶體區段），這些設定要從 A 換成 B
    * Update Thread Local Storage: 換掉 Thread Local 的變數表
    * Save/Load FS & GS: 換掉 FS/GS：給 Thread Local Storage 用的特殊暫存器
    * Store User Stack: 保存舊的使用者堆疊
    * Switch PDA Contexts: 切換 PDA，每個 CPU 有一份自己的筆記：IRQ 記錄、中斷資訊、快取、MMU 狀態，這邊也要換成 B 的
    * Update Debug Registers: 如果 A 有設斷點或除錯設定，也要更新成 B 的設定
    * Update I/O Permissions: 切換是否能存取某些硬體裝置的權限表
    * Restore Saved Registers: 把 Process B 的工具全部拿回來放上 CPU，這步完成後，CPU 就是 完全以 B 的身份工作了！

---

## CPU Mode Switch vs Process Context Switch

這部分要特別說明區分 「CPU模式切換」 和 「Process內容切換（Context Switch）」 這兩種看起來類似但完全不同的行為  

### CPU Mode Switch
當 CPU 遇到 interrupt 或 exception 時，CPU 會從 User mode 切換成 Kernel mode
* CPU 只是把目前這個 process 的一些使用者狀態先存起來
* 然後進入 kernel mode 處理事情
* 因此**不會切到別的 process**

### Process Context Switch
把 CPU 從正在執行的 Process A 換成 Process B，交接所有東西  
* 把 A 的全部狀態存起來
* 把 B 的狀態讀出來，套用到 CPU 裡

---


# System Call

## What is System Call

### 目標: 讓使用者空間（user space）的應用程式，可以與作業系統核心（kernel）互動
### 怎麼達成: Kernel 提供一組「系統呼叫接口（system call interfaces）」這些接口可以讓你：

* 開檔案 (open)
* 寫檔案 (write)
* 建立行程 (fork)
* 分配記憶體 (mmap)

<img width="1112" height="127" alt="image" src="https://github.com/user-attachments/assets/cb4fb248-adb8-4eaf-8cfe-a9f4402f3bfc" />  

* 應用程式程式碼呼叫： `printf("Hello")`
* C 標準函式庫： 其實是呼叫 `write()`
* Kernel 內部實作： 實際執行 `sys_write`

所以平常在用的 `printf()`，最終其實會透過底層的 `write()` 系統呼叫，讓 kernel 寫資料到螢幕  

### `sys_call_table`

這是 kernel 中的一張「系統呼叫表格」，裡面列出**每一個 system call 的對應編號**，例如:  
* 編號 0 -> 指令 read -> Kernel 端函式 `__x64_sys_read()`
* 編號 1 -> 指令 write -> Kernel 端函式 `__x64_sys_write()`

---

## System Call Handler

### How to System Call
1. 假設你是一個在 user mode 的應用程式（例如 Process A）你呼叫 `write()`
2. CPU 會 從 user mode → 切到 kernel mode 去幫你跑 system call handler
    * 不是 process context switch❗
  
3. kernel 查 `sys_call_table` 執行對應的功能

### Details for System call handler (該如何通知 Kernel)
現在假設在 User Mode (Process A)
1. 透過 軟體中斷（software interrupt） 的方式
2. 用指令：`INT $0x80`（這是中斷號碼 128）
3. 把你想要叫的 system call number 塞到 `eax` 暫存器裡
4. kernel 就會觸發對應的 system call handler，執行 INT $0x80 這條指令 → 軟體中斷觸發

至此，已切換到 Kernel Mode
1. 進入 System Call Handler
2. 從 `sys_call_table[eax]` 查出對應的函式（例如 sys_write）並執行
3. 執行完之後回到 user mode，Process A 繼續執行

---

## How to Implement a System Call (如何實作一個 System Call)

以把它想像成「你寫一個新功能，希望 user-space 可以叫 kernel 幫你執行」  

* 以下是在 Kernel 內新增 `sys_hello()`的範例:
  
1. 寫一個新的 C 檔

```c
// sys_hello.c
asmlinkage long sys_hello(void) {
    printk("Hello from kernel!\n");
    return 0;
}
```

2. 建一個 Makefile（讓它可以被編譯）

3. 把這個新函式「註冊」進 `syscall table`
要改 `syscall_64.tbl`: 像是加上這行 `436     common     hello    __x64_sys_hello`

4. 加入標頭檔
去 `include/linux/syscalls.h` 加一行

5. 編譯、安裝、重啟kernel

總結： 把一個 C 函式寫進 kernel，📝 註冊到 syscall table，🧩 加到 header，🧱 重編 kernel，就能用 system call 呼叫它  

---

# Inturrupts

## Handling Hardware Events (處理硬體事件)

### How CPUs work with slower hardware devices (CPU 如何和慢速硬體設備溝通?)
* Polling: Kernel 定期主動檢查硬體狀態 
* Inturrupts: 硬體自己發出訊號（signal）通知 kernel
    * 硬體設備實體產生電子訊號(按下滑鼠)，這些「電子訊號」會傳到一個叫 Interrupt Controller 的元件，它會決定是誰要被中斷，然後告訴 CPU
 
### What causes interrupts

我們知道 inturrupt 會發出訊號，告訴 CPU 需要中斷。那甚麼原因會導致 inturrupt?

1. Interrupt（Asynchronous, Hardware Interrupts）
* 硬體設備物理產生電子訊號，這些訊號被引導至 interrupt controller （例如鍵盤、磁碟）的 input pins
* 之後會觸發 CPU 執行對應的中斷處理程式

2. Exception（Synchronous, Software Interrupts）
* 由 CPU 自己產生，發生在執行某個指令的當下
* 通常是錯誤、例外或特殊事件引起

---

## IRQ (Interrupt Request, 中斷請求)

* 不同的裝置（像是鍵盤、滑鼠、硬碟）都可以對 CPU 發出中斷，但要讓 CPU 能區分是誰發的，就要給每個中斷一個獨一無二的編號（interrupt value）
* Linux/ x86 架構中最多可以有 256 個中斷編號（0~255）。這個設計就像 system call 一樣：用編號來辨識要執行的功能
   * 每個 IRQ（中斷號碼）都會對應到一個「特定的裝置」，在系統啟動時，kernel 就已經知道這些對應關係
       * IRQ 0 -> timer interrupt
       * IRQ 1 -> keyboard interrupt
       * Dynamically Allocate IRQ # -> PCI devices

---

## How an Interrupt Work

1. 硬體裝置送出 interrupt signal (例如你按下鍵盤，就會觸發IRQ1)
2. CPU 停止目前正在執行的程式
3. Interrupt controller 把中斷號碼 **INT#** 傳給 CPU (例如 IRQ1 被轉換成 INT 0x81)
4. CPU 利用中斷號碼，去中斷向量表找對應的服務程式位置
5. CPU 開始執行中斷服務程式（ISR: Interrupt Service Routine）

<img width="374" height="396" alt="image" src="https://github.com/user-attachments/assets/760935af-f8bc-4578-aeba-8e590908b8bc" />  

* 小補充：為什麼要 ×4？因為每個中斷向量項目佔 4 bytes
---

## Inturrupt Handler

### Interrupt Service Routine (ISR) 是什麼？

* 當某個硬體裝置產生中斷（例如鍵盤被按下），Kernel 會跳到一段程式去處理這個事件
* 這段程式就叫做 中斷服務程式，簡稱 ISR
    * 它通常是驅動程式（Device Driver）的一部分：例如鍵盤驅動、網路卡驅動裡就會包含對應的 ISR
    * 這些程式碼是由 kernel 管理的，屬於 kernel space

### ISR is non-blockable (Cannot Sleep)
* ISR（中斷處理）不能 sleep
* 因為「中斷」本質上就是一個極度緊急的事件，像是要處理硬體發出 interrupt
* 如果你在 ISR 裡 sleep，就會卡住整個 CPU


### 將 ISR 拆分為兩部分處理

* Top Halves: Run immediately upon receipt of the interrupt and performs only the work that is time-critical
    * Top Half 是在中斷發生當下執行的👉 必須非常快、不能阻塞
    * 所以把重要的、急的事情留在 Top Half 做
        * 一收到硬體中斷 → 立刻執行、做最基本的清理、資料轉移
 

* Bottom Halves: Work that can be performed later is deferred until the bottom half
    * 等 Top Half 做完。不是立即執行、而是 deferred（延後）
    * 不影響即時性，但把事情做完
        * 處理封包內容（例如解析 HTTP 請求）
     
* 舉例說明:
    * Top Half = 火災現場搶救人命，最急要立刻做！
    * Bottom Half = 事後調查失火原因、報案記錄、開罰單 → 不急，稍後做就好



