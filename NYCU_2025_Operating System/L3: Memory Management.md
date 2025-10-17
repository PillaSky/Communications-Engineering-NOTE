# L3: Memory Management

## Agenda

* Physical Memory Management（實體記憶體管理）
    * 基本概念：Page vs Page Frame
    * 核心如何管理物理位址（分為 Page、Zone、Node）
    * 記憶體分配策略：
        * First Fit（最先符合法）
        * Best Fit（最佳符合法）
        * Worst Fit（最差符合法）
        * Buddy System
        * Slab Allocator（適合小物件分配）
        
* Virtual Memory Management（虛擬記憶體管理）
    * VMA（Virtual Memory Area）概念
    * CPU 如何走 page table（查表映射）

* Advanced Research Topics

---


## Basic Concepts of Memory Management

* Page frame: 將 **Physical address space** 切分成大小相同的區塊
* Page      : 將 **Virtual address space** 切分成大小相同的區塊

---

* Kernel 負責管理所有的 page frame 以及 page table

<img width="447" height="206" alt="image" src="https://github.com/user-attachments/assets/c5bdb5e2-efea-42f5-822b-57d8cee04709" />   

當程式需要記憶體（例如呼叫 `malloc()`），作業系統核心會: 

```markdown
1. P1 calls `malloc()`                         ← 使用者程式發出請求
2. Kernel allocates page frames                ← 找一塊實體記憶體（稱為 page frame）
3. Kernel updates page tables                  ← 幫它建立對應的虛擬位址映射（virtual → physical）
```

---

<img width="1433" height="408" alt="image" src="https://github.com/user-attachments/assets/ec0690d1-e918-436c-9e89-48511792b1c2" />

分成兩大區段，每個格子都是實體記憶體位址的 Page Frame：
1. Kernel Address Space (Low Mem): Kernel 自己要用的記憶體，例如page talble
2. User Address Space: 可以分配給使用者程式用的空間（如呼叫 malloc() 時要占用的）

---

<img width="1175" height="430" alt="image" src="https://github.com/user-attachments/assets/d475a4d7-adb9-4af3-b865-7c8434647474" />  
* 所有的 process 都會共用同一塊實體記憶體
    * 會透過 page table ，來讓多個 process 可以存到同一個PAS裡面
    * 此外每個 process 會各自有自己的 page table，即便不同的 process 在 VAS 內的位址都相同，但仍可透過各自不同的 page table 來把 VA 映射道不同的 PA

* 當我們想要把 VA 映射到 PA 時
    * 首先會透過 TLB 快取；如果沒有的話就會透過 MMU 查詢該 Process 的 page table
      
---

## Manage Physical Address 如何管理實體位址 (Page frame)

* 雖然 CPU 每次可以精確存取「一個位元組（byte）」
    * 但 MMU（Memory Management Unit） 在執行虛擬地址 ➝ 實體地址轉換時，是以「頁（page）」為基本單位
 
* page frame 指的是 DRAM 中一個固定大小（通常 4KB）的實體記憶體區塊
    * Linux 核心會幫 每一個 page frame 建立一個對應的 struct page 結構，用來記錄該頁框的狀態
    * struct page 是針對實體記憶體的資料結構，不是虛擬記憶體（VMA）
 
* 每一個 struct page 大約需要 40 bytes、假設系統有 4GB DRAM，假設每個 page frame 是 4KB
    * 那總共會有 1,048,576 page frames
    * 需要的 metadata 大小就是：1,048,576 × 40 bytes = 40MB
    * 所以在 4GB DRAM 下，只需要 40MB 的額外資料來管理所有 page frame
 
---

## Manage Kernel Physical Address (Zone)

<img width="713" height="151" alt="image" src="https://github.com/user-attachments/assets/6948d86c-ab24-411a-affb-d82e63fb6b23" />  

* kernel 會將 physical address space 依照功能分成不同區塊 (稱為 Zone)，主要分為以下三個區域:
    * ZONE_DMA: 可進行 DMA（直接記憶體存取）的 page frame (<16MB)
    * ZONE_NORMAL: Kernel 最常使用的記憶體 (16MB~896mb)
    * ZONE_HIGHMEM: 記憶體太高無法直接對映，需 Dynamicclly mapping (>896MB)
 
* 上表僅適用於 32-bit 系統（如 x86-32）
    * 在 64-bit 系統（x86-64）中因為虛擬位址空間變超大（理論上可支援數十 TB）
    * 所以不再需要 ZONE_HIGHMEM，所有實體記憶體都可以直接對映進 Kernel
 
---

## Manage Kernel Physical Address (Nodes)

<img width="1314" height="346" alt="image" src="https://github.com/user-attachments/assets/8e807726-a067-43f0-99f0-e9011d878fed" />   

### 什麼是 NUMA？
* NUMA（非一致性記憶體存取架構）是一種 多 CPU 系統的記憶體配置方式
    * 圖裡有兩個 CPU，各自連著一個 memory controller 和一組 DRAM → 這一組就叫做一個 NUMA node
    * CPU 可以快速存取「自己所屬 node 的 DRAM」→ Local Access
    * 如果存取別的 node 的記憶體 → Remote Access（大約慢兩倍）
 
