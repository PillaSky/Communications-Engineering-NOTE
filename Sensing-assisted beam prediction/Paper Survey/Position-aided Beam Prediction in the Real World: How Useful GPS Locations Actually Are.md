# Position-aided Beam Prediction in the Real World: How Useful GPS Locations Actually Are?

J. Morais, A. Bchboodi, H. Pezeshki and A. Alkhateeb, "Position-Aided Beam Prediction in the Real World: How Useful GPS Locations Actually are?," ICC 2023 - IEEE International Conference on Communications, Rome, Italy, 2023, pp. 1824-1829, doi: 10.1109/ICC45041.2023.10278998.

---
## 閱讀動機

情境有兩個部分:  
1. 真實世界: 在 Indoor 使用 mmWave Base station 發射三種形狀的beam
2. 虛擬世界: 創造一個虛擬的 Indoor 場景，並使用 NVIDIA Sionna，並把真實世界的那三種 beam 在虛擬世界裡刻出

接著我會把「少量量測點」當成訓練資料，讓模型學到「位置/環境 ↔ 最佳 beam（或候選 beam）」之間的對應關係，之後在沒量測過的位置做推論（beam prediction）  

## Abstract
* 毫米波（mmWave）通訊系統必須依賴窄波束（narrow beams），才能獲得足夠的接收信號功率。  

* 然而，要調整／對準這些波束通常需要付出很大的訓練開銷（training overhead）；而在高度移動的應用情境下，這種開銷會變得特別關鍵。如果能利用使用者的位置資訊，就可以幫助做波束選擇（beam selection），進而降低毫米波波束訓練所需的開銷。
* 過去的研究大多只用合成資料（synthetic data）來探討這個問題，而這些合成資料並不能精準地代表真實世界的量測結果。

* 在本文中使用商用現成 GPS所取得的真實世界量測資料，重新檢視位置輔助的波束預測（position-aided beam prediction）問題，並藉此歸納出：在實務上到底能省下多少波束訓練開銷  
    * 也會比較那些在合成資料上表現很好、但在真實資料上無法泛化的演算法，並思考是哪些因素導致推論性能退化
    * 此外，我們提出一個新的機器學習評估指標，使其能更好地反映最終通訊系統的目標(例如：接收功率、SNR、吞吐量、可靠度)。

* 本研究的目標，是在位置輔助的波束對準（position-aided beam alignment）這個議題上，縮小真實世界與模擬之間的落差

### 補充說明
1. 毫米波為何需要窄波束？
摘要在點出 mmWave 的基本特性：mmWave 頻段路徑損耗大、遮蔽敏感，因此若不把能量「集中」在特定方向，接收端收到的功率可能不足，鏈路品質就會差。

2. 窄波束的意思是什麼？
這裡的窄波束通常指：基地台用多天線陣列做 beamforming，把主瓣變得很窄、指向性很強。主瓣越窄代表能量越集中，方向不對就會掉很多功率。

3. synthetic data 是什麼？
通常指：用模擬（ray tracing、幾何通道、統計通道）或人為生成的資料，來做「位置→beam」的訓練與測試。

---

## Introduction

為了發揮 mmWave 頻段所能提供的卓越資料率，通訊系統被設想會使用大型天線陣列來對抗路徑損耗，為了讓這類天線陣列所形成的窄波束能被最佳化地對準到正確方向，現行系統必須承擔很高的波束訓練開銷（beam training overhead），因此就引出一個問題：毫米波通訊系統能否在「不花費無線通訊資源」的情況下，做出波束對準的決策

* 毫米波系統對於 LOS 鏈路的依賴，促使學術界利用使用者位置資訊來輔助波束對準 [2]–[4]
* mmWave 通常很仰賴 LOS 傳輸路徑。正因為 LOS 與「幾何位置」高度相關（LOS 是否存在、方向大致在哪裡），所以學術界想到：用使用者位置，幫助推斷應該對準哪個 beam

