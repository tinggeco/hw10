# Week 10 Homework：ARIA v7.0 — All-Weather Auditor

## 專案簡介

本作業為「遙測與空間資訊之分析與應用」Week 10 作業，主題為 **ARIA v7.0 — The All-Weather Auditor**。本研究以花蓮馬太鞍溪流域為案例，分析 2025 年鳳凰颱風後可能發生的淹水與堰塞湖溢流影響。

本次作業延續 Week 8–Week 9 的 ARIA 工作流程，使用 Microsoft Planetary Computer STAC API 串流遙測資料，不需要手動下載衛星影像。主要改進是加入 Sentinel-1 SAR 資料，使系統在颱風期間光學影像受雲遮蔽時，仍能透過雷達影像進行淹水偵測。

---

## 研究區與事件背景

研究區位於花蓮縣馬太鞍溪流域，涵蓋萬榮鄉、光復鄉與鳳林鎮部分區域。2025 年 11 月鳳凰颱風後，馬太鞍溪上游因崩塌形成堰塞湖，後續溢流可能導致下游地區淹水。本作業透過多源遙測資料融合，評估可能受災範圍。

研究區 BBOX：

```python
HUALIEN_BBOX = [121.2574, 23.6546, 121.4984, 23.7447]
```

時間範圍：

```python
PRE_DATE_RANGE = "2025-10-01/2025-11-05"
POST_DATE_RANGE = "2025-11-12/2025-11-30"
```

---

## 使用資料

本作業使用以下資料來源，皆透過 Planetary Computer STAC API 取得：

1. **Sentinel-1 RTC**
   - 用於 SAR 全天候淹水偵測
   - 使用 VV band
   - 災前與災後影像需使用相同軌道方向

2. **Sentinel-2 L2A**
   - 用於光學 NDWI 水體偵測
   - 使用 Green、NIR 與 SCL band
   - SCL 用於建立 cloud mask

3. **Copernicus DEM GLO-30**
   - 用於地形分析與坡度計算
   - 協助排除山區陡坡上的 SAR false positives

---

## 方法流程

### Task 1：SAR 全天候淹水偵測

本部分使用 Sentinel-1 RTC VV band 進行災後水體／淹水候選區偵測。

主要步驟包括：

1. 搜尋災前與災後 Sentinel-1 RTC 影像
2. 選擇相同 orbit direction 的災前與災後場景
3. 串流 VV band 並轉換為 dB
4. 套用 5×5 median filter 降低 speckle noise
5. 使用 SAR threshold 萃取低回波水體候選區
6. 使用 morphological opening 與 connected component filtering 清理雜訊
7. 計算水體候選區面積與平均後向散射值

本研究使用的 SAR threshold 為：

```python
SAR_THRESHOLD = -15.0
```

選擇此值是因為它能保留主要河道與低回波水體區，同時避免過於嚴格的 -18 dB 閾值漏掉颱風後濁水、淺水或較粗糙水面造成的水體訊號。

---

### Task 2：Sensor Fusion 多源融合確信度圖

本部分整合 Sentinel-1 SAR 水體遮罩、Sentinel-2 NDWI 水體遮罩與 Sentinel-2 SCL cloud mask，建立四分類融合確信度圖。

分類邏輯如下：

| 類別 | 條件 | 意義 |
|---|---|---|
| High Confidence | SAR 與 NDWI 都偵測到水，且非雲遮罩區 | 雙感測器一致，可信度最高 |
| SAR Only (Cloudy) | SAR 偵測到水，但光學受雲或不可靠像元影響 | SAR 補足雲下觀測能力 |
| Optical Only | NDWI 偵測到水，但 SAR 未偵測到 | 需人工檢核，可能為濕砂、淺水或光學誤判 |
| No Detection | SAR 與 NDWI 皆未偵測到水 | 無明顯水體證據 |

本研究使用的 NDWI threshold 為：

```python
NDWI_THRESHOLD = 0.0
```

