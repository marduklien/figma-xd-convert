---
name: "figma-xd-convert"
description: "在 Figma 與 Adobe XD 之間雙向轉檔：把 Figma 畫框轉成可編輯的 .xd 檔，或把 .xd 檔轉成 Figma 中可編輯的圖層。使用者要求 Figma 轉 XD、XD 轉 Figma、匯出或匯入 XD 檔時使用。"
---

# Figma 與 Adobe XD 雙向轉檔

這個技能包含兩個獨立的流程。依使用者要的方向擇一執行，另一個方向的內容可以略過：

- 使用者有 Figma 檔案、要拿到 `.xd` 檔：依「甲、Figma 轉 Adobe XD」。
- 使用者有 `.xd` 檔、要在 Figma 裡得到圖層：依「乙、Adobe XD 轉 Figma」。

兩個流程各有自己的程式與腳本，只寫入並執行該方向需要的那一份，不要混用。兩者都需要 Figma 連接器，而且連接的帳號對檔案要有編輯權限。方向不明確時先問使用者。

## 甲、Figma 轉 Adobe XD

把 Figma 檔案中的畫框轉成 `.xd` 檔：每個畫框變成一個 XD 畫板，文字是真正的文字圖層，圖層階層與名稱都保留。產出的格式已用 Adobe XD 61 實測可開啟。

### 前置條件

- 需要 Figma 連接器的 `use_figma` 與 `get_metadata` 兩個工具。呼叫 `use_figma` 前，若有 figma-use 技能要先載入。
- 連接的 Figma 帳號必須對檔案有編輯權限。若工具回報沒有編輯權限，請使用者在 Figma 把檔案「複製到你的草稿」，再提供複本連結。
- 流程會在 Figma 檔案裡暫時建立一個名為 `__xd_export_tmp__` 的畫框存放資料，讀完必須刪除。開始前先告訴使用者這件事。

### 步驟

1. 從 Figma 連結取出 fileKey 與節點 ID（網址的 `node-id=1-701` 要寫成 `1:701`）。目標可以是一個或多個畫框，也可以是區段或頁面（會轉換其下所有最上層畫框）。不確定要轉哪些時，先用 `get_metadata` 看頁面結構，再向使用者確認。
2. 把下方「轉檔程式」原樣寫入工作目錄的 `figma2xd.py`。不要改寫或精簡內容。
3. 用 `use_figma` 執行下方「匯出腳本」，只修改第一行的 `ROOT_IDS`。確認回傳的 `namesIntact` 為 true，並記下 `holderId`。
4. 對 `holderId` 呼叫 `get_metadata`。結果很大，工具會自動存成檔案並回報路徑，記下該路徑。不要把內容讀進對話。
5. 立刻用 `use_figma` 執行下方「清理腳本」刪除暫存畫框，確認 `leftover` 為 0。即使後面的步驟失敗也要先做這一步。
6. 解碼並轉檔：
   ```bash
   python3 figma2xd.py decode <get_metadata結果檔路徑> work
   python3 figma2xd.py build --tree work/tree.json --images work --out <檔名>.xd --title <檔名>
   ```
   `decode` 會列出畫框編號。畫板預設依 Figma 中由左到右排列，要指定子集或順序時加 `--frames 0,2,1`。
7. 檢查：看 `build` 輸出的統計與警告；用 `python3 -c "import zipfile,sys; print(zipfile.ZipFile(sys.argv[1]).testzip())" <檔名>.xd` 確認壓縮包完整（應輸出 None）。
8. 把 `.xd` 檔交給使用者，並說明下方的已知限制，以及統計中出現的略過或警告項目。這裡無法開啟 XD，請使用者實際開啟確認。

### 已知限制（交付時要告知）

- 元件實體會變成一般群組，自動排版不保留，只保留當下的位置與尺寸。
- 多行文字的折行是估算的，XD 開啟後會自行重排，行數或位置可能有些微差異。
- 帶描邊的向量線條會轉成外框化的填色路徑，無法再調整線條粗細。
- 只支援純色、線性漸層、圖片三種填色；放射狀等其他漸層會改用灰色並出現警告。
- 圖片只支援 PNG 與 JPEG，而且一律以等比例填滿並裁切的方式呈現；在 Figma 中設為「符合」或自訂裁切位置的圖片，顯示範圍可能不同。
- 沒有填色的畫框上的背景模糊會被略過；原型連結、變數、樣式庫不會轉換。
- 開啟檔案的電腦需安裝設計稿使用的字型，否則 XD 會提示缺字型。

### 疑難排解

- XD 打不開：加 `--no-effects`（不含漸層、陰影、模糊）與 `--no-text`（不含文字）各產生一個對照檔，請使用者依序開啟，找出是哪一類圖層造成的。
- `decode` 回報「資料不完整」或「找不到圖層資料」：`get_metadata` 沒有列出全部暫存圖層。確認是對 `holderId` 本身呼叫，然後重跑步驟 3 到 5。
- `get_metadata` 的結果沒有被存成檔案而是直接顯示：把顯示的完整內容寫入一個文字檔，再把該檔交給 `decode`。
- 文字的垂直位置有固定偏移：在程式的 `FONT_METRICS` 為該字型補上（自動行高倍率, 基線係數）。

### 匯出腳本（use_figma）

```js
const ROOT_IDS = ['1:701'];
for (const old of figma.currentPage.children.filter(n => n.name === '__xd_export_tmp__')) old.remove();
const r2 = v => (typeof v === 'number' ? Math.round(v * 100) / 100 : v);
const r3 = v => (typeof v === 'number' ? Math.round(v * 1000) / 1000 : v);
const imgHashes = new Set();
const paints = ps => (ps === figma.mixed || !ps) ? [] : ps.filter(p => p.visible !== false).map(p => {
  const o = { t: p.type, o: r3(p.opacity === undefined ? 1 : p.opacity) };
  if (p.type === 'SOLID') o.c = [r3(p.color.r), r3(p.color.g), r3(p.color.b)];
  else if (p.type === 'IMAGE') { o.h = p.imageHash; o.sm = p.scaleMode; if (p.imageHash) imgHashes.add(p.imageHash); }
  else if (p.type.startsWith('GRADIENT')) { o.st = p.gradientStops.map(s => ({ p: r3(s.position), c: [r3(s.color.r), r3(s.color.g), r3(s.color.b), r3(s.color.a)] })); o.gt = p.gradientTransform; }
  return o;
});
function dump(n) {
  const at = n.absoluteTransform;
  const o = { n: n.name, ty: n.type, w: r2(n.width), h: r2(n.height), at: [r3(at[0][0]), r3(at[1][0]), r3(at[0][1]), r3(at[1][1]), r2(at[0][2]), r2(at[1][2])] };
  if (n.visible === false) o.hid = 1;
  if ('opacity' in n && n.opacity !== 1) o.op = r3(n.opacity);
  if ('fills' in n && n.type !== 'TEXT') { const f = paints(n.fills); if (f.length) o.f = f; }
  if ('strokes' in n) { const s = paints(n.strokes); if (s.length) { o.s = s; o.sw = n.strokeWeight === figma.mixed ? [n.strokeTopWeight, n.strokeRightWeight, n.strokeBottomWeight, n.strokeLeftWeight] : n.strokeWeight; o.sa = n.strokeAlign; if (n.dashPattern && n.dashPattern.length) o.dash = n.dashPattern; } }
  if ('cornerRadius' in n) { if (n.cornerRadius === figma.mixed) o.cr = [n.topLeftRadius, n.topRightRadius, n.bottomRightRadius, n.bottomLeftRadius]; else if (n.cornerRadius) o.cr = r3(n.cornerRadius); }
  if ('effects' in n && n.effects.length) { const fx = n.effects.filter(e => e.visible !== false).map(e => ({ t: e.type, c: e.color ? [r3(e.color.r), r3(e.color.g), r3(e.color.b), r3(e.color.a)] : undefined, x: e.offset ? e.offset.x : undefined, y: e.offset ? e.offset.y : undefined, r: e.radius, sp: e.spread })); if (fx.length) o.fx = fx; }
  if ('clipsContent' in n && n.clipsContent) o.clip = 1;
  if ('isMask' in n && n.isMask) o.mask = 1;
  if (n.type === 'TEXT') {
    o.tx = n.characters; o.ah = n.textAlignHorizontal; o.av = n.textAlignVertical; o.ar = n.textAutoResize;
    o.seg = n.getStyledTextSegments(['fontName', 'fontSize', 'fills', 'lineHeight', 'letterSpacing', 'textDecoration', 'textCase']).map(s => ({ a: s.start, b: s.end, ff: s.fontName.family, fs: s.fontName.style, sz: s.fontSize, f: paints(s.fills), lh: s.lineHeight.unit === 'AUTO' ? 'A' : [s.lineHeight.unit, r3(s.lineHeight.value)], ls: [s.letterSpacing.unit, r3(s.letterSpacing.value)], td: s.textDecoration === 'NONE' ? undefined : s.textDecoration, tc: s.textCase === 'ORIGINAL' ? undefined : s.textCase }));
  }
  if (['VECTOR', 'BOOLEAN_OPERATION', 'STAR', 'POLYGON', 'LINE'].includes(n.type)) {
    if (n.fillGeometry && n.fillGeometry.length) o.fg = n.fillGeometry.map(g => ({ d: g.data, wr: g.windingRule }));
    if (n.strokeGeometry && n.strokeGeometry.length) o.sg = n.strokeGeometry.map(g => ({ d: g.data, wr: g.windingRule }));
  }
  if ('children' in n && n.type !== 'BOOLEAN_OPERATION') o.ch = n.children.map(dump);
  return o;
}
function utf8(s) {
  const out = [];
  for (let i = 0; i < s.length; i++) {
    let c = s.charCodeAt(i);
    if (c >= 0xd800 && c < 0xdc00 && i + 1 < s.length) { const d = s.charCodeAt(i + 1); c = 0x10000 + ((c - 0xd800) << 10) + (d - 0xdc00); i++; }
    if (c < 0x80) out.push(c);
    else if (c < 0x800) out.push(0xc0 | (c >> 6), 0x80 | (c & 63));
    else if (c < 0x10000) out.push(0xe0 | (c >> 12), 0x80 | ((c >> 6) & 63), 0x80 | (c & 63));
    else out.push(0xf0 | (c >> 18), 0x80 | ((c >> 12) & 63), 0x80 | ((c >> 6) & 63), 0x80 | (c & 63));
  }
  return new Uint8Array(out);
}
const roots = [];
for (const id of ROOT_IDS) {
  const node = await figma.getNodeByIdAsync(id);
  if (!node) throw new Error('找不到節點 ' + id);
  if (!roots.length) { let p = node; while (p.type !== 'PAGE') p = p.parent; await figma.setCurrentPageAsync(p); }
  if (node.type === 'SECTION' || node.type === 'PAGE') roots.push(...node.children.filter(c => ['FRAME', 'COMPONENT', 'COMPONENT_SET', 'INSTANCE'].includes(c.type)));
  else roots.push(node);
}
const frames = roots.map(dump);
const json = JSON.stringify({ frames });
const blobs = [{ key: 'tree', b64: figma.base64Encode(utf8(json)) }];
const images = [];
for (const h of imgHashes) {
  const bytes = await figma.getImageByHash(h).getBytesAsync();
  images.push({ h, bytes: bytes.length });
  blobs.push({ key: 'img_' + h, b64: figma.base64Encode(bytes) });
}
let total = blobs.reduce((a, b) => a + b.b64.length, 0);
if (total < 200000) blobs.push({ key: 'pad', b64: 'A'.repeat(200000 - total) });
const holder = figma.createFrame();
holder.name = '__xd_export_tmp__'; holder.x = 100000; holder.y = 100000; holder.clipsContent = false; holder.fills = [];
holder.resize(5000, 5000);
const CH = 10000; let chunks = 0, intact = true;
for (const bl of blobs) {
  for (let i = 0, k = 0; i < bl.b64.length; i += CH, k++) {
    const r = figma.createRectangle(); r.resize(100, 100);
    const nm = bl.key + '|' + bl.b64.length + '|' + String(k).padStart(5, '0') + ':' + bl.b64.slice(i, i + CH);
    r.name = nm; if (r.name !== nm) intact = false;
    holder.appendChild(r); r.x = (chunks % 40) * 120; r.y = Math.floor(chunks / 40) * 120; chunks++;
  }
}
return { createdNodeIds: [holder.id], holderId: holder.id, chunks, namesIntact: intact, jsonLen: json.length, images, frames: frames.map((f, i) => [i, f.n, f.w, f.h]) };
```

暫存圖層必須直接放在大尺寸（5000×5000）的暫存畫框下、彼此不重疊；若包在小畫框裡，`get_metadata` 會把它當成圖示而省略子圖層。

### 清理腳本（use_figma）

若暫存畫框不在第一頁，先用 `await figma.setCurrentPageAsync(...)` 切到該頁再執行。

```js
const olds = figma.currentPage.children.filter(n => n.name === '__xd_export_tmp__');
const ids = olds.map(n => n.id);
for (const o of olds) o.remove();
return { removedIds: ids, mutatedNodeIds: ids, leftover: figma.currentPage.children.filter(n => n.name === '__xd_export_tmp__').length };
```

