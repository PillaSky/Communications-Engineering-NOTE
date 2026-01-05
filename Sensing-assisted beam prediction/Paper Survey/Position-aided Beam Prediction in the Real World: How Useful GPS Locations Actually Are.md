# Position-aided Beam Prediction in the Real World: How Useful GPS Locations Actually Are?

J. Morais, A. Bchboodi, H. Pezeshki and A. Alkhateeb, "Position-Aided Beam Prediction in the Real World: How Useful GPS Locations Actually are?," ICC 2023 - IEEE International Conference on Communications, Rome, Italy, 2023, pp. 1824-1829, doi: 10.1109/ICC45041.2023.10278998.

---
## 閱讀動機

情境有兩個部分:  
1. 真實世界: 在 Indoor 使用 mmWave Base station 發射三種形狀的beam
2. 虛擬世界: 創造一個虛擬的 Indoor 場景，並使用 NVIDIA Sionna，並把真實世界的那三種 beam 在虛擬世界裡刻出

接著我會把「少量量測點」當成訓練資料，讓模型學到「位置/環境 ↔ 最佳 beam（或候選 beam）」之間的對應關係，之後在沒量測過的位置做推論（beam prediction）  

## Abstract
毫米波（mmWave）通訊系統必須依賴窄波束（narrow beams），才能獲得足夠的接收信號功率。  
1. 毫米波為何需要窄波束？
摘要在點出 mmWave 的基本特性：mmWave 頻段路徑損耗大、遮蔽敏感，因此若不把能量「集中」在特定方向，接收端收到的功率可能不足，鏈路品質就會差。

2. 窄波束的意思是什麼？
這裡的窄波束通常指：基地台用多天線陣列做 beamforming，把主瓣變得很窄、指向性很強。主瓣越窄代表能量越集中，方向不對就會掉很多功率。


