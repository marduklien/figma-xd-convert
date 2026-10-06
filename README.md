# figma-xd-convert

讓 Claude 在 Figma 與 Adobe XD 之間雙向轉檔的技能。轉出來的是可以繼續編輯的圖層，不是一張平面圖：文字仍是文字、形狀仍是形狀，圖層階層與名稱都保留。

| 方向 | 輸入 | 輸出 |
| --- | --- | --- |
| Figma 轉 Adobe XD | Figma 檔案連結（畫框、區段或整頁） | 一個 `.xd` 檔，每個畫框是一個 XD 畫板 |
| Adobe XD 轉 Figma | 一個 `.xd` 檔，加上要放進去的 Figma 檔案 | Figma 頁面上的一個區段「XD 匯入：檔名」，每個 XD 畫板是一個畫框 |

## 為什麼需要這個技能

Figma 沒有匯出 `.xd` 的功能，Adobe XD 也不再積極更新，兩邊都沒有官方的轉檔管道。常見的替代做法是匯出 SVG 再匯入，但文字常被轉成外框、圖層結構被打散，對方拿到後很難接著改。

這個技能直接讀寫 `.xd` 的檔案結構，並透過 Figma 連接器讀取與建立圖層，所以轉檔後可以繼續編輯。

## 會保留與不會保留的內容

| 項目 | Figma 轉 XD | XD 轉 Figma |
| --- | --- | --- |
| 圖層階層與名稱 | 保留 | 保留 |
| 文字（可編輯、同段落多種樣式） | 保留 | 保留 |
| 矩形、圓形、圓角 | 保留為可調整的形狀 | 保留為可調整的形狀 |
| 向量路徑 | 保留為路徑 | 保留為路徑 |
| 純色、線性漸層、圖片填色 | 保留 | 保留 |
| 放射狀漸層 | 改用灰色並提出警告 | 近似呈現 |
| 陰影、圖層模糊、背景模糊 | 保留（內陰影不支援） | 保留 |
| 遮罩與裁切 | 轉成 XD 的遮罩群組 | 矩形遮罩轉成有裁切的畫框，其他形狀轉成遮罩圖層 |
| 元件與元件實體 | 變成一般群組 | 變成一般群組或畫框 |
| 自動排版、重複格線 | 不保留，只保留當下的位置與尺寸 | 不保留，重複格線變成群組 |
| 原型連結、變數、樣式庫 | 不轉換 | 不轉換 |

## 需求

- 可以使用技能的 Claude：網頁版、桌面版或 Claude Code 都可以。在網頁版或桌面版使用時，要先啟用程式碼執行功能。
- 已連接 Figma 連接器，而且連接的帳號對該 Figma 檔案有**編輯權限**。只有檢視權限時，請先把檔案「複製到你的草稿」，再提供複本的連結。
- 可執行 Python 3 的環境。只用到標準函式庫，不必另外安裝套件。
- Figma 轉 XD：對方的電腦要安裝設計稿使用的字型，否則 XD 會提示缺字型。
- XD 轉 Figma：建立圖層的動作在 Figma 的伺服器上執行，只能使用 Figma 提供的字型。只安裝在自己電腦上的字型會被換成字重相近的備用字型（預設為 Noto Sans TC），並在完成時列出替換清單。

## 安裝

技能本體是 `figma-xd-convert/SKILL.md` 這一個檔案，兩個方向的程式都寫在裡面。安裝方式有兩種，依使用的環境選擇：

| 使用環境 | 方式一：上傳壓縮檔 | 方式二：一行指令 |
| --- | --- | --- |
| Claude 網頁版 | 可以 | 不適用 |
| Claude 桌面版：聊天與一般工作的分頁 | 可以 | 不適用，這裡不會讀取 `~/.claude/skills/` |
| Claude 桌面版：「Code」分頁（在自己電腦上執行的工作階段） | 可以 | 可以 |
| 終端機裡的 Claude Code | 可以，需用同一個帳號登入 | 可以 |

上傳的技能會存在帳號裡並自動同步，同一個帳號登入的網頁版、桌面版與 Claude Code 都能使用。想要到處都能用，選方式一；只在 Claude Code 使用，方式二比較快。

### 方式一：上傳壓縮檔（所有環境都適用）