### 轉檔程式（figma2xd.py）

```python
#!/usr/bin/env python3
"""figma2xd: 把 Figma 圖層樹轉成 Adobe XD 的 .xd 檔（已用 XD 61 實測可開啟）。

用法:
  1) 解碼從 Figma 取回的資料（get_metadata 的結果檔）:
     python3 figma2xd.py decode <get_metadata結果檔> <工作資料夾>
  2) 產生 .xd:
     python3 figma2xd.py build --tree <工作資料夾>/tree.json --images <工作資料夾> --out <輸出>.xd
        [--frames 0,2,1] [--title 名稱] [--no-text] [--no-effects]
"""
import argparse, datetime, hashlib, json, os, random, re, struct, uuid, zipfile, zlib

APP_VERSION = "61.0.12.1"
UX_VERSION = "2.33"
AGC_VERSION = "1.5.0"
AGC_TYPE = "application/vnd.adobe.agc.graphicsTree+json"
RES_HREF = "/resources/graphics/graphicContent.agc"

# 字型的垂直量測值：(自動行高倍率, 基線係數 k)；基線 = 行框頂端 + 行高/2 + k*字級
FONT_METRICS = {
    "Noto Sans TC": (1.2, 0.38),
    "Inter": (1.2102, 0.3636),
    "Verdana": (1.2153, 0.3977),
    "Tahoma": (1.207, 0.3965),
}
DEFAULT_METRICS = (1.2, 0.36)

_rng = random.Random(20261006)


def new_id():
    return str(uuid.UUID(int=_rng.getrandbits(128), version=4))


def rnd(v, n=3):
    v = round(float(v), n)
    return int(v) if v == int(v) else v


# ---------- 矩陣 ----------
def m_inv(m):
    a, b, c, d, tx, ty = m
    det = a * d - b * c
    ia, ib, ic, idd = d / det, -b / det, -c / det, a / det
    return [ia, ib, ic, idd, -(ia * tx + ic * ty), -(ib * tx + idd * ty)]


def m_mul(m1, m2):
    """m1 ∘ m2：先套 m2 再套 m1"""
    a1, b1, c1, d1, x1, y1 = m1
    a2, b2, c2, d2, x2, y2 = m2
    return [a1 * a2 + c1 * b2, b1 * a2 + d1 * b2, a1 * c2 + c1 * d2, b1 * c2 + d1 * d2,
            a1 * x2 + c1 * y2 + x1, b1 * x2 + d1 * y2 + y1]


def xd_transform(m):
    return {"a": rnd(m[0], 5), "b": rnd(m[1], 5), "c": rnd(m[2], 5), "d": rnd(m[3], 5), "tx": rnd(m[4]), "ty": rnd(m[5])}


def is_identity(m):
    return all(abs(x - y) < 1e-6 for x, y in zip(m, [1, 0, 0, 1, 0, 0]))


# ---------- 顏色與填色 ----------
def xd_color(c, alpha=1.0):
    col = {"mode": "RGB", "value": {"r": round(c[0] * 255), "g": round(c[1] * 255), "b": round(c[2] * 255)}}
    if alpha < 0.999:
        col["alpha"] = rnd(alpha)
    return col


NO_STROKE = lambda: {"type": "none", "color": {"mode": "RGB", "value": {"r": 112, "g": 112, "b": 112}}, "width": 1}


def image_info(data):
    """回傳 (MIME 類型, 寬, 高)；只支援 PNG 與 JPEG"""
    if data[:8] == bytes([0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A]):
        return "image/png", int.from_bytes(data[16:20], "big"), int.from_bytes(data[20:24], "big")
    if data[:2] == bytes([0xFF, 0xD8]):
        i = 2
        while i + 9 < len(data):
            if data[i] != 0xFF:
                i += 1
                continue
            mk = data[i + 1]
            if mk in (0xC0, 0xC1, 0xC2, 0xC3, 0xC5, 0xC6, 0xC7, 0xC9, 0xCA, 0xCB, 0xCD, 0xCE, 0xCF):
                return "image/jpeg", int.from_bytes(data[i + 7:i + 9], "big"), int.from_bytes(data[i + 5:i + 7], "big")
            if mk in (0xD8, 0x01) or 0xD0 <= mk <= 0xD7:
                i += 2
            else:
                i += 2 + int.from_bytes(data[i + 2:i + 4], "big")
    return None, 0, 0


def make_png(w, h, rects, bg=(228, 228, 228)):
    """產生簡單的預覽圖：灰底加上各畫板的色塊。rects = [(x0, y0, x1, y1, (r, g, b))]"""
    rows = []
    for y in range(h):
        row = bytearray(bytes(bg) * w)
        for (x0, y0, x1, y1, c) in rects:
            if y0 <= y < y1 and x1 > x0:
                row[x0 * 3:x1 * 3] = bytes(c) * (x1 - x0)
        rows.append(bytes([0]) + bytes(row))
    chunk = lambda t, d: struct.pack(">I", len(d)) + t + d + struct.pack(">I", zlib.crc32(t + d) & 0xFFFFFFFF)
    return (bytes([0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A]) + chunk(b"IHDR", struct.pack(">IIBBBBB", w, h, 8, 2, 0, 0, 0))
            + chunk(b"IDAT", zlib.compress(b"".join(rows), 9)) + chunk(b"IEND", b""))


class Ctx:
    def __init__(self, images_dir, text=True, effects=True):
        self.images_dir = images_dir
        self.text = text
        self.effects = effects
        self.gradients = {}
        self.images = {}  # uid -> (bytes, w, h, mime)
        self.hash_to_uid = {}
        self.warnings = []
        self.stats = {}

    def count(self, k):
        self.stats[k] = self.stats.get(k, 0) + 1

    def image(self, h):
        if h in self.hash_to_uid:
            uid = self.hash_to_uid[h]
            return uid, self.images[uid][1], self.images[uid][2]
        p = os.path.join(self.images_dir, "img_%s" % h)
        if not os.path.exists(p):
            self.warnings.append("找不到圖片 %s，已改用灰色" % h)
            return None, 0, 0
        data = open(p, "rb").read()
        mime, w, hh = image_info(data)
        if mime is None:
            self.warnings.append("圖片 %s 不是 PNG 或 JPEG，已改用灰色" % h)
            return None, 0, 0
        uid = hashlib.md5(data).hexdigest()
        self.images[uid] = (data, w, hh, mime)
        self.hash_to_uid[h] = uid
        return uid, w, hh

    def fill(self, p):
        t = p["t"]
        if t == "SOLID":
            return {"type": "solid", "color": xd_color(p["c"], p.get("o", 1))}
        if t == "IMAGE":
            uid, w, h = self.image(p.get("h"))
            if uid is None:
                return {"type": "solid", "color": xd_color([0.8, 0.8, 0.8])}
            self.count("圖片填色")
            # XD 的 cover = 等比例填滿並裁切（對應 Figma 的填滿）；fill 會把圖片拉伸變形
            return {"type": "pattern", "pattern": {"width": w, "height": h, "meta": {"ux": {
                "scaleBehavior": "cover",
                "uid": uid, "hrefLastModifiedDate": 0}}, "href": ""}}
        if t == "GRADIENT_LINEAR":
            if not self.effects:
                last = p["st"][-1]["c"]
                return {"type": "solid", "color": xd_color(last[:3], last[3] * p.get("o", 1))}
            gt = p["gt"]
            inv = m_inv([gt[0][0], gt[1][0], gt[0][1], gt[1][1], gt[0][2], gt[1][2]])
            pt = lambda x, y: (inv[0] * x + inv[2] * y + inv[4], inv[1] * x + inv[3] * y + inv[5])
            (x1, y1), (x2, y2) = pt(0, 0.5), pt(1, 0.5)
            gid = new_id()
            self.gradients[gid] = {"type": "linear", "stops": [
                {"offset": rnd(s["p"], 4), "color": xd_color(s["c"][:3], s["c"][3] * p.get("o", 1))} for s in p["st"]]}
            self.count("漸層")
            return {"type": "gradient", "gradient": {"x1": rnd(x1, 4), "y1": rnd(y1, 4), "x2": rnd(x2, 4), "y2": rnd(y2, 4),
                                                     "units": "objectBoundingBox", "ref": gid}}
        self.warnings.append("未支援的填色類型 %s，已改用灰色" % t)
        return {"type": "solid", "color": xd_color([0.5, 0.5, 0.5])}

    def stroke(self, n):
        s = [p for p in n.get("s", []) if p["t"] == "SOLID"]
        sw = n.get("sw", 0)
        if not s or not isinstance(sw, (int, float)) or sw <= 0:
            return NO_STROKE()
        st = {"type": "solid", "color": xd_color(s[0]["c"], s[0].get("o", 1)), "width": rnd(sw)}
        al = {"INSIDE": "inside", "OUTSIDE": "outside"}.get(n.get("sa"))
        if al:
            st["align"] = al
        if n.get("dash"):
            st["dash"] = n["dash"]
        return st

    def filters(self, n, fill_opacity=1.0):
        out = []
        if not self.effects:
            return out
        for e in n.get("fx", []):
            if e["t"] == "DROP_SHADOW":
                c = e["c"]
                out.append({"type": "dropShadow", "params": {"dropShadows": [
                    {"dx": rnd(e.get("x", 0)), "dy": rnd(e.get("y", 0)), "r": rnd(e.get("r", 0)), "color": xd_color(c[:3], c[3])}]}})
                self.count("陰影")
            elif e["t"] == "BACKGROUND_BLUR":
                out.append({"type": "uxdesign#blur", "params": {"blurAmount": rnd(e.get("r", 0)), "brightnessAmount": 0,
                                                              "fillOpacity": rnd(fill_opacity), "backgroundEffect": True}})
                self.count("背景模糊")
            elif e["t"] == "LAYER_BLUR":
                out.append({"type": "uxdesign#blur", "params": {"blurAmount": rnd(e.get("r", 0)), "brightnessAmount": 0,
                                                              "fillOpacity": 1, "backgroundEffect": False}})
            else:
                self.warnings.append("未支援的效果 %s（%s）" % (e["t"], n["n"]))
        return out


# ---------- 形狀 ----------
def radii(n):
    cr = n.get("cr")
    if not cr:
        return None
    r = [cr] * 4 if isinstance(cr, (int, float)) else list(cr)
    lim = min(n["w"], n["h"]) / 2
    r = [rnd(min(x, lim)) for x in r]
    return r if any(r) else None


def rect_shape(n):
    sh = {"type": "rect", "x": 0, "y": 0, "width": rnd(n["w"]), "height": rnd(n["h"])}
    r = radii(n)
    if r:
        sh["r"] = r
    return sh


def norm_path(d):
    d = re.sub("([MLCQZHVAmlcqzhva])", lambda mo: " " + mo.group(1) + " ", d)
    return " ".join(d.split())


def shape_node(name, l10n, shape, style, m):
    node = {"type": "shape", "name": name, "meta": {"ux": {"nameL10N": l10n, "hasCustomName": True}}, "id": new_id()}
    if not is_identity(m):
        node["transform"] = xd_transform(m)
    node["style"] = style
    node["shape"] = shape
    return node


def conv_shape(n, m, ctx):
    """RECTANGLE / ELLIPSE / VECTOR / POLYGON / STAR / LINE / BOOLEAN_OPERATION -> shape 節點清單"""
    out = []
    ty = n["ty"]
    fills = n.get("f", [])
    stroke = ctx.stroke(n)
    has_stroke = stroke["type"] != "none"
    if ty == "RECTANGLE":
        geo, l10n = rect_shape(n), "SHAPE_RECTANGLE"
    elif ty == "ELLIPSE":
        rx, ry = n["w"] / 2, n["h"] / 2
        if abs(rx - ry) < 0.01:
            geo = {"cx": rnd(rx), "cy": rnd(ry), "type": "circle", "r": rnd(rx)}
        else:
            geo = {"cx": rnd(rx), "cy": rnd(ry), "type": "ellipse", "rx": rnd(rx), "ry": rnd(ry)}
        l10n = "SHAPE_ELLIPSE"
    else:
        geo, l10n = None, "SHAPE_PATH"
    if geo is not None:
        if not fills and not has_stroke:
            ctx.count("略過（無填色無邊框）")
            return out
        paints = fills or [None]
        for i, p in enumerate(paints):
            style = {"fill": ctx.fill(p) if p else {"type": "none"}, "stroke": stroke if i == len(paints) - 1 else NO_STROKE()}
            if i == 0:
                op = p.get("o", 1) if p else 1
                fl = ctx.filters(n, op)
                if fl:
                    style["filters"] = fl
                    if any(f["type"] == "uxdesign#blur" and f["params"]["backgroundEffect"] for f in fl) and p and p["t"] == "SOLID":
                        style["fill"] = {"type": "solid", "color": xd_color(p["c"], 1)}
            if n.get("op") is not None:
                style["opacity"] = n["op"]
            out.append(shape_node(n["n"], l10n, dict(geo), style, m))
        ctx.count("矩形" if ty == "RECTANGLE" else "圓形")
        return out
    # 路徑類
    if n.get("fg") and fills:
        d = " ".join(norm_path(g["d"]) for g in n["fg"])
        sh = {"type": "path", "path": d}
        if any(g.get("wr") == "EVENODD" for g in n["fg"]):
            sh["winding"] = "evenodd"
        for p in fills:
            style = {"fill": ctx.fill(p), "stroke": NO_STROKE()}
            if n.get("op") is not None:
                style["opacity"] = n["op"]
            out.append(shape_node(n["n"], "SHAPE_PATH", dict(sh), style, m))
        ctx.count("路徑")
    if n.get("sg") and n.get("s"):
        # 描邊以「外框化後的填色路徑」表示，確保外觀一致
        d = " ".join(norm_path(g["d"]) for g in n["sg"])
        sh = {"type": "path", "path": d}
        p = n["s"][0]
        style = {"fill": ctx.fill(p), "stroke": NO_STROKE()}
        out.append(shape_node(n["n"] + "（描邊）" if out else n["n"], "SHAPE_PATH", sh, style, m))
        ctx.count("描邊路徑（已外框化）")
    if not out:
        ctx.count("略過（無可見幾何）")
    return out


# ---------- 文字 ----------
def is_wide(ch):
    o = ord(ch)
    return o >= 0x2E80 or ch in "•·…—"


def char_w(ch, fam):
    if ch == " ":
        return 0.25 if fam == "Noto Sans TC" else 0.32
    if is_wide(ch):
        return 1.0
    base = 0.64 if fam == "Verdana" else 0.56
    if ch.isdigit():
        return base
    if ch.isupper():
        return base + 0.1
    if ch.isalpha():
        return base - 0.02
    return 0.4


NO_LINE_START = set("，。、；：！？）」』》〉】,.;:!?)")
NL = chr(10)


def break_lines(text, start, size, fam, ls_px, width):
    """粗估折行；回傳 [(from, to, 行寬)]，索引以整段 rawText 為準"""
    lines, i, n = [], 0, len(text)
    tokens = []
    while i < n:
        ch = text[i]
        if is_wide(ch) or ch == " ":
            tokens.append((i, i + 1))
            i += 1
        else:
            j = i
            while j < n and not is_wide(text[j]) and text[j] != " ":
                j += 1
            tokens.append((i, j))
            i = j
    tw = lambda a, b: sum(char_w(c, fam) * size + ls_px for c in text[a:b])
    cur_from, cur_w, cur_to = 0, 0.0, 0
    for (a, b) in tokens:
        w = tw(a, b)
        if cur_to > cur_from and cur_w + w - ls_px > width + 0.5 and text[a] not in NO_LINE_START and text[a] != " ":
            lines.append((start + cur_from, start + cur_to, cur_w))
            cur_from, cur_w = a, 0.0
        cur_w += w
        cur_to = b
    lines.append((start + cur_from, start + cur_to, cur_w))
    return lines


def conv_text(n, m, ctx):
    if not ctx.text:
        return []
    segs = n.get("seg") or []
    raw = n.get("tx", "").replace(chr(0x2028), NL).replace(chr(13), NL)
    if not raw or not segs:
        return []
    s0 = segs[0]
    size, fam, sty = s0["sz"], s0["ff"], s0["fs"]
    auto_ratio, k = FONT_METRICS.get(fam, DEFAULT_METRICS)
    lh_auto = s0["lh"] == "A"
    if lh_auto:
        lh = size * auto_ratio
    elif s0["lh"][0] == "PIXELS":
        lh = s0["lh"][1]
    else:
        lh = size * s0["lh"][1] / 100.0
    ls = s0["ls"]
    char_spacing = ls[1] * 10 if ls[0] == "PERCENT" else (ls[1] / size * 1000 if size else 0)
    ls_px = char_spacing / 1000.0 * size
    base0 = lh / 2 + k * size  # 第一行基線距文字區塊頂端
    align = {"LEFT": "left", "CENTER": "center", "RIGHT": "right", "JUSTIFIED": "left"}[n.get("ah", "LEFT")]
    af = {"left": 0, "center": 0.5, "right": 1}[align]

    def seg_fill(s):
        f = [p for p in s.get("f", []) if p["t"] == "SOLID"]
        return (f[0]["c"], f[0].get("o", 1)) if f else ([0, 0, 0], 1)

    def argb(c, a):
        return (round(a * 255) << 24) | (round(c[0] * 255) << 16) | (round(c[1] * 255) << 8) | round(c[2] * 255)

    ranged = []
    for s in segs:
        c, a = seg_fill(s)
        sls = s["ls"]
        scs = sls[1] * 10 if sls[0] == "PERCENT" else (sls[1] / s["sz"] * 1000 if s["sz"] else 0)
        ranged.append({"length": s["b"] - s["a"], "fontFamily": s["ff"], "fontStyle": s["fs"], "fontSize": s["sz"],
                       "charSpacing": rnd(scs), "underline": s.get("td") == "UNDERLINE", "strikethrough": s.get("td") == "STRIKETHROUGH",
                       "textTransform": {"UPPER": "uppercase", "LOWER": "lowercase", "TITLE": "titlecase"}.get(s.get("tc"), "none"),
                       "textScript": "none", "fill": {"value": argb(c, a)}})
    if len(segs) > 1:
        ctx.count("多重樣式文字")

    paras_src, pos = [], 0
    for part in raw.split(NL):
        paras_src.append((pos, part))
        pos += len(part) + 1

    ar = n.get("ar", "WIDTH_AND_HEIGHT")
    W, H = n["w"], n["h"]
    paragraphs = []
    if ar == "WIDTH_AND_HEIGHT":
        # 點文字：錨點在第一行基線，水平錨點依對齊方式
        frame = {"type": "positioned"}
        y = 0.0
        for (st, part) in paras_src:
            paragraphs.append({"lines": [[{"x": 0, "y": rnd(y, 2), "from": st, "to": st + len(part)}]]})
            y += lh
        local = [1, 0, 0, 1, W * af, base0]
        ctx.count("點文字")
    else:
        all_lines = []
        for (st, part) in paras_src:
            all_lines.append(break_lines(part, st, size, fam, ls_px, W))
        nlines = sum(len(x) for x in all_lines)
        content_h = nlines * lh
        av = n.get("av", "TOP")
        if ar == "HEIGHT" or av == "TOP":
            off = 0.0
        elif av == "CENTER":
            off = (H - content_h) / 2
        else:
            off = H - content_h
        if ar == "NONE" and av == "TOP":
            frame = {"type": "area", "width": rnd(W), "height": rnd(H)}
            ctx.count("區域文字（固定高）")
        else:
            frame = {"type": "autoHeight", "width": rnd(W)}
            ctx.count("區域文字（自動高）")
        y = base0
        for lines in all_lines:
            pl = []
            for (a, b, lw) in lines:
                x = max(0.0, (W - max(lw - ls_px, 0)) * af)
                pl.append([{"x": rnd(x, 2), "y": rnd(y, 2), "from": a, "to": b}])
                y += lh
            paragraphs.append({"lines": pl})
        local = [1, 0, 0, 1, 0, off]

    c0, a0 = seg_fill(s0)
    ps = fam.replace(" ", "") + ("-" + sty.replace(" ", "") if fam not in ("Verdana", "Tahoma") or sty != "Regular" else "")
    style = {"fill": {"type": "solid", "color": xd_color(c0, a0)},
             "font": {"postscriptName": ps, "family": fam, "style": sty, "size": size}}
    ta = {}
    if not lh_auto:
        ta["lineHeight"] = rnd(lh, 2)
    if align != "left":
        ta["paragraphAlign"] = align
    if abs(char_spacing) > 0.01:
        ta["letterSpacing"] = rnd(char_spacing)
    if ta:
        style["textAttributes"] = ta
    if n.get("op") is not None:
        style["opacity"] = n["op"]
    name = raw.replace(NL, " ").strip()[:60] or n["n"]
    node = {"type": "text", "name": name, "meta": {"ux": {"singleTextObject": True, "rangedStyles": ranged}}, "id": new_id(),
            "transform": xd_transform(m_mul(m, local)), "style": style,
            "text": {"frame": frame, "paragraphs": paragraphs, "rawText": raw}}
    return [node]


# ---------- 容器 ----------
def overflows(n):
    """子孫是否超出本節點範圍（以未旋轉近似）"""
    x0, y0 = n["at"][4], n["at"][5]
    x1, y1 = x0 + n["w"], y0 + n["h"]

    def chk(c):
        if c.get("hid"):
            return False
        cx, cy = c["at"][4], c["at"][5]
        if cx < x0 - 0.5 or cy < y0 - 0.5 or cx + c["w"] > x1 + 0.5 or cy + c["h"] > y1 + 0.5:
            return True
        return any(chk(g) for g in c.get("ch", []))

    return any(chk(c) for c in n.get("ch", []))


def conv_node(n, parent_abs, ctx):
    m = m_mul(m_inv(parent_abs), n["at"])
    ty = n["ty"]
    if ty == "TEXT":
        out = conv_text(n, m, ctx)
    elif ty in ("FRAME", "GROUP", "INSTANCE", "COMPONENT", "COMPONENT_SET", "SECTION"):
        out = conv_group(n, m, ctx)
    else:
        out = conv_shape(n, m, ctx)
    if n.get("hid"):
        for o in out:
            o["visible"] = False
    return out


def conv_group(n, m, ctx):
    children = []
    ident = [1, 0, 0, 1, 0, 0]
    if n.get("f") or (n.get("s") and isinstance(n.get("sw"), (int, float))):
        bg = dict(n)
        bg["ty"] = "RECTANGLE"
        bg["n"] = "背景"
        bg.pop("op", None)
        children += conv_shape(bg, ident, ctx)
    for c in n.get("ch", []):
        children += conv_node(c, n["at"], ctx)
    if not children:
        ctx.count("略過（空群組）")
        return []
    ux = {"nameL10N": "SHAPE_GROUP", "hasCustomName": True}
    if n.get("clip") and overflows(n):
        clip = {"type": "shape", "name": "遮罩", "meta": {"ux": {"nameL10N": "SHAPE_RECTANGLE"}}, "id": new_id(),
                "style": {"fill": {"type": "solid", "color": xd_color([1, 1, 1])}, "stroke": NO_STROKE()}, "shape": rect_shape(n)}
        ux["clipPathResources"] = {"children": [clip], "type": "clipPath"}
        ctx.count("遮罩群組")
    node = {"type": "group", "name": n["n"], "meta": {"ux": ux}, "id": new_id()}
    if not is_identity(m):
        node["transform"] = xd_transform(m)
    if n.get("op") is not None:
        node["style"] = {"opacity": n["op"]}
    node["group"] = {"children": children}
    ctx.count("元件實體→群組" if n["ty"] == "INSTANCE" else "群組")
    return [node]


# ---------- 打包 ----------
def jdump(o):
    return json.dumps(o, ensure_ascii=False, separators=(",", ":")).encode("utf-8")


def build(tree, frame_idx, ctx, title, preview=None, thumb=None):
    frames = [tree["frames"][i] for i in frame_idx]
    min_x = min(f["at"][4] for f in frames)
    min_y = min(f["at"][5] for f in frames)
    artboards_res, artboard_files, manifest_artboards = {}, [], []
    for f in frames:
        ax, ay = rnd(f["at"][4] - min_x), rnd(f["at"][5] - min_y)
        ref = new_id()
        path = "artboard-" + ref
        # 畫板內圖層以文件座標儲存：文件座標 = 相對畫框座標 + 畫板位置
        parent = [1, 0, 0, 1, f["at"][4] - ax, f["at"][5] - ay]
        children = []
        for c in f.get("ch", []):
            children += conv_node(c, parent, ctx)
        bg = [p for p in f.get("f", []) if p["t"] == "SOLID"]
        style = {"fill": {"type": "solid", "color": xd_color(bg[0]["c"] if bg else [1, 1, 1])}}
        agc = {"version": AGC_VERSION, "children": [{"type": "artboard", "id": new_id(), "meta": {"ux": {"path": path}}, "style": style,
                                                      "artboard": {"children": children, "meta": {"ux": {"path": path}}, "ref": ref}}],
               "resources": {"href": RES_HREF}, "artboards": {"href": RES_HREF}}
        W, H = rnd(f["w"]), rnd(f["h"])
        artboards_res[ref] = {"width": W, "height": H, "name": f["n"], "x": ax, "y": ay, "viewportHeight": H}
        artboard_files.append(("artwork/%s/graphics/graphicContent.agc" % path, jdump(agc)))
        manifest_artboards.append({"id": new_id(), "name": f["n"], "path": path,
                                   "children": [{"id": new_id(), "name": "graphics", "path": "graphics", "components": [
                                       {"id": new_id(), "name": "", "state": "unmodified", "path": "graphicContent.agc", "type": AGC_TYPE, "rel": "primary"}]}],
                                   "uxdesign#bounds": {"x": ax, "y": ay, "width": W, "height": H}, "uxdesign#viewport": {"height": H}})

    pasteboard = {"version": AGC_VERSION, "children": [], "resources": {"href": RES_HREF}, "artboards": {"href": RES_HREF}}
    resources = {"version": AGC_VERSION, "children": [], "resources": {"meta": {"ux": {
        "colorSwatches": [], "documentLibrary": {"version": 5, "isStickerSheet": False, "hashedMetadata": {"publishedDocLibId": None},
                                                 "elements": [], "groupData": {"groups": []}},
        "gridDefaults": {"defaultGrid": None, "layoutOverrides": None}, "colorProfile": "srgb", "symbols": [],
        "symbolsMetadata": {"usingNestedSymbolSyncing": True}}}, "gradients": ctx.gradients, "clipPaths": {}}, "artboards": artboards_res}

    doc_id = new_id()
    comp = lambda path, typ, rel="primary", name="", **kw: dict({"id": new_id(), "name": name, "state": "unmodified", "path": path, "type": typ, "rel": rel}, **kw)
    manifest = {"id": doc_id, "uxdesign#parentDocId": new_id(), "uxdesign#id": new_id(), "children": [
        {"id": new_id(), "name": "artwork", "path": "artwork", "children": manifest_artboards + [
            {"id": new_id(), "name": "pasteboard", "path": "pasteboard", "children": [
                {"id": new_id(), "name": "graphics", "path": "graphics", "components": [comp("graphicContent.agc", AGC_TYPE)]}]}]},
        {"id": new_id(), "name": "interactions", "path": "interactions",
         "components": [comp("interactions.json", "application/vnd.adobe.uxdesign.interactions+json", name="interactions")]},
        {"id": new_id(), "name": "resources", "path": "resources", "children": [
            {"id": new_id(), "name": "graphics", "path": "graphics", "components": [comp("graphicContent.agc", AGC_TYPE)]}],
         "components": [comp(uid, v[3]) for uid, v in ctx.images.items()]},
        {"id": new_id(), "name": "sharing", "path": "sharing",
         "components": [comp("sharing.json", "application/vnd.adobe.uxdesign.sharing+json", name="sharing")]}],
        "uxdesign#applicationVersion": APP_VERSION, "uxdesign#version": UX_VERSION, "uxdesign#serializationId": new_id(),
        "uxdesign#compositeID": doc_id, "components": [], "name": title, "type": "application/vnd.adobe.sparkler.project+dcx",
        "manifest-format-version": 6, "state": "unmodified"}

    def png_size(b):
        return int.from_bytes(b[16:20], "big"), int.from_bytes(b[20:24], "big")

    def placeholder(max_dim):
        tw = max(v["x"] + v["width"] for v in artboards_res.values())
        th = max(v["y"] + v["height"] for v in artboards_res.values())
        sc = max_dim / max(tw, th)
        pw, ph = max(1, round(tw * sc)), max(1, round(th * sc))
        rects = [(round(v["x"] * sc), round(v["y"] * sc), min(pw, round((v["x"] + v["width"]) * sc)),
                  min(ph, round((v["y"] + v["height"]) * sc)), (255, 255, 255)) for v in artboards_res.values()]
        return make_png(pw, ph, rects)

    extra = []
    tb = open(thumb, "rb").read() if thumb else placeholder(512)
    if tb:
        w, h = png_size(tb)
        rp = "renditions/image-%d-%d.png" % (w, h)
        manifest["components"] += [comp(rp, "image/png", "rendition", "thumbnail", height=h, width=w),
                                   comp("thumbnail.png", "image/png", "thumbnail", "thumbnail", height=h, width=w)]
        extra += [(rp, tb), ("thumbnail.png", tb)]
    pb = open(preview, "rb").read() if preview else placeholder(1024)
    if pb:
        w, h = png_size(pb)
        rp = "renditions/image-%d-%d.png" % (w, h)
        manifest["components"] += [comp(rp, "image/png", "rendition", "preview", height=h, width=w),
                                   comp("preview.png", "image/png", "preview", "preview", height=h, width=w)]
        extra += [(rp, pb), ("preview.png", pb)]
    manifest["components"].append(comp("META-INF/metadata.xml", "application/rdf+xml", "metadata", "xmp-metadata"))

    now = datetime.datetime.now(datetime.timezone(datetime.timedelta(hours=8))).strftime("%Y-%m-%dT%H:%M:%S.000+08:00")
    did, iid, iid0 = new_id(), new_id(), new_id()
    xmp_lines = [
        '<?xpacket begin="%(bom)s" id="W5M0MpCehiHzreSzNTczkc9d"?>',
        '<x:xmpmeta xmlns:x="adobe:ns:meta/" x:xmptk="Adobe XMP Core 9.0-c001 1.000000, 0000/00/00-00:00:00        ">',
        ' <rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">',
        '  <rdf:Description rdf:about=""',
        '    xmlns:xmp="http://ns.adobe.com/xap/1.0/"',
        '    xmlns:dc="http://purl.org/dc/elements/1.1/"',
        '    xmlns:xmpMM="http://ns.adobe.com/xap/1.0/mm/"',
        '    xmlns:stEvt="http://ns.adobe.com/xap/1.0/sType/ResourceEvent#"',
        '   xmp:CreatorTool="XD"',
        '   xmp:CreateDate="%(now)s"',
        '   xmp:ModifyDate="%(now)s"',
        '   xmp:MetadataDate="%(now)s"',
        '   dc:format="application/vnd.adobe.sparkler.project+dcx"',
        '   xmpMM:DocumentID="%(did)s"',
        '   xmpMM:OriginalDocumentID="%(did)s"',
        '   xmpMM:InstanceID="%(iid)s">',
        '   <dc:title>',
        '    <rdf:Alt>',
        '     <rdf:li xml:lang="x-default">%(title)s</rdf:li>',
        '    </rdf:Alt>',
        '   </dc:title>',
        '   <xmpMM:History>',
        '    <rdf:Seq>',
        '     <rdf:li',
        '      stEvt:action="created"',
        '      stEvt:instanceID="%(iid0)s"',
        '      stEvt:when="%(now)s"',
        '      stEvt:softwareAgent="XD"/>',
        '     <rdf:li',
        '      stEvt:action="saved"',
        '      stEvt:instanceID="%(iid)s"',
        '      stEvt:when="%(now)s"',
        '      stEvt:softwareAgent="XD"/>',
        '    </rdf:Seq>',
        '   </xmpMM:History>',
        '  </rdf:Description>',
        ' </rdf:RDF>',
        '</x:xmpmeta>',
    ]
    xmp = (NL.join(xmp_lines) + NL) % {"bom": chr(0xFEFF), "now": now, "did": did, "iid": iid, "iid0": iid0, "title": title}
    xmp += (" " * 100 + NL) * 20 + '<?xpacket end="w"?>'

    files = [("mimetype", b"application/vnd.adobe.sparkler.project+dcxucf", False),
             ("manifest", jdump(manifest), True),
             ("META-INF/metadata.xml", xmp.encode("utf-8"), False)]
    files += [(p, b, True) for (p, b) in artboard_files]
    files += [("artwork/pasteboard/graphics/graphicContent.agc", jdump(pasteboard), True),
              ("interactions/interactions.json", b'{"version":"0.24"}', True)]
    files += [(p, b, False) for (p, b) in extra]
    files += [("resources/" + uid, v[0], False) for uid, v in ctx.images.items()]
    files += [("resources/graphics/graphicContent.agc", jdump(resources), True),
              ("sharing/sharing.json", b'{"version":"0.3","selectedLink":-1}', True)]
    return files


def write_xd(files, out):
    with zipfile.ZipFile(out, "w") as z:
        for (path, data, deflate) in files:
            zi = zipfile.ZipInfo(path, date_time=datetime.datetime.now().timetuple()[:6])
            zi.compress_type = zipfile.ZIP_DEFLATED if deflate else zipfile.ZIP_STORED
            zi.external_attr = 0o644 << 16
            z.writestr(zi, data)


def decode(src, outdir):
    """把暫存在 Figma 圖層名稱裡的資料（base64 分段）還原成 tree.json 與圖片檔"""
    import base64, html
    txt = open(src, encoding="utf-8").read()
    try:
        arr = json.loads(txt)
        if isinstance(arr, list):
            txt = NL.join(a.get("text", "") for a in arr if isinstance(a, dict))
    except ValueError:
        pass
    blobs = {}
    for mt in re.finditer('name="([^"|]+)[|]([0-9]+)[|]([0-9]{5}):([^"]*)"', txt):
        key, ln, idx, data = mt.group(1), int(mt.group(2)), int(mt.group(3)), html.unescape(mt.group(4))
        blobs.setdefault(key, {"len": ln, "chunks": {}})["chunks"][idx] = data
    blobs.pop("pad", None)
    if "tree" not in blobs:
        raise SystemExit("找不到圖層資料：請確認 get_metadata 是對暫存畫框本身呼叫，且結果完整")
    os.makedirs(outdir, exist_ok=True)
    for key, b in blobs.items():
        s = "".join(b["chunks"][i] for i in sorted(b["chunks"]))
        if len(s) != b["len"]:
            raise SystemExit("資料不完整：%s 應為 %d 字元，實得 %d" % (key, b["len"], len(s)))
        raw = base64.b64decode(s)
        if key == "tree":
            tree = json.loads(raw.decode("utf-8"))
            json.dump(tree, open(os.path.join(outdir, "tree.json"), "w", encoding="utf-8"), ensure_ascii=False)
            for i, f in enumerate(tree["frames"]):
                print("畫框 %d: %s (%s x %s)" % (i, f["n"], f["w"], f["h"]))
        else:
            open(os.path.join(outdir, key), "wb").write(raw)
            print("圖片:", key, len(raw), "bytes")


def main():
    import sys
    if len(sys.argv) >= 2 and sys.argv[1] == "decode":
        if len(sys.argv) != 4:
            raise SystemExit("用法: figma2xd.py decode <get_metadata結果檔> <工作資料夾>")
        return decode(sys.argv[2], sys.argv[3])
    if len(sys.argv) >= 2 and sys.argv[1] == "build":
        sys.argv.pop(1)
    ap = argparse.ArgumentParser()
    ap.add_argument("--tree", required=True)
    ap.add_argument("--images", required=True)
    ap.add_argument("--out", required=True)
    ap.add_argument("--frames", default="")
    ap.add_argument("--title", default="")
    ap.add_argument("--no-text", action="store_true")
    ap.add_argument("--no-effects", action="store_true")
    ap.add_argument("--preview")
    ap.add_argument("--thumb")
    a = ap.parse_args()
    tree = json.load(open(a.tree, encoding="utf-8"))
    idx = [int(x) for x in a.frames.split(",")] if a.frames else sorted(range(len(tree["frames"])), key=lambda i: tree["frames"][i]["at"][4])
    ctx = Ctx(a.images, text=not a.no_text, effects=not a.no_effects)
    title = a.title or os.path.splitext(os.path.basename(a.out))[0]
    files = build(tree, idx, ctx, title, a.preview, a.thumb)
    write_xd(files, a.out)
    print(os.path.basename(a.out), os.path.getsize(a.out), "bytes")
    print("  統計:", json.dumps(ctx.stats, ensure_ascii=False))
    for w in sorted(set(ctx.warnings)):
        print("  警告:", w)


if __name__ == "__main__":
    main()
```