就作者所知，位置輔助的波束對準在真實世界情境中仍尚未被驗證
* 文獻 [2] 中，作者使用 SVM 在多使用者、multi-cell 的模擬環境中進行波束對準，但其位置資訊是假設為完美的
* 文獻 [3] 使用深度神經網路，並以合成的使用者設備位置與朝向資料來降低波束對準的開銷。
* 文獻 [5] 提出一種資料庫／查表（lookup table）的方法來做位置輔助波束對準。這不是訓練一個複雜模型，而是用「位置→beam」的對應表/資料庫，把過去收集到的資料整理成 mapping，部署時直接查詢輸出 beam，GPS 位置仍然被視為完美無誤
* 文獻 [6] 的作者評估了多種機器學習方法，並且考慮了位置誤差。然而，用高斯雜訊（Gaussian noise）來建模真實世界的 GPS 誤差可能並不合適，因為高斯雜訊缺乏空間相關性（spatial correlation），而且也很容易在時間上被平均掉

---
透過模擬所得到的亮眼結果，引出了這樣一個問題：在真實世界中，若使用商用現成的 GPS 與毫米波通訊系統，是否也能達到相近的效能？然而，針對「位置輔助波束預測（position-aided beam prediction）」的真實世界研究仍相當稀少。理解「真實世界」與「模擬」之間到底差在哪裡，是縮小學術界與產業界落差的一個關鍵步驟。基於上述考量，本研究嘗試回答以下問題：  
1. 那些在合成資料（synthetic data）上表現很好的演算法，在真實世界資料上表現如何？
2. 在真實資料中，是哪些因素會讓傳統的「位置輔助波束對準（position-aided beam alignment）」方法效能下降？
3. 在真實資料中，如果根據 GPS 位置來做波束預測，我們究竟能省下多少波束訓練開銷（beam training overhead）？
4. 在通訊系統中，用來評估「波束預測」機器學習模型的良好指標（metric）應該是什麼？

---

* [2] M. Arvinte, M. Tavares, and D. Samardzija, “Beam management in 5gnr using geolocation side information,” in 2019 53rd Annual Conference on Information Sciences and Systems (CISS), 2019, pp. 1–6.  
* [3] S. Rezaie, C. N. Manchon, and E. de Carvalho, “Location- and ´orientation-aided millimeter wave beam selection using deep learning,” in IEEE International Conference on Communications, 2020, pp. 1–6.  
* [4] Y. Heng and J. G. Andrews, “Machine learning-assisted beam alignment for mmwave systems,” IEEE Transactions on Cognitive Communications and Networking, pp. 1–1, 2021.
* [5] V. Va, J. Choi, T. Shimizu, G. Bansal, and R. W. Heath, “Inverse multipath fingerprinting for millimeter wave v2i beam alignment,” IEEE Transactions on Vehicular Technology, vol. 67, pp. 4042–4058, 2018.
* [6] Y. Wang, A. Klautau, M. Ribero, M. Narasimha, and R. W. Heath, “Mmwave vehicular beam training with situational awareness by machine learning,” in 2018 IEEE Globecom Workshops, 2018, pp. 1–6.
* [7] A. W. Soundy, B. J. Panckhurst, and T. C. Molteno, “Enhanced noise models for gps positioning,” in 2015 6th International Conference on Automation, Robotics and Applications (ICARA), 2015, pp. 28–33.
* [8] M. Joao, “Position-aided beam prediction,” github.com/jmoraispk/ Position-Beam-Prediction, 2021.
* [9] A. Alkhateeb, G. Charan, M. Alrabeiah, T. Osman, A. Hredzak, N. Srinivas, and M. Seth, “DeepSense 6G: Real-world multi-modal sensing and CSI dataset for 6G deep learning research,” available on arXiv, 2021. [Online]. Available: https://www.DeepSense6G.net

---

## System Model

* 基地台具有 𝑁 根天線，並且用單天線的 UE 進行通訊
* 基地台在傳輸時，會從它的 codebook 𝐹={𝑓𝑚}𝑚=1~𝑀 中選擇 𝑀 個波束成形向量之一 𝑓𝑚∈𝐶𝑁×1 來發送
* the received symbol in the downlink is given byy = hTfmx + n