* Why NUMA?
    * Fast local access → 每個 CPU 存自己對應的記憶體很快，效能好
    * Higher bandwidth → 每個 CPU 有自己的 DRAM，整體通道變多了，頻寬更高
    * Easy to scale out → 若要加 CPU，只要加一組 node 即可，架構非常適合擴充
 
<img width="1597" height="431" alt="image" src="https://github.com/user-attachments/assets/22b4957a-b85f-4644-a08e-78a5e772aeae" />

### Node
* `pg_data_t`
    * Linux 為每個 NUMA node 建立一個 `pg_data_t` 結構
    * 負責管理該 node 上的「記憶體分配狀況」
 
* 在 NUMA 每個 CPU 與某一個 DRAM 距離比較近，當作業系統需要幫一個 process 配 memory 時，會優先提取近的那個
* 資料結構層級（Granularity): Nodes → Zones → Page Frames
    * Node: 對應 NUMA node，也就是整塊 DRAM
    * Zone: Node 內部依照記憶體用途再細分的區域，如 ZONE_DMA, ZONE_NORMAL, ZONE_HIGHMEM
    * Page frame: 每個 zone 裡是由 4KB 大小的 page frame 所組成
 
---

## Basic Concepts of Memory Allocation

* 最初，所有記憶體都是可用的，就像是一整塊大洞 (hole)
    * 當有 process 要分配記憶體時，系統就要從所有目前的 hole 中「找一個夠大的」給它。
    * 所以關鍵在於：搜尋哪一個 hole？用什麼策略
      
* 三種記憶體分配策略
    * First Fit: 搜尋並分配第一個足夠大的空洞 （浪費記憶體大小）
    * Best Fit: 搜尋所有空洞，並分配最小且足夠大的空洞 （浪費時間）
    * Worst Fit: 搜尋所有空洞，並分配最大的空洞 （浪費記憶體大小和時間）

* Fragmentation (Low Utilization)記憶體無法有效利用的兩種情況:
    * External Fragmentation: 所有可用空間總和大於某個 process 所需要，但因為這些空間不連續所以無法配給該 process 使用，造成 memory 空間閒置。(解決方法: paging)
    * Internal Fragmentation: 作業系統配置給 process 的 memory 空間大於 process 真正所需的空間，這些多出來的空間該 process 用不到，而且也沒辦法供其他 process 使用，形成浪費
 
---


## Linux Page Allocation

* `alloc_pages()`: 分配一個或2^order個 page
    * 具體上要如何分配page呢? 則會透過 Buddy system 把記憶體切成 2^0, 2^1, ..., 2^N 大小的區塊（例如 1, 2, 4, 8, 16 頁）

* `kmalloc()`: For more general byte-sized allocations


### Buddy System

* alloc_pages(order) 會配置 2^order 個連續實體頁面（例如 order=2 表示請求 4 個 page，連續）


<img width="1403" height="846" alt="image" src="https://github.com/user-attachments/assets/d3290f68-fe01-4f0e-8380-092432579c54" />

* 每個 `free_area_t` 存有 對應大小的空閒區塊清單
    * Linux kernel 會根據你請求的 order，去對應的 free_area[order] 找是否有空閒區塊

* 假設一個 process 想要分配一塊 大小為 2² = 4 pages 的連續實體區塊，但系統檢查發現符合大小的空間都被占用：
    1. order=2（4 pages）❌ 無空閒
    2. order=3（8 pages）❌ 無空閒
    3. 找 order=4 有空閒，取出一塊 2⁴ page block
    4. 分裂成兩塊 2³ block，一塊用來繼續切，一塊丟回 order=3
    5. 再分裂成兩塊 2² block，一塊給 process，另一塊丟回 order=2
 
* 至此，我們就可以透過 2⁴ page block 不斷切割，最終形成大小為 2² = 4 pages 的連續實體區塊分配給 process


---

<img width="563" height="395" alt="image" src="https://github.com/user-attachments/assets/bc8dd9c7-a2c3-4f1e-8015-b2a907352ee3" />  

#### 左邊：User-space Memory Allocation

* `malloc()：` 你在 C 程式碼中寫的 malloc()，是呼叫 C Library 中的 memory allocator。
    * 根據所需大小，會選擇呼叫：
    * brk()：用來調整 heap 區大小（小 allocations）
    * mmap()：直接對 kernel 請求映射一塊頁面（大 allocations）


#### 右邊：Kernel-space Memory Allocation

* `kmalloc()`: 核心程式中，如果要分配「小型記憶體區塊」（小於一頁），會走 kmalloc()
* `vmalloc()`: 若要配置「大塊但非連續實體位址」的記憶體，會使用 vmalloc()

#### 中間：Low-Level Memory Allocator

* Buddy System 是 Linux 核心中實體頁面配置的主要演算法。
* 它以 page 為基本單位，使用 2 order 來分配

#### 最底層：Physical DRAM

* 所有的分配，最終都會落到實體記憶體 page frame 上，分配實際的位址空間

---

## Slab Layer