## 乙、Adobe XD 轉 Figma

讀取 `.xd` 檔，在指定的 Figma 檔案中重建圖層：每個 XD 畫板變成一個畫框，文字是真正的文字圖層，矩形與橢圓是可調整的形狀，群組階層與圖層名稱都保留。成果會放在目標頁面上一個名為「XD 匯入：<檔名>」的區段裡。

做法分三段：用 Python 把 `.xd` 讀成一份壓縮過的「建構計畫」；把計畫與圖片資料送進 Figma（暫存在名為 `__xd_chunks__` 的圖層名稱裡）；再用一段固定的腳本讀取資料並建立圖層。

### 前置條件

- 需要 Figma 連接器的 `use_figma`。呼叫前若有 figma-use 技能要先載入。
- 連接的 Figma 帳號對目標檔案必須有編輯權限。沒有時請使用者把檔案「複製到你的草稿」再給連結，或用 `create_new_file` 建新檔（先載入 figma-create-new-file 技能；使用者有多個團隊時要先問放在哪一個）。
- 需要使用者提供 `.xd` 檔，以及要放進哪個 Figma 檔案、哪一頁。
- `use_figma` 在 Figma 的伺服器環境執行，只能使用 Figma 提供的字型。只安裝在使用者電腦上的字型（例如 Verdana、Tahoma、微軟正黑體）會被替換成相近字重的備用字型，並列在回傳的 `fontSubs`。