## Problem Formulation

* 採用「接收功率（receive power）」作為效能衡量指標時，所謂最佳波束選擇，就是 BS 挑選那個能使接收功率最大的波束成形器
    * 理想的波束成形器 𝑓⋆ 可以透過下式求得：f⋆ = argmax|hTf|2
    * 在這項工作中，我們不依賴顯式的通道知識，而是旨在僅基於 UE 的即時位置資訊來預測最優波束
    
* 𝑔∈𝑅2 表示二維位置向量，並由緯度 𝑔lat​ 與經度 𝑔long 所組成
    * 我們的問題就是用一個估計值 𝑓^ 來近似（逼近）𝑓⋆
    * 其中 𝑓^ 被定義為能使條件機率 𝑃(𝑓^=𝑓⋆∣𝑔) 最大的估計
 
---

## III. POSITION-AIDED BEAM PREDICTION: PROPOSED MACHINE LEARNING SOLUTIONS

* 利用位置資訊來做 beam prediction 的動機，來自於:
    * 毫米波（mmWave）窄波束的「方向性」特徵，以及
    * 毫米波對 LOS 路徑更高的依賴程度
 
* 位置與最佳波束之間的對應關係，有可能透過先前觀測到的「位置–波束配對資料」來學得
    * mapping relation / function：指一個函數關係：輸入是位置，輸出是最佳 beam
    * 表示系統先前已經看過許多資料點，每個資料點都有「位置」與「該位置的最佳 beam」的配對
 
* 我們要用一個「預測函數」$f_{\Theta}(g)$ 來近似（approximate）理想最佳 beam $f^{\star}$。這個預測函數裡有一組參數 $\Theta$，$\Theta$ 就是模型的參數集合。

#### 逐點詳細解釋

1. **$f^{\star}$ 是什麼？**
   - 意思是：在給定通道向量 $h$ 的情況下，從碼本 $\mathcal{F}$ 的所有候選 beamforming 向量 $f$ 中，選出能讓等效增益 $\left|h^{T}f\right|^{2}$ 最大的那一個。
   - 它是「理想/真值」的最佳 beam（ground truth optimal beam）。

2. **為什麼要近似（approximate）$f^{\star}$？**
   - 因為 $f^{\star}$ 的定義需要通道資訊 $h$；但作者指出 $h$ 不容易取得（hard to acquire）。
   - 所以改用可即時取得的資訊（此處是位置 $g$）來預測一個「近似最佳」的 beam。

3. **$f_{\Theta}(g)$ 代表什麼？**
   - 它是一個從「位置 $g$」映射到「預測 beam」的函數：輸入是 $g$，輸出是一個預測的 beamforming 向量（通常是碼本 $\mathcal{F}$ 中的其中一個）。
   - 你可以把它視為「模型/演算法」的總稱；不同方法（Lookup Table、KNN、Neural Network）都能寫成 $f_{\Theta}(\cdot)$ 的形式，只是內部計算方式不同。

4. **$\Theta$ 是什麼？為什麼說模型是參數化（parameterized）？**
   - “parameterized by a set $\Theta$” 表示：模型的行為不是固定的，而是由參數 $\Theta$ 決定。
   - 這個 $\Theta$ 是從  dataset 𝐷={(𝑔𝑘,𝑓𝑘⋆):𝑘=1,…,𝐾} 學出來的
   - 一個輸入位置 𝑔𝑘，以及該位置對應的最佳 beamforming vector 𝑓𝑘⋆


---

$f_{\Theta}(g)$ 的內部運作方式，取決於我們採用的學習策略，分別是以下三種方法:  ​
1. 查表法（lookup table, LT）
2. K 近鄰法（K-nearest neighbors, KNN）
3. fully connected neural network, NN

