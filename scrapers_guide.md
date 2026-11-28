# 房源爬蟲運作機制與除錯指南（Scrapers Technical Guide）

本文件依資料管線執行順序，記錄各資料來源（1. 內政部實價登錄、2. 591、3. 樂居）的實際抓取原理、實測驗證結論（包含已淘汰的舊端點與反爬蟲陷阱），以及當未來網站改版時的快速診斷與樣本更新指令。

---

## 1. 內政部實價登錄 Open Data（`src/scrapers/lvr_opendata.py`）

### 1.1 實測驗證結論（Testing & Discovery Summary）
1. **Session Cookie + Referer 驗證機制**：
   * 直接請求 `/Download` 或 `/DownloadSeason` 會被政府伺服器導向「系統訊息」HTML 錯誤頁（導致 `BadZipFile`）。
   * 必須先透過同一個 `httpx.AsyncClient` 請求 `https://moi.gov.tw` 取得 `JSESSIONID` Cookie，並帶上 `Referer: https://moi.gov.tw` 與瀏覽器 `User-Agent`，才能下載到真實 ZIP（標頭 `PK\x03\x04`）。
2. **CSV 雙表頭與真實欄位名稱**：
   * 每個 CSV 第 1 列為中文欄名、第 2 列為英文說明（`The villages and towns urban district...`），解析時必須過濾英文說明列。
   * **車位面積正確欄名**：真實欄位為 `車位移轉總面積平方公尺`（非 `車位總面積平方公尺`），需同時扣除 `車位總價元` 與 `車位移轉總面積平方公尺` 才不會高估單價。
   * **交易標的過濾陷阱**：住宅案件的 `交易標的` 為 `房地(土地+建物)` 或 `房地(土地+建物)+車位`，過濾純土地/純車位時必須使用完整錨點 `^土地$|^車位$`，不可寫成無錨點的 `土地|車位$`。
3. **成屋（`_a`）與預售屋（`_b`）雙軌分流**：
   * `{code}_lvr_land_a.csv`（成屋買賣）存為 `tx_category="existing"`；地址常出現重複縣市前綴（如 `新竹市新竹市南大路...`），`extract_road_name()` 需使用 `while` 迴圈剝除前綴。
   * `{code}_lvr_land_b.csv`（預售屋買賣）存為 `tx_category="presale"`，且內含 `建案名稱` 欄位，可直接建立新案社區基準均價。
4. **歷史季檔回溯（`DownloadSeason`）**：
   * 季檔格式為 `https://moi.gov.tw`，透過官方唯一 `編號`（`serial_no`）去重寫入 `lvr_transactions` 後重算滾動均價與漲跌趨勢（`trend_pct`）。

### 1.2 未來改版時的快速診斷與樣本重抓指令
```bash
# 1. 執行內政部 Schema 漂移即時整合測試與離線單元測試
python -m pytest -m "integration" -k "lvr" -v
python -m pytest -m "not integration" -k "lvr" -v

# 2. 一鍵重新下載最新真實 _a (成屋) 與 _b (預售屋) 黃金樣本至 tests/fixtures/
python scripts/refresh_fixtures.py

# 3. 依 profiles/ 中啟用計劃的縣市，回溯近 3 年（預設，可改 --years 5）歷史季檔並重算滾動均價與趨勢
python scripts/backfill_lvr.py --years 3
```

---

## 2. 591 房屋交易網（`src/scrapers/source_591.py`）

### 2.1 實測驗證結論（Testing & Discovery Summary）
開發過程中已針對 591 進行實網探測，確認以下關鍵技術事實（未來除錯可直接參考，無需重複踩雷）：
1. **舊版 JSON API 已淘汰**：傳統 `/home/search/rsList` 與 `/list?region=...` 皆已回傳 `404`。591 目前採用 **Nuxt 3 SSR** 架構，列表與內頁資訊直接渲染於 HTML 中。
2. **縣市切換必須靠 `urlJumpIp` Cookie**：
   * 僅傳遞網址參數 `?shType=list&regionid=4` 仍會回傳預設「台北市（`regionid=1`）」物件。
   * 必須在 `httpx.AsyncClient` 設定 `client.cookies.set("urlJumpIp", str(region_id), domain=".591.com.tw")`（新竹市 `4`、新竹縣 `5`、苗栗縣 `7`），才能取得該縣市真實列表。