### 步驟

1. 把下方「讀取程式」原樣寫入工作目錄的 `xd2figma.py`。不要改寫或精簡。
2. 執行 `python3 xd2figma.py <檔案.xd> work`。輸出會列出根節點（畫板）、統計、使用的字型、圖片數量、資料總量與警告。
3. 取得目標頁面的節點 ID：用 `use_figma` 執行 `return figma.root.children.map(p => ({ id: p.id, name: p.name }));`。
4. 把資料送進 Figma，依資料量擇一：
   - **直接模式**（資料總量約 80,000 字元以下）：對 `work/parts/` 裡的每個 `part_NN.txt`，用 Read 讀出內容，整段貼進下方「載入腳本」的 `CH` 陣列，設定 `PAGE_ID` 後用 `use_figma` 執行。每個檔案一次呼叫。資料必須逐字照抄，不可省略、截斷或改寫。回傳的 `badItems` 是檢查碼不符的項目索引，代表抄寫出錯：重新讀檔，只重送那些項目即可（已載入的項目會自動略過）。全部送完後 `incomplete` 必須是空陣列。
   - **檔案模式**（資料量大，或圖片多）：先試自動上傳：呼叫 `upload_assets`（`count` 為 1、帶 `currentPageId`），再執行 `curl -sS -X POST -F "file=@work/<檔名>_匯入用.svg;type=image/svg+xml" "<submitUrl>"`。如果連線被網路限制擋下（例如 403），不要換其他方式重試，改把 `work/<檔名>_匯入用.svg` 交給使用者，請他把檔案拖進目標頁面，完成後回報。
   - **折衷**：加上 `--no-images` 重跑步驟 2，再用直接模式只送建構計畫。圖片位置會先以灰色色塊代替，並列在回傳的 `missingImages`，事後請使用者自行置換。
