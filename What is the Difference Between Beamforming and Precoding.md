# What is the Difference Between Beamforming and Precoding?
本篇文章參考該[國外論壇](https://ma-mimo.ellintech.se/2017/10/03/what-is-the-difference-between-beamforming-and-precoding/)

這篇文章旨在探討「波束成形」(Beamforming) 與「預編碼」(Precoding) 之間的區別，這兩個術語在無線通訊領域經常被交替使用，但其具體涵義卻有多種解釋。以下整理了幾種常見的觀點   



### 觀點一： Beamforming 與 Precoding 是同義詞
最廣義的觀點認為，這兩個術語指的是同一概念：利用天線陣列來發射或接收具有特定空間指向性的訊號。這種解釋將所有利用多天線技術來集中能量、增強訊號方向性的方法，都歸類為波束成形或預編碼。



### 觀點二: Beamforming 可以分為兩類
* Analog Beamforming: 相同的訊號被送到每個天線，然後使用 **analog phase-shifters**來控制陣列發射的訊號
* Digital Beamforming: 在數位層面為每個天線獨立設計不同的訊號。這種方式提供了更大的靈活性，因為可以獨立控制每個天線的功率和相位，甚至針對頻譜的不同部分（例如子載波）進行調整。預編碼在頻寬較大時也大有裨益，因為在固定相位下，訊號在頻段的不同部分會獲得不同的方向性。

根據這個觀點:  
* Precoding = digital beamforming: 由於數位波束成形可以實現 Spatial Multiplexing ，即同時「向多個使用者」傳輸「多個獨立的資料流」，因此預編碼特別適用於此類應用。
<img width="482" height="251" alt="image" src="https://github.com/user-attachments/assets/ca635864-eab7-4881-9398-8c9a42970785" />

1. 左圖可理解成: 所有天線一起發送「同一個訊號」，差別在於它們被個別乘上某個 weight / phase
2. 右圖可理解成: 所有天線個別發送「不同的訊號」



### 觀點三：波束成形用於單一資料流，預編碼用於多資料流
這個解釋將兩者區分得更為明確：  
* Beamforming： 指發射單一的訊號，通常包含「一個主瓣和一些旁瓣」，主要用於提升「單一使用者單一資料流」的訊號品質
* Precoding： 指將多個波束疊加，同時「傳輸多個獨立的資料流」，以實現 spatial multiplexing ，提升系統的整體容量i9o0-kuj]'



### 觀點四： Beamforming 限制於 LoS 通訊； Precoding 則更廣泛
* Beamforming： 專指在 LoS 環境下，將能量集中發射到特定角度方向；因為當向 NLoS 使用者傳輸時，發射訊號可能沒有清晰的角度方向性
* Precoding： 涵蓋所有類型天線陣列傳輸，不僅限於視距環境。在非視距 (NLoS) 環境下，訊號透過多路徑傳播到達接收端，此時預編碼旨在讓到達接收端的多路徑分量能建設性地疊加，以最大化接收到的功率。



### 觀點五：Precoding 包含兩部分：選擇方向性和選擇發射功率
這個解釋將預編碼視為一個更全面的過程，它包含兩個關鍵步驟：  
* 選擇方向性（波束成形）：決定訊號發射的方向。
* 選擇發射功率（功率分配）：決定分配給各個方向的能量。

***

## 結論
兩者差異的幾種主要觀點：
* 廣義同義詞：有些觀點認為兩者是同一個概念，都指利用天線陣列實現訊號指向性。
* 類比 vs. 數位：波束成形可以是類比或數位；而預編碼則等同於功能更強大、能實現空間復用的「數位波束成形」。
* 單流 vs. 多流：波束成形通常指發射單一、高指向性的訊號；預編碼則指疊加多個波束，實現多個資料流的空間復用。
* 視距 vs. 廣泛應用：波束成形可能被限制在視距通訊的應用中；預編碼則是一個更廣泛的概念，能適用於多路徑複雜的非視距通訊環境。
* 部分 vs. 整體：預編碼被視為一個包含「波束成形」（選擇方向性）和「功率分配」的完整過程。

***

## 補充說明
### 什麼是 Hybrid Beamforming?
在大規模 MIMO 系統（例如 mmWave 或 6G 中的 ISAC 應用）中，傳統的 fully digital beamforming 會「為每根天線配置一條數位 RF Chain」（包含 DAC/ADC、混頻器、放大器等）。但這樣做：成本高、耗能高、硬體複雜度極高  

因此，為了節省成本與功耗，Hybrid Beamforming 應運而生，它結合了：  
* 數位 baseband beamformer（用較少的 RF chain 在 baseband 執行 beamforming）
* 類比 RF beamformer（使用相位器控制天線信號相位）

### Fully-Connected Hybrid Beamforming 架構的定義
在 fully-connected 架構中：  
* 每一條 RF chain 都會有一組 analog phase shifters
* 每一條 RF chain 都會與所有天線相連。
    * 每條 RF chain 負責控制所有天線的訊號相位。
 
### 數學描述總 analog phase shifter
若系統有：𝑈 條 RF 鏈 、 𝑁𝑡 根 傳送天線，則需要的 analog phase shifter 數量為: 𝑈×𝑁𝑡  

* 實際舉例說明: 假設系統有 8 根傳送天線 & 2 條 RF Chain
* 總相位器數量：2 (RF chains)×8 (antennas)=16 個 analog phase shifters