* 查表法與 KNN 方法之所以是吸引人的選擇，是因為它們簡單且直觀；同時它們也「隱含地」整合了空間相關性的知識。
* 也就是說，它們假設：具有相似 LOS 條件的位置應該會有相似的最佳波束
* 而神經網路具有較高的複雜度，但它們是強大的推論系統，因此可能有助於從位置資料中學習

這三種演算法都會估計一個機率分佈 𝑃∈{p1​,…,pM​}  
* P 是對所有候選 beam 的機率集合，因為碼本裡有 𝑀 個候選 beamforming 向量 𝑓1,…,𝑓𝑀，所以作者定義一個長度為 𝑀 的機率集合
* 𝑝𝑚 表示：「第 𝑚 個候選波束 𝑓𝑚 是真正最佳波束 𝑓⋆」的機率 (模型會對每個候選 beam 給出一個信心程度（機率）)
* 接著選擇機率最高的波束成形器 $$m_b \=\; \arg\max_{m \in \{1,\ldots,M\}} \; p_m .$$

---

## Method A. Lookup Table / Position Database

* 這個方法的動機是一個 beam 至少會覆蓋一塊連續的物理區域，所以可以把整個位置空間切成小格子（cells），每個格子指定一個 beam
    * 把連續位置空間 → 變成有限個 cell；推論時不直接算 beam，而是先找到位置落在哪個 cell，再查該 cell 對應的 beam

### 範例說明: 30m × 40m 室內空間，只有 10 個量測點，套用 Lookup Table（LT）


#### 場景設定

- 室內空間：寬 30（x 軸 0~30）、長 40（y 軸 0~40）
- 碼本（codebook）：
  \[
  \mathcal{F}=\{f_1,f_2,f_3,f_4\}
  \]
- 你有 10 筆量測資料（位置 + 該位置的 ground-truth 最佳 beam）

#### 選格子數 \(N_{lt}\)

為了示範清楚，取
\[
N_{lt}=16 \;\Rightarrow\; \sqrt{N_{lt}}=4
\]
也就是把空間切成 \(4\times 4\) 的格子（cells）。

因此每個 cell 的實體大小大約是：
- x 方向：\(30/4=7.5\) m
- y 方向：\(40/4=10\) m

---

#### Step 1：把室內座標 normalize 到 \([0,1]\)

因為 \(x\in[0,30]\)、\(y\in[0,40]\)，做 min-max 正規化可寫成：
\[
\tilde x=\frac{x}{30},\qquad \tilde y=\frac{y}{40}.
\]

---

#### Step 2：用 \(Q(g)\) 把位置丟到 \(4\times 4\) 格子

依照論文式 (4) 把區間切成 bins：

- x bins（row）：
  - row 0：\([0,7.5)\)
  - row 1：\([7.5,15)\)
  - row 2：\([15,22.5)\)
  - row 3：\([22.5,30)\)

- y bins（col）：
  - col 0：\([0,10)\)
  - col 1：\([10,20)\)
  - col 2：\([20,30)\)
  - col 3：\([30,40)\)

---

#### Step 3：10 筆量測資料（位置 → 最佳 beam）

假設你量測到下面 10 個點（\((x,y)\) 單位 m）：

| k  | 位置 \((x,y)\) | 標籤（最佳 beam） |
|---:|:--------------:|:-----------------:|
| 1  | (2, 5)         | \(f_1\) |
| 2  | (6, 8)         | \(f_1\) |
| 3  | (10, 6)        | \(f_2\) |
| 4  | (14, 9)        | \(f_2\) |
| 5  | (12, 7)        | \(f_1\) |
| 6  | (18, 12)       | \(f_3\) |
| 7  | (21, 15)       | \(f_3\) |
| 8  | (19, 14)       | \(f_2\) |
| 9  | (8, 28)        | \(f_2\) |
| 10 | (26, 34)       | \(f_4\) |

---

#### 4) Step 4：把 10 筆資料分配到 cells（算 \(Q(g_k)\)）

把每個點映射到 cell（row, col）：