5. 用 `use_figma` 執行下方「建構腳本」，只修改最上方的 `PAGE_ID`。檢查回傳：`errors` 應為空；`fontSubs` 是被替換的字型；`missingImages` 是沒載入或資料損壞的圖片。
   - 如果呼叫逾時：把 `ONLY` 設成 `[0]`、`CLEANUP` 設成 `false`，一次建立一個根節點，最後一次再把 `CLEANUP` 設回 `true`。同一個索引重跑會重複建立，重跑前先刪掉已建立的那個畫框。
6. 對回傳的 `sectionId` 呼叫 `get_screenshot` 檢查結果，再向使用者回報：建立了哪些畫板、哪些字型被替換、哪些圖片缺漏，以及下方的已知限制。

### 已知限制（回報時要告知）

- 元件與元件實體變成一般群組或畫框，重複格線變成群組；自動排版、原型連結、樣式庫不會轉換。
- 文字的垂直位置是依字型量測值推算的，可能有一兩個像素的差異；多行文字由 Figma 重新折行。
- 矩形遮罩變成有裁切的畫框；其他形狀的遮罩變成 Figma 的遮罩圖層。
- 放射狀漸層只做近似；複合形狀（布林運算）變成單一路徑。
- 圖片支援 PNG 與 JPEG。

### 疑難排解

- 建構腳本回報「找不到資料」：資料沒送進目標頁面，或 `PAGE_ID` 指到別頁。檔案模式下確認使用者是把 SVG 拖進同一頁。
- 「計畫資料不完整」：有資料段沒載入，重跑載入腳本並確認 `incomplete` 為空。
- 文字位置有固定偏移：在 `xd2figma.py` 的 `FONT_K` 為該字型補上基線係數後重跑。
- 建構中途失敗而留下暫存圖層：用 `use_figma` 刪除頁面上所有名為 `__xd_chunks__` 的圖層。

### 載入腳本（use_figma，直接模式用）

```js
const PAGE_ID = null;   // 目標頁面的節點 ID；null 表示檔案的第一頁
const CH = [
];
const page = PAGE_ID ? await figma.getNodeByIdAsync(PAGE_ID) : figma.currentPage;
await figma.setCurrentPageAsync(page);
function checksum(s) { let a = 1, b = 0; for (let i = 0; i < s.length; i++) { a = (a + s.charCodeAt(i)) % 65521; b = (b + a) % 65521; } return b * 65536 + a; }
let holder = page.children.find(n => n.type === 'FRAME' && n.name === '__xd_chunks__');
if (!holder) { holder = figma.createFrame(); holder.name = '__xd_chunks__'; holder.fills = []; holder.clipsContent = false; holder.resize(5000, 5000); holder.x = -100000; holder.y = -100000; }
const have = {}; for (const c of holder.children) have[c.name.slice(0, c.name.indexOf(':'))] = c;
const bad = []; let added = 0, n = holder.children.length;
for (let i = 0; i < CH.length; i++) {
  const s = CH[i][1];
  if (checksum(s) !== CH[i][0]) { bad.push(i); continue; }
  const head = s.slice(0, s.indexOf(':'));
  if (have[head]) continue;
  const r = figma.createRectangle(); r.resize(100, 100); r.name = s;
  holder.appendChild(r); r.x = (n % 40) * 120; r.y = Math.floor(n / 40) * 120; n++; added++; have[head] = r;
}
const keys = {};
for (const c of holder.children) {
  const m = /^([^|]+)[|]([0-9]+)[|]/.exec(c.name); if (!m) continue;
  const k = keys[m[1]] || (keys[m[1]] = { need: +m[2], have: 0 });
  k.have += c.name.length - c.name.indexOf(':') - 1;
}
const incomplete = Object.keys(keys).filter(k => keys[k].have !== keys[k].need);
return { createdNodeIds: [holder.id], holderId: holder.id, added, badItems: bad, total: n, incomplete };
```

### 建構腳本（use_figma）