3. **精準座標陷阱（雙 iframe 機制）**：
   * 物件詳情頁同時存在兩個 Google Maps iframe：
     * `#detail-map-free`（隱藏元素，固定為台北市假座標 `25.0564617, 121.5281307`）——**絕對不可使用**。
     * `#detail-map-free-new`（`src="...&q={lat},{lng}"`）——**這才是該房屋的真實 GPS 座標**。
4. **Httpx Cookie 最佳實踐**：新版 `httpx` 已棄用單次請求傳入 `cookies={...}`（會觸發 `DeprecationWarning`），統一改用 `client.cookies.set(...)`。

### 2.2 兩階段抓取流程（Two-Stage SSR Pipeline）
1. **第一階段（列表頁 `parse_591_list_html`）**：請求 `https://591.com.tw{region_id}&order=posttime_desc`，解析 `<a href="/home/house/detail/2/{id}.html">` 卡片，先取得標題、行政區（立即過濾非目標鄉鎮市區）、總價、坪數、單價與房廳衛格局。
2. **第二階段（詳情頁 `enrich_591_listing_from_detail_html`）**：針對符合目標行政區的物件（搭配 `0.8~1.8` 秒隨機延遲），進入內頁解析 `#detail-map-free-new` 座標、`公設比`、`車位類型`、`建物型態`、`樓層/總樓層`、`屋齡`、`路段名稱`，以及 `法定用途`（揪出工業宅/一般事務所陷阱）與 `屋況特色`。

### 2.3 未來改版時的快速診斷與樣本重抓指令（Reusable Diagnostic）
1. **執行整合測試與離線單元測試**：
   ```bash
   python -m pytest -m "integration" -k "591" -v
   python -m pytest -m "not integration" -k "591" -v
   ```
2. **一鍵實網探測並更新離線黃金樣本（Golden Fixtures）**：
   ```bash
   python -c '
   import re, httpx
   from bs4 import BeautifulSoup
   from pathlib import Path

   host = ".".join(["sale", "591", "com", "tw"])
   headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"}

   with httpx.Client(headers=headers, timeout=30.0, follow_redirects=True) as c:
       c.cookies.set("urlJumpIp", "4", domain=".591.com.tw")
       r = c.get(f"https://{host}/", params={"shType": "list", "regionid": "4", "order": "posttime_desc"})
       soup = BeautifulSoup(r.text, "html.parser")
       links = [a["href"] for a in soup.select("a[href*=\"/home/house/detail/2/\"]")]
       print("List status:", r.status_code, "Detail links found:", len(links))
       if links:
           Path("tests/fixtures/sample_591_list.html").write_text(r.text, encoding="utf-8")
           first_link = next(iter(links))
           first_url = f"https://{host}{first_link}" if first_link.startswith("/") else first_link
           d = c.get(first_url)
           Path("tests/fixtures/sample_591_detail.html").write_text(d.text, encoding="utf-8")
           coord = re.search(r"id=\"detail-map-free-new\"[^>]*src=\"[^\"]*q=([0-9.-]+),([0-9.-]+)", d.text)
           print("Refreshed fixtures! Detail URL:", first_url, "Coords:", coord.groups() if coord else None)
   '
   ```

---

## 3. 樂居 LEJU（`src/scrapers/source_leju.py`）

### 3.1 實測驗證結論（Cloudflare Turnstile 限制與雙軌解法）
1. **雲端 IP 驗證機制**：樂居網站（`://leju.com.tw`）受 Cloudflare Turnstile 保護，從雲端資料中心 IP（如 GitHub Actions / Codespaces）直接請求會回傳 `403` 或「請稍候...」驗證頁。
2. **雙軌解決方案（免費中繼匯入 + 優雅降級）**：
   * **本地住宅 IP 中繼匯入（推薦，$0 成本）**：可於台灣家中電腦執行 `python scripts/collect_leju_local.py`，將抓到的物件存為 `data/incoming/leju_latest.json` 並推送至 Git；雲端排程執行 `SourceLejuScraper` 時會自動讀取 `data/incoming/leju_*.json` 併入主流程（同樣經過三層去重與實價比對）。
   * **選用代理或 Cookie**：亦可透過環境變數設定 `LEJU_PROXY_URL`（台灣住宅代理）或 `LEJU_CF_CLEARANCE`。若雲端直接請求仍遭阻擋，`SourceLejuScraper` 會安全跳過並回傳 `[]`，不影響其他資料來源。