選擇 0.0 的原因是颱風後河水與淹水區可能含有大量泥砂，若使用清水常用的 0.3 閾值，可能會漏判濁水。敏感度測試顯示，當 NDWI threshold 從 0.0 提高至 0.1 時，NDWI 水體面積由 20.5245 km² 急遽下降至 0.4883 km²；若使用 0.3，幾乎無法偵測到水體。因此本研究使用 0.0 以保留濁水河道與可能淹水區。

---

### Task 3：DEM 與坡度地形審計

SAR 影像在山區容易受到雷達陰影、疊置效應與前縮效應影響，導致陡坡上的低回波區被誤判為水體。因此本作業使用 Copernicus DEM 計算坡度，並套用地形過濾規則：

```python
SLOPE_THRESHOLD = 25
```

坡度大於 25° 的水體候選像元被視為可能的地形誤判，並從融合結果中移除。

本研究地形過濾共移除：

```text
False positives removed by topographic filter = 5.1595 km²
```

這些被移除的區域多位於研究區西側山區，較可能與 SAR 陰影、疊置效應、前縮效應或地形造成的低回波有關，而非真正穩定積水。

若 DEM 不適用，例如上游崩塌與堰塞湖區域災後地形已快速改變，則需搭配 morphological opening、connected component filtering 與人工目視檢核來輔助清理假水體。

---

### Task 4：AI Strategic Briefing 與 ARIA v7.0 報告

本部分將 ARIA v7.0 的融合結果整理為 key metrics，並輸入 LLM 產生災害應變策略簡報。

主要指標如下：

| 指標 | 數值 |
|---|---:|
| High confidence flood area | 2.9784 km² |
| SAR-only cloudy flood area | 1.1229 km² |
| False positives removed by topographic filter | 5.1595 km² |
| Cloud cover percentage within study area | 7.46% |
| SAR threshold | -15.0 dB |
| NDWI threshold | 0.0 |

AI strategic briefing 主要討論：

1. 哪些區域應優先撤離
2. 高信心區與 SAR-only 區域之間如何分配救災資源
3. 目前評估的不確定性與限制
4. 需要哪些額外資料提升判釋信心

---

## Week 9 與 Week 10 比較

Week 9 的 ARIA v6.0 為 optical-only 分析，主要依賴光學影像與 NDWI。Week 10 的 ARIA v7.0 則整合 SAR、NDWI、cloud mask 與 DEM slope filter。

| Metric | W9 Optical Only | W10 Fused | 說明 |
|---|---:|---:|---|
| Total detected flood / water candidate area | 66.659 km² | 16.4879 km² | W10 經過融合與地形過濾後更保守 |
| Cloud-covered area analyzed | 0 km² | 1.1229 km² | SAR 可補足雲下觀測能力 |
| False positives removed by topographic filter | Not applied | 5.1595 km² | W10 加入地形誤判審計 |
| Confidence levels | 3-zone | 4-class | W10 分級更細緻 |

W10 的總偵測面積比 W9 小，但這不代表偵測能力變差，而是因為 W10 加入 SAR 證據、雲遮罩、融合邏輯與 DEM 地形審計後，能排除較多不合理的誤判區。ARIA v7.0 的主要進步在於提升判釋可信度、增加雲下觀測能力，並提供更細緻的信心分級。

---

## 結論

本作業成功建立 ARIA v7.0 多源遙測淹水分析流程。相較於 Week 9 的 optical-only 方法，ARIA v7.0 透過 Sentinel-1 SAR 增加雲下觀測能力，並透過 Sentinel-2 NDWI、cloud mask 與 DEM slope filter 提升判釋可靠度。結果顯示，高信心淹水／水體區為 2.9784 km²，SAR-only cloudy 區域為 1.1229 km²，且地形過濾移除了 5.1595 km² 的疑似山區 false positives。整體而言，ARIA v7.0 不只是增加資料來源，也提供了更完整的信心分級與成果驗證流程。