```js
const PAGE_ID = null;              // 目標頁面的節點 ID（例如 '12:34'）；null 表示檔案的第一頁
const ONLY = null;                 // null = 建立全部；或填根節點索引陣列（例如 [0]）分批建立
const CLEANUP = true;              // 建立完成後刪除暫存資料圖層
const FALLBACK_FONT = 'Noto Sans TC';
const PAD = 100;
function inflate(src) {
  let pos = 0, bitBuf = 0, bitCnt = 0; const out = [];
  const bits = n => { while (bitCnt < n) { bitBuf |= src[pos++] << bitCnt; bitCnt += 8; } const v = bitBuf & ((1 << n) - 1); bitBuf >>>= n; bitCnt -= n; return v; };
  const build = lens => { const count = new Uint16Array(16), offs = new Uint16Array(16), symbol = new Uint16Array(lens.length);
    for (const l of lens) count[l]++; count[0] = 0;
    for (let i = 1; i < 16; i++) offs[i] = offs[i - 1] + count[i - 1];
    for (let s = 0; s < lens.length; s++) if (lens[s]) symbol[offs[lens[s]]++] = s;
    return { count, symbol }; };
  const decode = h => { let code = 0, first = 0, index = 0;
    for (let len = 1; len < 16; len++) { code |= bits(1); const c = h.count[len]; if (code - c < first) return h.symbol[index + (code - first)]; index += c; first += c; first <<= 1; code <<= 1; }
    throw new Error('資料解壓縮失敗'); };
  const LB = [3, 4, 5, 6, 7, 8, 9, 10, 11, 13, 15, 17, 19, 23, 27, 31, 35, 43, 51, 59, 67, 83, 99, 115, 131, 163, 195, 227, 258];
  const LE = [0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 2, 2, 2, 2, 3, 3, 3, 3, 4, 4, 4, 4, 5, 5, 5, 5, 0];
  const DB = [1, 2, 3, 4, 5, 7, 9, 13, 17, 25, 33, 49, 65, 97, 129, 193, 257, 385, 513, 769, 1025, 1537, 2049, 3073, 4097, 6145, 8193, 12289, 16385, 24577];
  const DE = [0, 0, 0, 0, 1, 1, 2, 2, 3, 3, 4, 4, 5, 5, 6, 6, 7, 7, 8, 8, 9, 9, 10, 10, 11, 11, 12, 12, 13, 13];
  const ORD = [16, 17, 18, 0, 8, 7, 9, 6, 10, 5, 11, 4, 12, 3, 13, 2, 14, 1, 15];
  let fixL = null, fixD = null, last = 0;
  do {
    last = bits(1); const type = bits(2);
    if (type === 0) { bitBuf = 0; bitCnt = 0; const len = src[pos] | (src[pos + 1] << 8); pos += 4; for (let i = 0; i < len; i++) out.push(src[pos++]); }
    else {
      let hl, hd;
      if (type === 1) {
        if (!fixL) { const l = []; for (let i = 0; i < 288; i++) l.push(i < 144 ? 8 : i < 256 ? 9 : i < 280 ? 7 : 8); fixL = build(l); fixD = build(new Array(30).fill(5)); }
        hl = fixL; hd = fixD;
      } else if (type === 2) {
        const nl = bits(5) + 257, nd = bits(5) + 1, nc = bits(4) + 4; const cl = new Array(19).fill(0);
        for (let i = 0; i < nc; i++) cl[ORD[i]] = bits(3);
        const hc = build(cl); const lens = [];
        while (lens.length < nl + nd) { const s = decode(hc); if (s < 16) lens.push(s); else { let rep, val = 0; if (s === 16) { val = lens[lens.length - 1]; rep = 3 + bits(2); } else if (s === 17) rep = 3 + bits(3); else rep = 11 + bits(7); while (rep--) lens.push(val); } }
        hl = build(lens.slice(0, nl)); hd = build(lens.slice(nl));
      } else throw new Error('資料解壓縮失敗');
      for (;;) { let s = decode(hl); if (s < 256) out.push(s); else if (s === 256) break; else { s -= 257; const len = LB[s] + bits(LE[s]); const ds = decode(hd); const dist = DB[ds] + bits(DE[ds]); const from = out.length - dist; for (let i = 0; i < len; i++) out.push(out[from + i]); } }
    }
  } while (!last);
  return out;
}
function utf8dec(b) {
  let s = '', i = 0; const parts = [];
  while (i < b.length) {
    let c = b[i++];
    if (c >= 0xf0) { c = (((c & 7) << 18) | ((b[i++] & 63) << 12) | ((b[i++] & 63) << 6) | (b[i++] & 63)) - 0x10000; parts.push(String.fromCharCode(0xd800 + (c >> 10), 0xdc00 + (c & 1023))); }
    else if (c >= 0xe0) parts.push(String.fromCharCode(((c & 15) << 12) | ((b[i++] & 63) << 6) | (b[i++] & 63)));
    else if (c >= 0xc0) parts.push(String.fromCharCode(((c & 31) << 6) | (b[i++] & 63)));
    else parts.push(String.fromCharCode(c));
    if (parts.length > 8192) { s += parts.join(''); parts.length = 0; }
  }
  return s + parts.join('');
}
const page = PAGE_ID ? await figma.getNodeByIdAsync(PAGE_ID) : figma.currentPage;
await figma.setCurrentPageAsync(page);
const holders = page.findAll(n => n.name === '__xd_chunks__');
const blobs = {};
for (const h of holders) for (const c of h.children) {
  const m = /^([^|]+)[|]([0-9]+)[|]([0-9]{5}):(.*)$/.exec(c.name);
  if (!m) continue;
  const b = blobs[m[1]] || (blobs[m[1]] = { len: +m[2], parts: {} });
  b.parts[+m[3]] = m[4];
}
function text64(key) {
  const b = blobs[key]; if (!b) return null;
  const s = Object.keys(b.parts).map(Number).sort((x, y) => x - y).map(k => b.parts[k]).join('');
  return s.length === b.len ? s : null;
}
function checksum(s) { let a = 1, b = 0; for (let i = 0; i < s.length; i++) { a = (a + s.charCodeAt(i)) % 65521; b = (b + a) % 65521; } return b * 65536 + a; }
if (!blobs.plan) throw new Error('找不到資料：頁面上沒有名為 __xd_chunks__ 的圖層');
const plan64 = text64('plan');
if (!plan64) throw new Error('計畫資料不完整，請重新載入');
const plan = JSON.parse(utf8dec(inflate(figma.base64Decode(plan64))));
// ---- 字型 ----
const avail = await figma.listAvailableFontsAsync();
const fam = {};
for (const f of avail) (fam[f.fontName.family] = fam[f.fontName.family] || []).push(f.fontName.style);
const flat = s => s.toLowerCase().split(' ').join('').split('-').join('');
const wt = s0 => { const s = flat(s0);
  if (s.includes('thin') || s.includes('hairline')) return 100;
  if (s.includes('extralight') || s.includes('ultralight')) return 200;
  if (s.includes('demilight')) return 350;
  if (s.includes('semibold') || s.includes('demibold')) return 600;
  if (s.includes('extrabold') || s.includes('ultrabold')) return 800;
  if (s.includes('light')) return 300;
  if (s.includes('medium')) return 500;
  if (s.includes('bold')) return 700;
  if (s.includes('black') || s.includes('heavy')) return 900;
  return 400; };
const fontCache = {}, fontSubs = {};
function pick(family, style) {
  const key = family + '|' + style;
  if (fontCache[key]) return fontCache[key];
  let f = fam[family] ? family : (fam[FALLBACK_FONT] ? FALLBACK_FONT : 'Inter');
  const styles = fam[f];
  let st = styles.find(x => x === style) || styles.find(x => flat(x) === flat(style));
  if (!st) { const it = flat(style).includes('italic'); const pool = styles.filter(x => flat(x).includes('italic') === it); const cand = pool.length ? pool : styles; const w = wt(style); st = cand.slice().sort((a, b) => Math.abs(wt(a) - w) - Math.abs(wt(b) - w))[0]; }
  if (f !== family || st !== style) fontSubs[family + ' ' + style] = f + ' ' + st;
  return (fontCache[key] = { family: f, style: st });
}
await figma.loadFontAsync({ family: 'Inter', style: 'Regular' });
for (const f of plan.fonts) await figma.loadFontAsync(pick(f[0], f[1]));
// ---- 填色、邊框、效果 ----
const imgCache = {}, missingImages = [];
function imgHash(uid) {
  if (uid in imgCache) return imgCache[uid];
  const s = text64('img_' + uid);
  if (!s || checksum(s) !== plan.imgs[uid]) { missingImages.push(uid + (s ? '（資料損壞）' : blobs['img_' + uid] ? '（資料不完整）' : '（未載入）')); return (imgCache[uid] = null); }
  return (imgCache[uid] = figma.createImage(figma.base64Decode(s)).hash);
}
const rgb = c => ({ r: c[0] / 255, g: c[1] / 255, b: c[2] / 255 });
function paints(list) {
  const out = [];
  for (const f of list || []) {
    if (f[0] === 's') out.push({ type: 'SOLID', color: rgb(f.slice(1)), opacity: f[4] });
    else if (f[0] === 'g') {
      const stops = f[6].map(s => ({ position: Math.min(1, Math.max(0, s[0])), color: { r: s[1] / 255, g: s[2] / 255, b: s[3] / 255, a: s[4] } }));
      if (f[1] === 'l') { const dx = f[4] - f[2], dy = f[5] - f[3], L = dx * dx + dy * dy || 1; out.push({ type: 'GRADIENT_LINEAR', gradientStops: stops, gradientTransform: [[dx / L, dy / L, -(f[2] * dx + f[3] * dy) / L], [-dy / L, dx / L, (f[2] * dy - f[3] * dx) / L + 0.5]] }); }
      else { const r = f[4] || 0.5; out.push({ type: 'GRADIENT_RADIAL', gradientStops: stops, gradientTransform: [[1 / (2 * r), 0, 0.5 - f[2] / (2 * r)], [0, 1 / (2 * r), 0.5 - f[3] / (2 * r)]] }); }
    } else if (f[0] === 'i') {
      const h = imgHash(f[1]);
      if (!h) out.push({ type: 'SOLID', color: { r: 0.8, g: 0.8, b: 0.8 } });
      else if (f[2] === 'STRETCH') out.push({ type: 'IMAGE', imageHash: h, scaleMode: 'CROP', imageTransform: [[1, 0, 0], [0, 1, 0]] });
      else out.push({ type: 'IMAGE', imageHash: h, scaleMode: 'FILL' });
    }
  }
  return out;
}
function effects(list) {
  const out = [];
  for (const e of list) {
    if (e[0] === 'd') out.push({ type: 'DROP_SHADOW', color: { r: e[4][0] / 255, g: e[4][1] / 255, b: e[4][2] / 255, a: e[4][3] }, offset: { x: e[1], y: e[2] }, radius: e[3], spread: 0, visible: true, blendMode: 'NORMAL' });
    else out.push({ type: e[0] === 'bb' ? 'BACKGROUND_BLUR' : 'LAYER_BLUR', radius: e[1], visible: true });
  }
  return out;
}
const errors = []; let made = 0;
function style(node, p) {
  if ('fills' in node && node.type !== 'TEXT' && node.type !== 'GROUP') node.fills = paints(p.k === 'F' ? p.bg : p.f);
  if (p.s && 'strokes' in node) {
    node.strokes = [{ type: 'SOLID', color: rgb(p.s.c), opacity: p.s.c[3] }]; node.strokeWeight = p.s.w;
    if (p.s.al) node.strokeAlign = p.s.al; else node.strokeAlign = 'CENTER';
    if (p.s.dash) node.dashPattern = p.s.dash;
    if (p.s.join) node.strokeJoin = p.s.join;
    if (p.s.cap) node.strokeCap = p.s.cap;
  }
  if (p.fx && 'effects' in node) {
    const fx = effects(p.fx);
    try { node.effects = fx; } catch (e) { try { node.effects = fx.map(x => x.type === 'DROP_SHADOW' ? x : Object.assign({ blurType: 'NORMAL' }, x)); } catch (e2) { errors.push('效果無法套用：' + p.n); } }
  }
  if (p.r && 'topLeftRadius' in node) { node.topLeftRadius = p.r[0]; node.topRightRadius = p.r[1]; node.bottomRightRadius = p.r[2]; node.bottomLeftRadius = p.r[3]; }
  if (p.op !== undefined) node.opacity = p.op;
  if (p.bm) { try { node.blendMode = p.bm; } catch (e) { } }
  if (p.hid) node.visible = false;
  node.name = p.n;
}
function place(node, m, ox, oy, base) {
  node.relativeTransform = [[m[0], m[2], m[4] + m[0] * ox + m[2] * oy + base], [m[1], m[3], m[5] + m[1] * ox + m[3] * oy + base]];
}
async function build(p, parent, base) {
  let node = null;
  try {
    if (p.k === 'G') {
      const kids = [];
      for (const c of p.ch) { const k = await build(c, parent, base); if (k) kids.push(k); }
      if (!kids.length) return null;
      node = figma.group(kids, parent);
      if (p.ch[0].msk && kids.length > 1) kids[0].isMask = true;
      style(node, p); made++;
      return node;
    }
    let ox = 0, oy = 0;
    if (p.k === 'F') {
      node = figma.createFrame(); parent.appendChild(node);
      node.resize(Math.max(p.w, 0.01), Math.max(p.h, 0.01)); node.clipsContent = !!p.clip;
      style(node, p); place(node, p.m, 0, 0, base);
      for (const c of p.ch || []) await build(c, node, 0);
      made++;
      return node;
    }
    if (p.k === 'R') { node = figma.createRectangle(); parent.appendChild(node); node.resize(Math.max(p.w, 0.01), Math.max(p.h, 0.01)); }
    else if (p.k === 'E') { node = figma.createEllipse(); parent.appendChild(node); node.resize(Math.max(p.w, 0.01), Math.max(p.h, 0.01)); }
    else if (p.k === 'V') { node = figma.createVector(); parent.appendChild(node); node.vectorPaths = [{ windingRule: p.wr ? 'EVENODD' : 'NONZERO', data: p.d }]; ox = node.x; oy = node.y; }
    else if (p.k === 'T') {
      node = figma.createText(); parent.appendChild(node);
      node.fontName = pick(p.runs[0][1], p.runs[0][2]); node.characters = p.tx;
      let pos = 0;
      for (const r of p.runs) {
        const a = pos, b = Math.min(pos + r[0], p.tx.length); pos = b; if (b <= a) continue;
        node.setRangeFontName(a, b, pick(r[1], r[2])); node.setRangeFontSize(a, b, r[3]);
        node.setRangeLetterSpacing(a, b, { unit: 'PERCENT', value: r[4] / 10 });
        node.setRangeFills(a, b, [{ type: 'SOLID', color: rgb(r[5]), opacity: r[5][3] }]);
        const fl = r[6] || 0;
        if (fl & 1) node.setRangeTextDecoration(a, b, 'UNDERLINE'); else if (fl & 2) node.setRangeTextDecoration(a, b, 'STRIKETHROUGH');
        if (fl & 4) node.setRangeTextCase(a, b, 'UPPER'); else if (fl & 8) node.setRangeTextCase(a, b, 'LOWER'); else if (fl & 16) node.setRangeTextCase(a, b, 'TITLE');
      }
      if (p.lh) node.lineHeight = { unit: 'PIXELS', value: p.lh };
      node.textAlignHorizontal = { L: 'LEFT', C: 'CENTER', R: 'RIGHT' }[p.al];
      const size = p.runs[0][3];
      if (p.ar === 'P') {
        node.textAutoResize = 'WIDTH_AND_HEIGHT';
        const lh = p.lh || node.height / p.tx.split(String.fromCharCode(10)).length;
        ox = -({ L: 0, C: 0.5, R: 1 }[p.al]) * node.width; oy = -(lh / 2 + p.kk * size);
      } else {
        if (p.ar === 'A' && p.h) { node.resize(Math.max(p.w, 1), Math.max(p.h, 1)); node.textAutoResize = 'NONE'; }
        else { node.resize(Math.max(p.w, 1), node.height); node.textAutoResize = 'HEIGHT'; }
        const lh = p.lh || size * 1.2;
        if (p.fy !== undefined) oy = p.fy - (lh / 2 + p.kk * size);
      }
    } else return null;
    style(node, p); place(node, p.m, ox, oy, base); made++;
    return node;
  } catch (e) {
    if (errors.length < 20) errors.push((p.n || p.k) + '：' + String(e && e.message ? e.message : e));
    if (node && p.k !== 'G') { try { node.remove(); } catch (e2) { } }
    return null;
  }
}
// ---- 區段與根節點 ----
const secName = 'XD 匯入：' + plan.name;
const isHolder = c => c.name === '__xd_chunks__' || ('children' in c && c.children.some(k => k.name === '__xd_chunks__'));
let sec = page.children.find(n => n.type === 'SECTION' && n.name === secName);
if (!sec) {
  let maxX = -Infinity;
  for (const c of page.children) if (!isHolder(c)) maxX = Math.max(maxX, c.x + c.width);
  sec = figma.createSection(); sec.name = secName;
  sec.x = isFinite(maxX) ? maxX + 400 : 0; sec.y = 0;
  sec.resizeWithoutConstraints(plan.W + PAD * 2, plan.H + PAD * 2);
}
const built = [];
for (let i = 0; i < plan.roots.length; i++) {
  if (ONLY && !ONLY.includes(i)) continue;
  const n = await build(plan.roots[i], sec, PAD);
  if (n) built.push({ index: i, id: n.id, name: n.name });
}
let cleaned = 0;
if (CLEANUP) for (const h of holders) { const par = h.parent; h.remove(); cleaned++; if (par && par.type === 'FRAME' && par.children.length === 0) par.remove(); }
return { createdNodeIds: [sec.id].concat(built.map(b => b.id)), sectionId: sec.id, roots: plan.roots.map((r, i) => i + '=' + r.n), built, nodes: made, fontSubs, missingImages, errors, cleaned };
```

### 讀取程式（xd2figma.py）

