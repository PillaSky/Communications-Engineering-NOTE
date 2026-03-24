# SBFD (Sub-Band Full Duplex)

TDD模式下，把頻帶分成三個或兩個子頻帶，有點像FDD那樣，用意是要在任何時間都盡量要有UL跟DL頻寬可以傳，降低延遲，而且考量DL流量遠大於UL。共有三種模式:
```
1. DU
2. UD
3. DUD
```
我有把限制成四種配置推進3GPP標準裡:
分別是:
```
1. UD / DU
2. DU / DUD
3. DUD / UD
4. DUD / DUD
```
營運商在配置時要小心相鄰頻帶營運商的相鄰頻帶，Guard band不夠大或ACLR不夠高，會影響到別人UPLINK

# FR1 FR2 FR3

**FR1（Sub-7 GHz）**:

```頻率範圍：410 MHz ～ 7.125 GHz```

特性：
覆蓋好
穿透強
傳統 cellular 主力

**FR2 (毫米波)**:

- FR2-1
```
24.25 GHz ～ 52.6 GHz
```
- FR2-2
```
52.6 GHz ～ 71 GHz
```
合起來：
```
FR2 = 24.25 GHz ～ 71 GHz
```

**FR3（研究/討論中)**:

常見定義：
```7 GHz ～ 24.25 GHz```

👉 也有人講：

7～15 GHz（保守）
7～20 GHz（常見）
7～24 GHz（最完整）

# SCS (SubCarrier Spacing)
SCS指的是OFDM子載波寬度
相對的SCS越大，Symbol duration越小

$T_{symbol}​\propto \frac{1}{\Delta f}$

# Numerology
Numerology可簡單看作是SCS (subcarrier spacing)
但實際上Numerology 不只是 SCS 本身，而是：

```一組由 SCS μ 決定的 OFDM waveform 設定```

包含：
```
1. SCS（Δf）
2. Symbol duration（符號時間）
3. Slot length（slot 長度）
4. CP 長度（cyclic prefix）
5. TTI granularity（時間解析度)
```
5G時代每個頻帶用多個Numerology，截至Release 17，總共有 7 種 Numerology，分別對應：15、30、60、120、240、480、960 kHz 的子載波間隔。，6G傾向每個頻帶單一Numerology

baseline for 6G:

FR1 FDD
：15 kHz ,TDD：30 kHz

~7 GHz
：30 kHz

# PAPR
PAPR是功率的峰值相對平均的比值

$\mathrm{PAPR} = \frac{\max\limits_{t} |x(t)|^2}{\mathbb{E}\left[|x(t)|^2\right]}$


PAPR 高代表PA（Power Amplifier）容易 nonlinear
會造成：
```
spectral regrowth（ACLR 變差）
EVM 變差
SNR degradation (Net Gain掉下來)
```
OFDM有天生PAPR高的缺點，因為多個子載波可能同時波峰==>功率爆炸
解決方法是:
```
1. DFT-s-OFDM (又稱SC-FDMA（Single Carrier FDMA）) (QAM symbols → DFT → mapping → IFFT (OFDM)) 能量被「攤平，波形變得比較像 單載波
2. Transparent and non-transparent PAPR reduction (e.g., Frequency Domain Spectrum Shaping (FDSS), Tone Reservation (TR), CFR-SE, Selected Mapping (SLM))
```
transparent : RX知道TX怎麼傳

non-transparent : RX不知道TX怎麼傳，直接解就對了

6G多數公司傾向將用transparent與non-transparent PAPR reduction的技術方法，因為DL DFT-s-OFDM or UL multi-rank DFT-s-OFDM會額外顯著增加實作複雜度跟成本

# Memory effect 記憶效應
記憶效應對PA來說很重要：

尤其在很大的通道頻寬情況下，PA 的記憶效應與功率頻譜密度（PSD）不平衡現象會顯著增加，這些考量必須納入 PA 模型中，才能對 ACLR 與 SEM (頻率發射遮罩)進行準確評估
(envelop上下起伏變化很快，envelop變化速度 > PA 反應速度，出現 memory effect，失真變嚴重)

簡單模型(無記憶（memory-less）的 PA 模型)不足以反映實際行為，因為他無法描述 ACLR 的非對稱特性。特別是在接近飽和操作點時，需使用高階模型（例如 GMP）或量測式模型
(前面偏壓之類的特性還沒恢復，但因為envelop變化太快，使得本來不會跑到非線性區的跑到非線性區，所以ACLR實際上在真實PA中很可能不是對稱，無記憶的 PA 模型無法反映這個情況)

# PC2 and PC3
Power Class (PC)=> 傳輸功率等級

PC2: 26 dBm 

PC3: 23 dBm 