- (2,5)：
  \[
  q_{\text{row}}=\left\lfloor \frac{2}{30}\cdot 4\right\rfloor=0,\quad
  q_{\text{col}}=\left\lfloor \frac{5}{40}\cdot 4\right\rfloor=0
  \Rightarrow (0,0)
  \]
- (6,8) → (0,0)  
- (10,6) → (1,0)  
- (14,9) → (1,0)  
- (12,7) → (1,0)  
- (18,12) → (2,1)  
- (21,15) → (2,1)  
- (19,14) → (2,1)  
- (8,28) → (1,2)  
- (26,34) → (3,3)

---

#### 5) Step 5：每個 cell 內統計 beam 出現次數（mode），得到 top-1/top-2

### cell (0,0)
落在 (0,0) 的點：k=1,2 → \(\{f_1,f_1\}\)

- 次數：\(f_1:2\)（其他 0）
- top-1：\(f_1\)（機率 \(p_1=1\)）

### 其他 cell（例如 (0,2)、(2,3)…）
沒有任何訓練資料 → random predict（論文中的設定）

---

## Step 6：推論（給新位置，查表輸出）

##### 推論例 1：新點 \(g=(11,8)\)  → row,col =1,0  
因此查表得到：
- top-1：\(f_2\)
- top-2：\(f_1\)

##### 推論例 2：新點 \(g=(5,25)\)→ row,col=0,2
但此 cell 沒有任何訓練資料，所以 → random predict

---

## Method B. K-Nearest Neighbors

在已知某些訓練資料(g_k, f_k) 的前提下，假設我們現在要預測位置 g 的 beam  