```python
#!/usr/bin/env python3
"""xd2figma: 讀取 Adobe XD 的 .xd 檔，產生讓 Figma 建構腳本使用的資料包。

用法:
  python3 xd2figma.py <檔案.xd> <輸出資料夾> [--no-images]

輸出:
  plan.json            可讀的建構計畫（除錯用）
  parts/part_NN.txt    直接模式：每行一個 [檢查碼,'資料'] 項目，整個檔案貼進「載入腳本」的 CH 陣列
  <檔名>_匯入用.svg     檔案模式：把這個 SVG 拖進 Figma（或上傳），資料就在圖層名稱裡
"""
import base64, copy, json, math, os, re, sys, zipfile, zlib

IDENT = [1, 0, 0, 1, 0, 0]
CHUNK = 4000
PART_CHARS = 38000
NL = chr(10)
# 基線係數 k：基線 = 行框頂端 + 行高/2 + k*字級
FONT_K = {"Noto Sans TC": 0.38, "Inter": 0.3636, "Verdana": 0.3977, "Tahoma": 0.3965}
L10N = {"SHAPE_RECTANGLE": "Rectangle", "SHAPE_ELLIPSE": "Ellipse", "SHAPE_PATH": "Path", "SHAPE_LINE": "Line",
        "SHAPE_GROUP": "Group", "SHAPE_MASK": "Mask Group", "SHAPE_POLYGON": "Polygon", "SHAPE_REPEAT_GRID": "Repeat Grid"}


def m_mul(m1, m2):
    a1, b1, c1, d1, x1, y1 = m1
    a2, b2, c2, d2, x2, y2 = m2
    return [a1 * a2 + c1 * b2, b1 * a2 + d1 * b2, a1 * c2 + c1 * d2, b1 * c2 + d1 * d2,
            a1 * x2 + c1 * y2 + x1, b1 * x2 + d1 * y2 + y1]


def tr(x, y):
    return [1, 0, 0, 1, x, y]


def tf(n):
    t = n.get("transform")
    if not t:
        return IDENT
    return [t.get("a", 1), t.get("b", 0), t.get("c", 0), t.get("d", 1), t.get("tx", 0), t.get("ty", 0)]


def is_translation(m):
    return abs(m[0] - 1) < 1e-6 and abs(m[1]) < 1e-6 and abs(m[2]) < 1e-6 and abs(m[3] - 1) < 1e-6


def rnd(v, n=3):
    v = round(float(v), n)
    return int(v) if v == int(v) else v


def rm(m):
    return [rnd(m[0], 5), rnd(m[1], 5), rnd(m[2], 5), rnd(m[3], 5), rnd(m[4]), rnd(m[5])]


# ---------- 路徑正規化：只輸出絕對座標的 M L C Q Z ----------
TOKEN = re.compile("[MmLlHhVvCcSsQqTtAaZz]|[-+]?(?:[0-9]*[.][0-9]+|[0-9]+[.]?)(?:[eE][-+]?[0-9]+)?")


def arc_to_cubics(x1, y1, rx, ry, phi, fa, fs, x2, y2):
    if rx == 0 or ry == 0:
        return [("L", [x2, y2])]
    rx, ry = abs(rx), abs(ry)
    sp, cp = math.sin(phi), math.cos(phi)
    dx, dy = (x1 - x2) / 2, (y1 - y2) / 2
    x1p, y1p = cp * dx + sp * dy, -sp * dx + cp * dy
    lam = x1p * x1p / (rx * rx) + y1p * y1p / (ry * ry)
    if lam > 1:
        rx, ry = rx * math.sqrt(lam), ry * math.sqrt(lam)
    num = rx * rx * ry * ry - rx * rx * y1p * y1p - ry * ry * x1p * x1p
    den = rx * rx * y1p * y1p + ry * ry * x1p * x1p
    co = math.sqrt(max(0.0, num / den)) if den else 0.0
    if fa == fs:
        co = -co
    cxp, cyp = co * rx * y1p / ry, -co * ry * x1p / rx
    cx, cy = cp * cxp - sp * cyp + (x1 + x2) / 2, sp * cxp + cp * cyp + (y1 + y2) / 2
    ang = lambda ux, uy, vx, vy: math.atan2(ux * vy - uy * vx, ux * vx + uy * vy)
    t1 = ang(1, 0, (x1p - cxp) / rx, (y1p - cyp) / ry)
    dt = ang((x1p - cxp) / rx, (y1p - cyp) / ry, (-x1p - cxp) / rx, (-y1p - cyp) / ry)
    if not fs and dt > 0:
        dt -= 2 * math.pi
    elif fs and dt < 0:
        dt += 2 * math.pi
    segs = max(1, int(math.ceil(abs(dt) / (math.pi / 2) - 1e-9)))
    d = dt / segs
    k = 4.0 / 3.0 * math.tan(d / 4)
    out = []
    pt = lambda t: (cx + rx * math.cos(t) * cp - ry * math.sin(t) * sp, cy + rx * math.cos(t) * sp + ry * math.sin(t) * cp)
    dv = lambda t: (-rx * math.sin(t) * cp - ry * math.cos(t) * sp, -rx * math.sin(t) * sp + ry * math.cos(t) * cp)
    for i in range(segs):
        a, b = t1 + i * d, t1 + (i + 1) * d
        p0, p3 = pt(a), pt(b)
        d0, d3 = dv(a), dv(b)
        out.append(("C", [p0[0] + k * d0[0], p0[1] + k * d0[1], p3[0] - k * d3[0], p3[1] - k * d3[1], p3[0], p3[1]]))
    return out


def fnum(v):
    s = ("%.3f" % v).rstrip("0").rstrip(".")
    return "0" if s in ("-0", "") else s


def norm_path(d):
    toks = TOKEN.findall(d or "")
    out, i, cmd = [], 0, None
    cx = cy = sx = sy = 0.0
    pcx = pcy = None
    pkind = ""

    def num():
        nonlocal i
        v = float(toks[i])
        i += 1
        return v

    while i < len(toks):
        t = toks[i]
        if t.isalpha():
            cmd = t
            i += 1
            if cmd in "Zz":
                out.append(("Z", []))
                cx, cy = sx, sy
                pkind = ""
            continue
        if cmd is None or cmd in "Zz":
            break
        rel = cmd.islower()
        C = cmd.upper()
        ox, oy = (cx, cy) if rel else (0.0, 0.0)
        if C == "M":
            x, y = num() + ox, num() + oy
            out.append(("M", [x, y]))
            cx, cy, sx, sy = x, y, x, y
            cmd = "l" if rel else "L"
            pkind = ""
        elif C == "L":
            x, y = num() + ox, num() + oy
            out.append(("L", [x, y]))
            cx, cy, pkind = x, y, ""
        elif C == "H":
            x = num() + ox
            out.append(("L", [x, cy]))
            cx, pkind = x, ""
        elif C == "V":
            y = num() + oy
            out.append(("L", [cx, y]))
            cy, pkind = y, ""
        elif C == "C":
            v = [num() + ox, num() + oy, num() + ox, num() + oy, num() + ox, num() + oy]
            out.append(("C", v))
            pcx, pcy, cx, cy, pkind = v[2], v[3], v[4], v[5], "C"
        elif C == "S":
            x1, y1 = (2 * cx - pcx, 2 * cy - pcy) if pkind == "C" else (cx, cy)
            v = [x1, y1, num() + ox, num() + oy, num() + ox, num() + oy]
            out.append(("C", v))
            pcx, pcy, cx, cy, pkind = v[2], v[3], v[4], v[5], "C"
        elif C == "Q":
            v = [num() + ox, num() + oy, num() + ox, num() + oy]
            out.append(("Q", v))
            pcx, pcy, cx, cy, pkind = v[0], v[1], v[2], v[3], "Q"
        elif C == "T":
            x1, y1 = (2 * cx - pcx, 2 * cy - pcy) if pkind == "Q" else (cx, cy)
            v = [x1, y1, num() + ox, num() + oy]
            out.append(("Q", v))
            pcx, pcy, cx, cy, pkind = x1, y1, v[2], v[3], "Q"
        elif C == "A":
            rx, ry, rot, fa, fs = num(), num(), num(), num(), num()
            x, y = num() + ox, num() + oy
            out += arc_to_cubics(cx, cy, rx, ry, math.radians(rot), bool(fa), bool(fs), x, y)
            cx, cy, pkind = x, y, ""
        else:
            break
    return " ".join(" ".join([c] + [fnum(x) for x in v]) for c, v in out)


# ---------- 讀取 XD ----------
class Reader:
    def __init__(self, path, with_images=True):
        self.z = zipfile.ZipFile(path)
        self.names = set(self.z.namelist())
        self.man = json.loads(self.z.read("manifest"))
        self.res = json.loads(self.z.read("resources/graphics/graphicContent.agc"))
        rr = self.res.get("resources") or {}
        self.gradients = rr.get("gradients") or {}
        self.symbols = {}
        for s in (((rr.get("meta") or {}).get("ux") or {}).get("symbols") or []):
            self._index_symbol(s)
        self.with_images = with_images
        self.images = {}
        self.fonts = set()
        self.warn = []
        self.stats = {}

    def count(self, k):
        self.stats[k] = self.stats.get(k, 0) + 1

    def _kids(self, n):
        t = n.get("type")
        c = (n.get(t) or {}).get("children") if isinstance(n.get(t), dict) else None
        return c if isinstance(c, list) else []

    def _index_symbol(self, n):
        if not isinstance(n, dict):
            return
        if n.get("id"):
            self.symbols.setdefault(n["id"], n)
        for st in (((n.get("meta") or {}).get("ux") or {}).get("states") or []):
            self._index_symbol(st)
        for c in self._kids(n):
            self._index_symbol(c)

    def _merge(self, src, ovr, top=True):
        if isinstance(src, dict) and isinstance(ovr, dict):
            out = copy.deepcopy(src)
            for k, v in ovr.items():
                if top and k in ("type", "syncSourceGuid", "guid"):
                    continue
                if k in ("children", "group", "shape") and k in out:
                    out[k] = self._merge(out[k], v, False)
                else:
                    out[k] = copy.deepcopy(v)
            return out
        if isinstance(src, list) and isinstance(ovr, list):
            return [self._merge(src[i], ovr[i], False) if i < len(src) else copy.deepcopy(ovr[i]) for i in range(len(ovr))] + copy.deepcopy(src[len(ovr):])
        return copy.deepcopy(ovr)

    def expand(self, n):
        """展開同步參照（元件實體內只記錄差異的節點）"""
        g = n.get("syncSourceGuid")
        if g:
            src = self.symbols.get(g)
            if src is None:
                self.warn.append("找不到同步來源，已略過一個圖層")
                return None
            n = self._merge(src, n)
            n["id"] = n.get("guid") or n.get("id")
            self.count("同步參照（已展開）")
        return n

    # ----- 樣式 -----
    def color(self, c, mul=1.0):
        if not c:
            return [0, 0, 0, 1]
        v = c.get("value") or {}
        if c.get("mode") not in (None, "RGB"):
            self.warn.append("非 RGB 色彩模式 %s，顏色可能不正確" % c.get("mode"))
        return [int(round(v.get("r", 0))), int(round(v.get("g", 0))), int(round(v.get("b", 0))), rnd(c.get("alpha", 1) * mul)]

    def fills(self, fill):
        if not fill or fill.get("type") in (None, "none"):
            return []
        t = fill["type"]
        if t == "solid":
            return [["s"] + self.color(fill.get("color"))]
        if t == "gradient":
            g = fill.get("gradient") or {}
            rs = self.gradients.get(g.get("ref")) or (((g.get("meta") or {}).get("ux") or {}).get("gradientResources")) or {}
            stops = [[rnd(s.get("offset", 0), 4)] + self.color(s.get("color")) for s in rs.get("stops", [])]
            if len(stops) < 2:
                self.warn.append("漸層缺少色標，已改用單色")
                return [["s"] + (stops[0][1:] if stops else [128, 128, 128, 1])]
            self.count("漸層")
            if rs.get("type") == "radial":
                return [["g", "r", rnd(g.get("cx", 0.5), 4), rnd(g.get("cy", 0.5), 4), rnd(g.get("r", 0.5), 4), 0, stops]]
            return [["g", "l", rnd(g.get("x1", 0), 4), rnd(g.get("y1", 0), 4), rnd(g.get("x2", 1), 4), rnd(g.get("y2", 0), 4), stops]]
        if t == "pattern":
            ux = ((fill.get("pattern") or {}).get("meta") or {}).get("ux") or {}
            uid = ux.get("uid")
            if uid and ("resources/" + uid) in self.names:
                if uid not in self.images:
                    self.images[uid] = self.z.read("resources/" + uid)
                self.count("圖片填色")
                # XD：cover = 等比例填滿並裁切；fill = 拉伸填滿
                return [["i", uid, "FILL" if ux.get("scaleBehavior") == "cover" else "STRETCH"]]
            self.warn.append("找不到圖片資源，已改用灰色")
            return [["s", 204, 204, 204, 1]]
        self.warn.append("未支援的填色類型 %s" % t)
        return []

    def stroke(self, s):
        if not s or s.get("type") in (None, "none") or not s.get("width"):
            return None
        o = {"c": self.color(s.get("color")), "w": rnd(s.get("width", 1))}
        al = {"inside": "INSIDE", "outside": "OUTSIDE"}.get(s.get("align"))
        if al:
            o["al"] = al
        if s.get("dash"):
            o["dash"] = [rnd(x) for x in s["dash"]]
        if s.get("join") in ("round", "bevel"):
            o["join"] = s["join"].upper()
        if s.get("cap") in ("round", "square"):
            o["cap"] = s["cap"].upper()
        return o

    def effects(self, st):
        out, fill_op = [], None
        for f in st.get("filters") or []:
            if f.get("visible") is False:
                continue
            p = f.get("params") or {}
            if f.get("type") == "dropShadow":
                for d in p.get("dropShadows") or []:
                    out.append(["d", rnd(d.get("dx", 0)), rnd(d.get("dy", 0)), rnd(d.get("r", 0)), self.color(d.get("color"))])
                    self.count("陰影")
            elif f.get("type") == "uxdesign#blur":
                if p.get("backgroundEffect"):
                    out.append(["bb", rnd(p.get("blurAmount", 0))])
                    fill_op = p.get("fillOpacity")
                else:
                    out.append(["b", rnd(p.get("blurAmount", 0))])
                self.count("模糊")
            else:
                self.warn.append("未支援的效果 %s" % f.get("type"))
        return out, fill_op

    def common(self, node, n, st):
        if st.get("opacity") is not None and st["opacity"] < 0.999:
            node["op"] = rnd(st["opacity"])
        if n.get("visible") is False:
            node["hid"] = 1
        bm = st.get("blendMode")
        if bm and bm not in ("normal", "pass-through"):
            node["bm"] = bm.upper().replace("-", "_")
        return node

    def name(self, n, fallback):
        nm = n.get("name")
        if nm:
            return nm
        l = ((n.get("meta") or {}).get("ux") or {}).get("nameL10N")
        return L10N.get(l, fallback)

    # ----- 節點 -----
    def conv(self, n, cur, mask=False):
        n = self.expand(n)
        if n is None:
            return None
        t = n.get("type")
        st = n.get("style") or {}
        M = m_mul(cur, tf(n))
        if t == "shape":
            node = self.shape(n, M, st, mask)
        elif t == "text":
            node = self.text(n, M, st)
        elif t == "group":
            node = self.group(n, M, st)
        else:
            self.warn.append("未支援的圖層類型 %s，已略過" % t)
            return None
        if node is None:
            return None
        return self.common(node, n, st)

    def shape(self, n, M, st, mask=False):
        sh = n.get("shape") or {}
        ty = sh.get("type")
        if ty == "rect":
            node = {"k": "R", "n": self.name(n, "Rectangle"), "m": rm(m_mul(M, tr(sh.get("x", 0), sh.get("y", 0)))),
                    "w": rnd(sh.get("width", 0)), "h": rnd(sh.get("height", 0))}
            r = sh.get("r")
            if r and any(r):
                node["r"] = [rnd(x) for x in (list(r) + [r[-1]] * 4)[:4]]
            self.count("矩形")
        elif ty in ("circle", "ellipse"):
            rx = sh.get("r", sh.get("rx", 0))
            ry = sh.get("r", sh.get("ry", 0))
            node = {"k": "E", "n": self.name(n, "Ellipse"), "m": rm(m_mul(M, tr(sh.get("cx", 0) - rx, sh.get("cy", 0) - ry))),
                    "w": rnd(rx * 2), "h": rnd(ry * 2)}
            self.count("橢圓")
        else:
            if ty == "line":
                d = "M %s %s L %s %s" % (sh.get("x1", 0), sh.get("y1", 0), sh.get("x2", 0), sh.get("y2", 0))
            elif ty == "polygon" and not sh.get("path"):
                pts = sh.get("points") or []
                d = " ".join(("M" if i == 0 else "L") + " %s %s" % (p.get("x", 0), p.get("y", 0)) for i, p in enumerate(pts)) + " Z"
            else:
                d = sh.get("path") or ""
            d = norm_path(d)
            if not d:
                self.count("略過（空路徑）")
                return None
            node = {"k": "V", "n": self.name(n, "Path"), "m": rm(M), "d": d}
            if sh.get("winding") == "evenodd":
                node["wr"] = 1
            self.count("路徑")
        fx, fill_op = self.effects(st)
        f = [["s", 0, 0, 0, 1]] if mask else self.fills(st.get("fill"))
        if fill_op is not None and f and f[0][0] == "s":
            f[0][4] = rnd(fill_op)
        if f:
            node["f"] = f
        s = None if mask else self.stroke(st.get("stroke"))
        if s:
            node["s"] = s
        if fx and not mask:
            node["fx"] = fx
        if not f and not s:
            self.count("無填色無邊框的形狀")
        return node

    def text(self, n, M, st):
        tx = n.get("text") or {}
        raw = tx.get("rawText") or ""
        if not raw:
            self.count("略過（空文字）")
            return None
        font = st.get("font") or {}
        ta = st.get("textAttributes") or {}
        base_col = self.color((st.get("fill") or {}).get("color")) if (st.get("fill") or {}).get("type") == "solid" else [0, 0, 0, 1]
        fam, sty, size = font.get("family", "Inter"), font.get("style", "Regular"), font.get("size", 16)
        ls0 = ta.get("letterSpacing", 0)
        runs, total = [], 0
        for r in (((n.get("meta") or {}).get("ux") or {}).get("rangedStyles") or []):
            ln = int(r.get("length", 0))
            if ln <= 0:
                continue
            col = base_col
            v = (r.get("fill") or {}).get("value")
            if isinstance(v, (int, float)):
                v = int(v)
                col = [(v >> 16) & 255, (v >> 8) & 255, v & 255, rnd(((v >> 24) & 255) / 255.0)]
            run = [ln, r.get("fontFamily", fam), r.get("fontStyle", sty), r.get("fontSize", size), rnd(r.get("charSpacing", ls0)), col]
            flags = (1 if r.get("underline") else 0) | (2 if r.get("strikethrough") else 0) | ({"uppercase": 4, "lowercase": 8, "titlecase": 16}.get(r.get("textTransform"), 0))
            if flags:
                run.append(flags)
            runs.append(run)
            total += ln
        if not runs:
            runs = [[len(raw), fam, sty, size, rnd(ls0), base_col]]
        elif total < len(raw):
            runs[-1][0] += len(raw) - total
        for r in runs:
            self.fonts.add((r[1], r[2]))
        fr = tx.get("frame") or {}
        ftype = fr.get("type", "positioned")
        node = {"k": "T", "n": self.name(n, raw.replace(NL, " ")[:40]), "m": rm(M), "tx": raw, "runs": runs,
                "ar": {"positioned": "P", "area": "A", "autoHeight": "H"}.get(ftype, "P"),
                "al": {"center": "C", "right": "R"}.get(ta.get("paragraphAlign"), "L"), "kk": FONT_K.get(runs[0][1], 0.36)}
        if ftype != "positioned":
            node["w"] = rnd(fr.get("width", 100))
            node["h"] = rnd(fr.get("height", 0))
            try:
                node["fy"] = rnd(tx["paragraphs"][0]["lines"][0][0].get("y", 0))
            except (KeyError, IndexError, TypeError):
                pass
        if ta.get("lineHeight"):
            node["lh"] = rnd(ta["lineHeight"])
        self.count({"P": "點文字", "A": "區域文字", "H": "區域文字"}[node["ar"]])
        return node

    def group(self, n, M, st):
        ux = (n.get("meta") or {}).get("ux") or {}
        kids_src = (n.get("group") or {}).get("children") or []
        clip = (ux.get("clipPathResources") or {}).get("children") or []
        if len(clip) > 1:
            self.warn.append("遮罩含多個形狀，只使用第一個（%s）" % n.get("name", ""))
        cs = clip[0] if clip else None
        if cs is not None and (cs.get("shape") or {}).get("type") == "rect" and is_translation(M) and is_translation(tf(cs)):
            sh, ct = cs["shape"], tf(cs)
            ox, oy = ct[4] + sh.get("x", 0), ct[5] + sh.get("y", 0)
            kids = [k for k in (self.conv(c, tr(-ox, -oy)) for c in kids_src) if k]
            if not kids:
                self.count("略過（空群組）")
                return None
            node = {"k": "F", "n": self.name(n, "Mask Group"), "m": rm(m_mul(M, tr(ox, oy))), "w": rnd(sh.get("width", 0)), "h": rnd(sh.get("height", 0)),
                    "clip": 1, "ch": kids}
            r = sh.get("r")
            if r and any(r):
                node["r"] = [rnd(x) for x in (list(r) + [r[-1]] * 4)[:4]]
            self.count("遮罩群組→裁切畫框")
            return node
        kids = []
        if cs is not None:
            mk = self.conv(cs, M, mask=True)
            if mk:
                mk["msk"] = 1
                kids.append(mk)
                self.count("遮罩群組→遮罩")
        kids += [k for k in (self.conv(c, M) for c in kids_src) if k]
        if not kids or (cs is not None and len(kids) == 1 and kids[0].get("msk")):
            self.count("略過（空群組）")
            return None
        self.count("群組")
        return {"k": "G", "n": self.name(n, "Group"), "ch": kids}

    # ----- 文件 -----
    def plan(self, title):
        artwork = [c for c in self.man.get("children", []) if c.get("path") == "artwork"]
        entries = artwork[0].get("children", []) if artwork else []
        boards, paste = [], []
        for e in entries:
            p = "artwork/%s/graphics/graphicContent.agc" % e.get("path")
            if p not in self.names:
                continue
            agc = json.loads(self.z.read(p))
            if str(e.get("path", "")).startswith("artboard-"):
                for a in agc.get("children", []):
                    if a.get("type") != "artboard":
                        continue
                    ab = a.get("artboard") or {}
                    info = (self.res.get("artboards") or {}).get(ab.get("ref")) or {}
                    b = e.get("uxdesign#bounds") or {}
                    boards.append({"name": info.get("name") or e.get("name") or "Artboard", "x": info.get("x", b.get("x", 0)), "y": info.get("y", b.get("y", 0)),
                                   "w": info.get("width", b.get("width", 100)), "h": info.get("height", b.get("height", 100)),
                                   "style": a.get("style") or {}, "children": ab.get("children") or []})
            else:
                paste += agc.get("children", [])
        xs = [b["x"] for b in boards] + [tf(n)[4] for n in paste]
        ys = [b["y"] for b in boards] + [tf(n)[5] for n in paste]
        minx, miny = (min(xs), min(ys)) if xs else (0, 0)
        roots = []
        for b in sorted(boards, key=lambda b: (b["x"], b["y"])):
            kids = [k for k in (self.conv(c, tr(-b["x"], -b["y"])) for c in b["children"]) if k]
            node = {"k": "F", "ab": 1, "n": b["name"], "m": rm(tr(b["x"] - minx, b["y"] - miny)), "w": rnd(b["w"]), "h": rnd(b["h"]), "clip": 1, "ch": kids}
            bg = self.fills(b["style"].get("fill"))
            if bg:
                node["bg"] = bg
            roots.append(node)
            self.count("畫板")
        for n in paste:
            k = self.conv(n, tr(-minx, -miny))
            if k:
                roots.append(k)
                self.count("工作區上的物件")
        maxx = max([b["x"] + b["w"] for b in boards] + [minx + 100]) - minx
        maxy = max([b["y"] + b["h"] for b in boards] + [miny + 100]) - miny
        return {"v": 1, "name": title, "W": rnd(maxx), "H": rnd(maxy), "fonts": sorted([list(f) for f in self.fonts]), "roots": roots}


def checksum(text):
    """與建構腳本相同的檢查碼，用來確認圖片資料在傳輸後沒有損壞"""
    a, b = 1, 0
    for ch in text:
        a = (a + ord(ch)) % 65521
        b = (b + a) % 65521
    return b * 65536 + a


def chunks(key, data_b64):
    return ["%s|%d|%05d:%s" % (key, len(data_b64), i // CHUNK, data_b64[i:i + CHUNK]) for i in range(0, len(data_b64), CHUNK)]


def main():
    args = [a for a in sys.argv[1:] if not a.startswith("--")]
    if len(args) != 2:
        raise SystemExit("用法: xd2figma.py <檔案.xd> <輸出資料夾> [--no-images]")
    src, out = args
    with_images = "--no-images" not in sys.argv
    title = os.path.splitext(os.path.basename(src))[0]
    rd = Reader(src, with_images)
    plan = rd.plan(title)
    img_b64 = {uid: base64.b64encode(data).decode() for uid, data in rd.images.items()}
    plan["imgs"] = {uid: checksum(b) for uid, b in img_b64.items()}
    os.makedirs(os.path.join(out, "parts"), exist_ok=True)
    for old in os.listdir(os.path.join(out, "parts")):
        os.remove(os.path.join(out, "parts", old))
    raw = json.dumps(plan, ensure_ascii=False, separators=(",", ":")).encode("utf-8")
    json.dump(plan, open(os.path.join(out, "plan.json"), "w", encoding="utf-8"), ensure_ascii=False)
    co = zlib.compressobj(9, zlib.DEFLATED, -15)
    packed = co.compress(raw) + co.flush()
    lines = chunks("plan", base64.b64encode(packed).decode())
    img_lines = []
    if with_images:
        for uid, b in img_b64.items():
            img_lines += chunks("img_" + uid, b)
    all_lines = lines + img_lines
    # 直接模式：分成多個檔案，每個檔案的總字元數不超過 PART_CHARS
    parts, cur, size = [], [], 0
    for ln in all_lines:
        if cur and size + len(ln) > PART_CHARS:
            parts.append(cur)
            cur, size = [], 0
        cur.append(ln)
        size += len(ln)
    if cur:
        parts.append(cur)
    for i, p in enumerate(parts):
        # 每行是一個 [檢查碼, '資料'] 項目，可直接貼進載入腳本的 CH 陣列
        open(os.path.join(out, "parts", "part_%02d.txt" % (i + 1)), "w", encoding="utf-8").write(NL.join("[%d,'%s']," % (checksum(ln), ln) for ln in p) + NL)
    # 檔案模式：資料放在 SVG 矩形的 id（匯入 Figma 後成為圖層名稱）
    cols = 40
    rows = (len(all_lines) + cols - 1) // cols
    W, H = max(5000, cols * 120), max(5000, rows * 120)
    svg = ['<svg xmlns="http://www.w3.org/2000/svg" width="%d" height="%d" viewBox="0 0 %d %d">' % (W, H, W, H), '<g id="__xd_chunks__">']
    for i, ln in enumerate(all_lines):
        svg.append('<rect id="%s" x="%d" y="%d" width="100" height="100" fill="#888888"/>' % (ln, (i % cols) * 120, (i // cols) * 120))
    svg += ["</g>", "</svg>"]
    svg_path = os.path.join(out, title + "_匯入用.svg")
    open(svg_path, "w", encoding="utf-8").write(NL.join(svg))
    total = sum(len(l) for l in all_lines)
    print("計畫大小: JSON %d bytes → 壓縮後 %d bytes" % (len(raw), len(packed)))
    print("根節點: " + "、".join("%d=%s" % (i, r["n"]) for i, r in enumerate(plan["roots"])))
    print("統計:", json.dumps(rd.stats, ensure_ascii=False))
    print("字型:", "、".join("%s %s" % (a, b) for a, b in sorted(rd.fonts)))
    print("圖片: %d 張，共 %d bytes%s" % (len(rd.images), sum(len(v) for v in rd.images.values()), "" if with_images else "（未打包）"))
    print("資料總量: %d 字元，直接模式需 %d 次載入；檔案模式用 %s (%d bytes)" % (total, len(parts), os.path.basename(svg_path), os.path.getsize(svg_path)))
    for w in sorted(set(rd.warn)):
        print("警告:", w)


if __name__ == "__main__":
    main()
```