## 1. Compaction 為什麼耗時？—— 磁碟實體操作的本質

你的理解完全正確。原因可以拆成兩層：

### (1) 對象是「實體磁碟資料」，不是記憶體
Compaction 要做的事是：把磁碟上所有檔案實際搬動位置，讓所有 file 緊密排列在一起，藉此把散落各處的小塊外部碎片（external fragmentation）合併成一塊完整的連續空間。

這跟記憶體中做 compaction（例如搬移 process 位置）性質上類似，但磁碟的物理特性讓成本高非常多：

| 比較項目 | Memory (RAM) | Disk |
| :--- | :--- | :--- |
| **存取速度** | 奈秒等級 (nanosecond) | 毫秒等級 (millisecond)，慢了 10 萬倍以上 |
| **移動方式** | 電子訊號傳輸 | 機械動作：磁頭要移動 (seek)、碟片要轉到定位 (rotational latency)，才能真正讀寫資料 |
| **搬移大量資料的代價** | 相對便宜 | 非常昂貴——搬一個大檔案等於要整個「讀出來 → 寫到新位置」，牽涉大量 I/O |

### (2) 搬移過程中系統通常要暫停服務
要安全地搬動檔案位置，往往代表在 compaction 執行期間，這個磁碟（或至少受影響的區域）無法正常提供讀寫服務，這對一般使用中的系統來說代價很高，因此 compaction 通常只能排在系統離峰時段做（如深夜維護），不能隨時做。

---

### 一句話總結
Compaction 之所以耗時，本質原因就是它要在「機械式、高延遲」的磁碟上，實際搬動大量資料的位置，而不是像記憶體那樣單純的電子層級操作——這也是為什麼 contiguous allocation 雖然讀取效能最好，但幾乎不被現代檔案系統直接採用的主因之一。

---

## 2. Extent-Based Allocation：解決「檔案不易擴充」的具體做法

### 先回顧問題本身
Contiguous allocation 要求一個檔案從頭到尾，必須佔用一整段連續的磁碟空間。

問題來了：如果檔案建立時只配置了「剛好夠用」的大小，之後檔案變大（例如你不斷往一個 log 檔案寫入資料），檔案後面緊鄰的磁碟空間可能早就被別的檔案佔走了，根本無法「就地」往後長大。

### Extent 的核心概念
* **Extent（延伸區塊）** = 一段連續的磁碟空間（一個 chunk）。
* **Extent-based allocation** 的做法是：一個檔案不再被要求「整個檔案」必須是單一一段連續空間，而是允許由「好幾個 extent」組成，每個 extent 各自內部連續，但 extent 與 extent 之間不需要彼此相鄰。

### 具體怎麼運作？
檔案 A 的內容 = $\text{Extent 1} + \text{Extent 2} + \text{Extent 3} + \dots$

* **Extent 1**: 從磁碟位址 $100$ 開始，長度 $50$ blocks（block $100 \sim 149$，內部連續）
* **Extent 2**: 從磁碟位址 $300$ 開始，長度 $30$ blocks（block $300 \sim 329$，內部連續）
* **Extent 3**: 從磁碟位址 $500$ 開始，長度 $20$ blocks（block $500 \sim 519$，內部連續）

每個 extent 用 $(\text{起始位址}, \text{長度})$ 這一組資訊表示——這其實跟你剛剛問的 free space management 的 Counting 方法概念上是同一套邏輯：善用「連續性」，用一組數字代表一大段空間，而不是逐個 block 記錄。

### 檔案變大時怎麼處理？
* **情境 A**：原本的 extent 後面剛好還有空間 $\rightarrow$ 直接往後擴充這個 extent 的長度即可。
* **情境 B**：原本的 extent 後面沒有空間了 $\rightarrow$ 系統另外找一塊新的連續空間，配置一個新的 extent，並把它「串接」到這個檔案的 extent 清單後面。

### 用一張表整理 Extent 帶來的效果

| 特性 | 純 Contiguous Allocation | Extent-Based Allocation |
| :--- | :--- | :--- |
| **整個檔案是否要求連續** | ✅ 是，整份檔案必須連在一起 | ❌ 否，只要求每個 extent 內部連續 |
| **檔案能否輕鬆擴充** | 困難，很容易後面沒空間可長 | 容易，長不下去就再開一個新 extent |
| **外部碎片問題** | 嚴重 | 大幅緩解（因為不強求整檔連續，小空隙也能被某個 extent 利用） |
| **讀取效能** | 最佳（完全連續，seek 開銷最小） | 接近 contiguous（因為大部分資料還是分段連續，只在 extent 交界處才需要額外 seek，遠比「打散成一堆零碎 block」的效能好） |

---

### 一句話總結
Extent-based allocation 的做法，是把「整個檔案必須連續」的硬性要求，放寬成「檔案由數個各自連續的區段 (extent) 組成」。當檔案需要擴充、原地長不下去時，系統就再配置一個新的連續 extent 接到檔案後面，而不需要事先預留大片空間或整份搬遷——這樣既保留了 contiguous allocation「大範圍連續讀取」的效能優勢，又解決了檔案難以擴充、容易產生外部碎片的缺點，可以說是 contiguous 和 linked allocation 之間的一種折衷設計。
"""

file_path = "contiguous_allocation_analysis.md"
with open(file_path, "w", encoding="utf-8") as f:
    f.write(markdown_content)
print(f"File successfully created: {file_path}")