---

## 4. 樂屋網 Rakuya（`src/scrapers/source_rakuya.py`）

### 4.1 實測驗證結論（Testing & Discovery Summary）
1. **雲端 IP 直連穩定**：樂屋網（`://rakuya.com.tw/sell/result?city={id}&sort=11`）整合信義、永慶、住商、中信、東森等多家仲介物件，且直接回傳完整 SSR HTML（`200 OK`，無 Cloudflare Turnstile 阻擋），適合作為 GitHub Actions 雲端主力房源之一。
2. **縣市代碼對照（注意與 591 不同）**：
   * `city=4` 為桃園市、`city=5` 為新竹市、`city=6` 為新竹縣、`city=7` 為苗栗縣（依台灣由北到南順序編號）。
3. **列表頁即含完整物理規格**：列表卡片直接提供 `社區名稱`、`建物型態`、`屋齡`、`樓層/總樓層`、`總建坪` 與 `主建坪`，爬蟲可直接計算 `public_ratio_pct`（公設比），無需逐筆請求內頁。

### 4.2 未來改版時的快速診斷與樣本重抓指令
```bash
# 1. 執行樂屋網即時整合測試與離線單元測試
python -m pytest -m "integration" -k "rakuya" -v
python -m pytest -m "not integration" -k "rakuya" -v

# 2. 一鍵重新抓取新竹市最新樂屋網列表樣本至 tests/fixtures/sample_rakuya_list.html
python -c "import httpx; from pathlib import Path; r = httpx.get("https://" + ".".join(["www","rakuya","com","tw"]) + "/sell/result", params={"city":"5","sort":"11"}, headers={"User-Agent":"Mozilla/5.0"}, follow_redirects=True); Path("tests/fixtures/sample_rakuya_list.html").write_text(r.text, encoding="utf-8"); print("Saved:", r.status_code, len(r.text))"
```

---

## 5. 信義房屋 Sinyi Realty（`src/scrapers/source_sinyi.py`）

### 5.1 實測驗證結論（Testing & Discovery Summary）
1. **雲端 IP 直連穩定**：信義房屋（`://sinyi.com.tw/buy/list/{City-slug}/default-desc/1`）首頁列表直接回傳完整 SSR HTML（`200 OK`，單頁約 18 筆直營物件，無 Cloudflare Turnstile 阻擋），適合作為 GitHub Actions 雲端主力房源之一。
2. **URL 縣市代碼對照**：採用英文縣市 slug（如 `Hsinchu-city`、`Hsinchu-county`、`Miaoli-county`），由 `CITY_CODE_TO_SINYI_SLUG` 自動轉換。
3. **列表卡片欄位結構**：直接包含 `總價`、`物件標題`、`路名地址`（透過 `extract_road_name` 萃取路段）、`社區名稱`、`建物型態`、`建坪`、`屋齡` 與 `房廳衛格局`。

### 5.2 未來改版時的快速診斷與樣本重抓指令
```bash
# 1. 執行信義房屋即時整合測試與離線單元測試
python -m pytest -m "integration" -k "sinyi" -v
python -m pytest -m "not integration" -k "sinyi" -v

# 2. 一鍵重新抓取新竹市最新信義房屋列表樣本至 tests/fixtures/sample_sinyi_list.html
python -c "import httpx; from pathlib import Path; r = httpx.get("https://" + ".".join(["www","sinyi","com","tw"]) + "/buy/list/Hsinchu-city/default-desc/1", headers={"User-Agent":"Mozilla/5.0"}, follow_redirects=True); Path("tests/fixtures/sample_sinyi_list.html").write_text(r.text, encoding="utf-8"); print("Saved:", r.status_code, len(r.text))"
```

## 自適應分頁介面契約（`known_ids` & `max_pages`）
- 所有繼承 `BaseScraper` 的爬蟲皆支援 `fetch_listings(city_targets, known_ids=None, max_pages=1)`。
- 当 `known_ids` 非空且單頁已看過物件比例達 `>= 80%` 時，爬蟲立即提前停止翻頁（禮貌延遲 1 秒），降低來源網站負載。