1. 下載壓好的檔案：[figma-xd-convert.zip](https://github.com/marduklien/figma-xd-convert/raw/main/figma-xd-convert.zip)。不必解壓縮。
2. 在 Claude 開啟「自訂」→「技能」，新增技能並選擇上傳，把這個 ZIP 檔傳上去。
3. 確認清單裡 figma-xd-convert 的開關是開啟的。

想自己打包的話，把儲存庫裡的 `figma-xd-convert` 資料夾壓縮成 ZIP 檔即可。壓縮檔的最上層必須是 `figma-xd-convert` 資料夾，資料夾名稱要與技能名稱相同。

介面可能會調整，最新步驟請看官方說明：[在 Claude 中使用技能](https://support.claude.com/en/articles/12512180-use-skills-in-claude)。

### 方式二：一行指令（Claude Code）

```bash
npx skills add marduklien/figma-xd-convert -g -a claude-code -y
```

執行後技能會裝到 `~/.claude/skills/figma-xd-convert/`，所有專案都能使用。需要先安裝 Node.js；macOS、Linux、Windows 都適用。

| 參數 | 作用 |
| --- | --- |
| `-g` | 裝到個人層級，所有專案共用。拿掉就只裝到目前所在的專案（`./.claude/skills/`） |
| `-a claude-code` | 指定安裝給 Claude Code。這個工具也支援其他能使用技能的程式，換成對應的名稱即可 |
| `-y` | 略過確認，直接安裝 |

三個參數都不加時（`npx skills add marduklien/figma-xd-convert`），會改用互動方式逐項詢問。

這行指令使用的是開放原始碼的 [skills](https://github.com/vercel-labs/skills) 工具，不是 Claude 內建的功能。之後要更新到新版，執行 `npx skills update -g`。

#### 沒有 Node.js 時

技能只有一個檔案，直接下載到技能目錄也可以。

macOS 或 Linux：

```bash
mkdir -p ~/.claude/skills/figma-xd-convert && curl -fsSL https://raw.githubusercontent.com/marduklien/figma-xd-convert/main/figma-xd-convert/SKILL.md -o ~/.claude/skills/figma-xd-convert/SKILL.md
```

Windows（PowerShell）：

```powershell
New-Item -ItemType Directory -Force "$HOME\.claude\skills\figma-xd-convert" | Out-Null; Invoke-WebRequest "https://raw.githubusercontent.com/marduklien/figma-xd-convert/main/figma-xd-convert/SKILL.md" -OutFile "$HOME\.claude\skills\figma-xd-convert\SKILL.md"
```

## 使用方式

安裝後直接用平常的說法提出需求即可，Claude 會依方向選擇對應的流程。

Figma 轉 Adobe XD：

> 把這個 Figma 檔案的三個畫框轉成 XD 檔：
> https://www.figma.com/design/檔案代碼/檔名?node-id=1-701

Adobe XD 轉 Figma：

> 把附件的 XD 檔轉進這個 Figma 檔案的「首頁」那一頁：
> https://www.figma.com/design/檔案代碼/檔名

## 運作方式

**Figma 轉 XD**

1. 在 Figma 檔案裡執行一段腳本，把圖層樹與用到的圖片讀出來。
2. 用 `figma2xd.py` 把圖層樹轉成 XD 的檔案結構，打包成 `.xd` 檔。

**XD 轉 Figma**

1. 用 `xd2figma.py` 讀取 `.xd` 檔，產生一份壓縮過的建構計畫。
2. 把建構計畫與圖片送進 Figma，再執行建構腳本建立圖層。圖片多或檔案大時，會改產生一個「匯入用」的 SVG 檔，請你把它拖進 Figma 頁面，資料就會跟著進去。

兩個方向都需要在 Figma 檔案裡**暫時建立一個存放資料的圖層**（名稱分別是 `__xd_export_tmp__` 與 `__xd_chunks__`），流程結束時會自動刪除。如果中途失敗而留下這些圖層，可以直接手動刪除，不影響原本的設計。

## 已知限制

Figma 轉 XD：

- 多行文字的折行是估算的，XD 開啟後會自行重排，行數或位置可能有些微差異。
- 帶描邊的向量線條會轉成外框化的填色路徑，之後無法再調整線條粗細。
- 圖片只支援 PNG 與 JPEG，並且一律以等比例填滿並裁切的方式呈現。在 Figma 中設為「符合」或自訂裁切位置的圖片，顯示範圍可能不同。
- 沒有填色的畫框上的背景模糊會被略過。

XD 轉 Figma：

- 文字的垂直位置是依字型量測值推算的，可能有一兩個像素的差異；多行文字由 Figma 重新折行。
- 複合形狀（布林運算）會變成單一路徑。
- 圖片只支援 PNG 與 JPEG。

## 測試狀況

- Figma 轉 XD：產生的檔案已用 Adobe XD 61 實際開啟，物件與圖片顯示正常。
- XD 轉 Figma：已在 Figma 中完整重建一份約九百個圖層的設計稿，沒有錯誤。
- 文字位置已校正的字型有 Noto Sans TC、Inter、Verdana、Tahoma，其他字型使用通用的預設值，垂直位置可能有固定偏移。遇到這種情況，可以在程式裡為該字型補上量測值，`SKILL.md` 的「疑難排解」有說明。
- 尚未用「在 Adobe XD 裡從頭製作」的各種檔案做大量測試。遇到轉不出來或顯示不對的圖層，歡迎開議題回報，並盡量附上能重現問題的檔案。

## 檔案結構

```
figma-xd-convert/
├── README.md
├── LICENSE
├── figma-xd-convert.zip      壓好的技能檔，上傳到 Claude 用
└── figma-xd-convert/
    └── SKILL.md              技能本體：流程說明、轉檔程式與 Figma 腳本
```

## 授權

採用 [MIT 授權條款](LICENSE)，可以自由使用、修改與散布，只需保留原本的著作權聲明與授權條款。