1. 首先挑選 Nknn 個 nearest neighbors
2. 被選到的這些鄰居樣本，會回報它們的最佳波束
3. 把這些回報鄰居標籤中的 mode (出現最多次的 beam）當作我們的最佳波束估計

---

## Method C. 

- 我們使用標準的全連接神經網路架構。架構和主要超參數如表 I 所示。
- 我們發現該架構是拐點，之後進一步增加模型複雜度將不再帶來任何效益。
- 值得注意的是，一個包含 128 個節點的隱藏層平均只會使準確率降低 2.5%。
- 因此，我們確信該網路規模不會限制我們的結果。
- 對於神經網路訓練，在訓練 60 個 epoch 後，我們觀察網路在哪個 epoch 在驗證集上獲得了最高的準確率，並使用該模型評估其在測試集上的準確率。
- 我們使用交叉熵損失函數，並使用 Adam 優化器調整神經網路權重。
- Adam 的初始學習率為 0.01，我們使用多步驟學習率調度器，在第 20 和 40 個 epoch 時將所有學習參數乘以 0.2。最後，作為神經網路的額外預處理步驟，我們將輸入資料量化為 200 個區間，即解析度為 0.005，因為歸一化後的輸入值介於 0 和 1 之間。

---

## IV. MEASUREMENTS AND DATASET DESCRIPTION
* 本研究使用的資料來自 DeepSense [9] 的一部分。
* DeepSense 是由 Wireless Intelligence Lab 建立的一個大型、多模態（multi-modal）的真實世界資料集。此資料集在每一筆資料點中，整合了同時存在（co-existing）的 GPS 位置、相機影像、雷達、光達（lidar），以及 beam training 的接收功率/CSI 量測。
* 就本研究而言，我們使用 DeepSense 中 Scenario 1 到 9 的「校正後（calibrated）」GPS 位置與功率資料。
* 本節將說明資料集如何取得，以及我們進行了哪些前處理步驟。

---

### A. Data acquisition (資料取得方式)

* DeepSense 包含在 ASU 校園周邊不同地點、白天與夜晚所取得的資料。
    * UE 安裝在一台移動中的車內 (配有 GPS 與一個在 60 GHz 運作的 mmWave 全向（omni-directional）發射器)
    * BS 是固定不動的，並配有 mmWave 接收器以及一個具有 64 個天線單元的均勻正方形陣列
    * 該車會以不同方向從 BS 前方經過
 
* BS 大約以每秒 10 次的頻率 sweeps the codebook 𝐹，並對碼本中的每個 beam 取樣量測其接收功率
* 在圖 1 中，我們針對部分 scenario 展示了俯視的 GPS 圖，以及位置散點圖
    * 在散點圖中，顏色用來表示該位置的最佳 beam。這個最佳 beam 是從 64 個可能的 beam 中選出，而這 64 個 beam 會掃描不同的方位角

<img width="1015" height="250" alt="image" src="https://github.com/user-attachments/assets/83fdef7f-9d54-46a5-96d8-0c8609b10de4" />

---

### B. Data normalization（資料正規化）

* 我們針對 UE 的絕對座標之緯度/經度，使用 **min-max 正規化**。我們比較過幾種替代方案後選擇此方法，
    * 例如：使用相對於 BS 的相對座標、使用 divide-by-max 類型的正規化（其特性是不會扭曲資料尺度）
    * 以及使用極座標（polar coordinates），也就是距離與角度。
    * 然而，簡單的 min-max 正規化總是能提供最佳結果。
 
---

### C. Data split
為了訓練模型，我們對每個 scenario 的資料做如下切分：  
* 60% 作為訓練集（training）
* 20% 作為驗證集（validation）
* 20% 作為測試集（testing）。
* 驗證集與測試集採用相同的比例是一種常見做法。我們也測試過數種相近的資料切分方式，而對效能的影響很小。

---

## V. REAL-WORLD EXPERIMENTAL RESULTS

* 在本節中，我們使用前述的真實世界資料集來測試不同演算法的效能，並且根據資料本身的特性，說明為什麼「合成資料」與「真實世界」的結果會有差異。
* 我們也會進一步提出適合真實世界運作的、有意義的效能指標，例如:
    * 評估碼本大小對預測準確率的影響
    * 針對不同的中斷機率計算可節省的訓練開銷
 
---

### A. Algorithmic comparison

### 首先，我們比較三種採用的機器學習方法的效能，並回答：使用神經網路是否過度或是合理

* 圖 2(a) 中，我們呈現三種演算法，看到 NN 在各種情境中都持續地優於 KNN 與 LT
* 圖 2(b) 顯示了每個輸入對應輸出（顏色用來表示 beam 的索引）
    * 我們取用在 Scenario 6 上訓練好的 ML 模型，並用它們去預測 10,000 (100x100) 個「均勻間隔」且覆蓋整個輸入空間的樣本點的 beam

<img width="1041" height="531" alt="image" src="https://github.com/user-attachments/assets/696e2290-32e9-4fe1-9c06-d83dd29e0dda" />


### KNN 圖片分析
* KNN 的顏色變化大致呈現「由左到右逐漸從藍→綠→黃→紅」的趨勢，但同時也有一些
    * 顏色變化不是完全平滑，而像是由幾塊區域拼起來；有些地方存在明顯的分界/轉折
    * 是因為KNN 對每個輸入位置 𝑔，只看最近的 𝑁𝑘𝑛𝑛 個訓練點，再用這些點的 beam 標籤做眾數投票得到 top-1 beam
 
### Lookup Table 圖片分析
* 大範圍呈現「像彩色雜訊一樣」的密集點狀變化，同時也有少數區域看起來是一整片比較一致的色塊
   * LT 先把輸入空間切成很多 cell（2D grid）每個 cell 用「該 cell 內訓練樣本的 beam 眾數」當 top-1（次多當 top-2…）
   * 若 cell 沒有訓練資料就是把該 cell 的分佈設成均勻 𝑝𝑚=1/𝑀
 
### NN 圖片分析
* 顏色從左到右幾乎是平滑連續地變化，很少出現像 LT 那樣的隨機散點，也不像 KNN 那樣呈現很多由邊界切割的塊狀
    * NN 是一個高度非線性的模型，權重是用所有訓練樣本（或 batch）持續調整而來
    * 因此它在整個輸入空間上的輸出呈現更穩定、更有一致性的「泛化形狀」
 
---

