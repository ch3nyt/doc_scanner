# Doc Scanner

**線上使用 Try it：https://ch3nyt.github.io/doc_scanner/**

把手機拍的文件拉正成掃描檔，或把照片原樣轉成 PDF；照片和既有 PDF 可以混著放，依你排的順序合併成一份 PDF。
**所有處理都在你的瀏覽器裡完成，檔案不會上傳到任何伺服器。**

Flatten phone photos of documents into scan-like pages, or convert photos to PDF as they are. Mix in existing PDFs, arrange the order, and export one merged PDF. **Everything runs in your browser; no file is uploaded anywhere.**

整個工具是單一檔案 `index.html`，沒有後端、不需要安裝。

---

## 使用方式

### 開啟

任選一種：

1. **線上版**：<https://ch3nyt.github.io/doc_scanner/>，手機、電腦都能用。
2. **本機**：下載 `index.html`，用瀏覽器（Chrome、Edge、Safari、Firefox）直接開啟即可。

第一次開啟會看到兩頁標示「範例」的霍格華茲成績單照片（虛構內容），用來示範操作；選擇自己的檔案後範例會自動清除。

### 操作流程

1. **加入檔案**：點「選擇檔案」或頁面列的「＋」，可一次多選照片和 PDF；電腦上也可以直接拖進編輯區。
2. **逐張處理照片**，每張二選一：
   - **拉正**：拖曳四個藍點（TL 左上、TR 右上、BR 右下、BL 左下）對準紙張四角，按「拉正，下一頁」。拖曳時會出現放大鏡，方便在手機上對準。
   - **不調整**：照片不裁切、不套效果，原樣放進 PDF。照片很多時可用「其餘全部不調整」一次處理完。
3. **調整順序**：拖曳下方頁面列的縮圖（手機請長按再拖），或用「← 前移／後移 →」。
4. **匯出 PDF**：右側「匯出 PDF」，依頁面列順序合併。

### 其他功能

| 功能 | 說明 |
|---|---|
| 旋轉 ↺ ↻ | 照片方向不對時轉 90°，已擺好的四角會跟著轉 |
| 紙張 | A4 直式／橫式（300 dpi，2480 × 3508 px）、Letter，或依四角自動 |
| 效果 | 彩色掃描、灰階掃描（兩者都會去除陰影與不均勻光線）、黑白、原色 |
| PDF 輸入 | 原始頁面直接複製進合併檔，不轉成圖片，文字保持清晰可選取 |
| 下載單頁 | 已校正的頁面可單獨下載 JPG |
| 刪除 | 刪錯可在提示中按「復原」 |

---

## 限制與注意事項

- **有密碼保護的 PDF 無法合併。**
- **含數位簽章的 PDF（例如學校核發的電子成績單），合併後簽章會失效。** 工具偵測到簽章時會提醒你。需要讓審查端驗證真偽的正式文件，請單獨繳交原檔。
- 四角目前需要手動對準，沒有自動偵測紙張邊緣。
- 照片過大時會先縮小到約 1,600 萬像素以內（iOS Safari 的 canvas 上限），對 300 dpi A4 輸出沒有影響。
- 拉正與編碼在背景執行緒（Web Worker）進行：按下「拉正，下一頁」會立刻跳到下一張，前一張在背景處理，縮圖顯示「處理中」；匯出時會等所有頁面處理完。不支援 OffscreenCanvas 的舊瀏覽器會改在主執行緒處理，期間頁面會短暫停住。

## 隱私與安全

- 照片和 PDF 只在你的瀏覽器記憶體中處理，關掉分頁就消失；本工具沒有伺服器，**你的檔案內容永遠不會被傳送到任何地方**。
- **匿名使用統計**：線上版使用 [GoatCounter](https://www.goatcounter.com/) 計算瀏覽次數，以及「匯出 PDF」「下載單頁 JPG」的次數。不使用 cookie，不記錄個人資料或 IP；GoatCounter 只會收到頁面網址、來源網站、瀏覽器類型、螢幕尺寸與大致國家。在本機直接開啟 `index.html` 時不會計數。
- 頁面會向外部載入三項資源：
  - **Google Fonts**（IBM Plex Sans／Mono、Noto Sans TC）：Google 會看到你的 IP 與瀏覽器資訊，但不會收到你的檔案。離線時會退回系統字型，功能不受影響。
  - **pdf-lib 1.17.1**（cdnjs）與 **GoatCounter count.v4.js**：皆以 [Subresource Integrity](https://developer.mozilla.org/docs/Web/Security/Subresource_Integrity) 雜湊鎖定版本，CDN 上的檔案若被竄改，瀏覽器會拒絕執行。
- 想完全離線使用，可以把 pdf-lib 下載到本機並改成相對路徑；沒有 pdf-lib 時，照片轉 PDF 仍可運作（內建簡易 PDF 產生器），只有「PDF 輸入」功能無法使用。

---

## 運作原理

1. **透視校正**：手機斜拍讓紙張在照片裡變成四邊形，旋轉、縮放這類仿射變換無法把它拉回長方形。本工具由使用者標出的四個角點與目標長方形的四個角，解出 3×3 單應性矩陣（homography，8 個未知數的線性方程組，以高斯消去法求解），再對輸出影像的每個像素反推回原圖座標並做雙線性插值。這和 OpenCV 的 `getPerspectiveTransform` + `warpPerspective` 是同一套數學。
2. **去陰影的掃描效果**：紙張的背景亮度變化很平緩，所以先把影像縮小約 20 倍，用積分影像（integral image）做方框模糊估出背景，再在原尺寸逐像素以雙線性內插取回背景值、把原圖除以背景，抹平光線不均，最後拉高對比讓紙變白、字變黑。縮小、除法、對比調整都在同一輪像素迴圈內完成。
3. **PDF 輸出**：每頁照片以 JPEG 嵌入，頁寬固定為 A4（595.28 pt），頁高依影像比例；有 PDF 輸入時改用 pdf-lib 複製原始頁面。

---

## 致謝與相關專案

**本工具的程式碼為從零撰寫，沒有複製或改寫以下任何專案的程式碼。** 開發過程使用了 Anthropic 的 Claude Code 協助撰寫。

**直接使用的第三方資源**

| 資源 | 用途 | 授權 |
|---|---|---|
| [pdf-lib](https://github.com/Hopding/pdf-lib) by Andrew Dillon | 讀取與合併 PDF | MIT |
| [IBM Plex](https://github.com/IBM/plex) | 介面字型 | SIL Open Font License 1.1 |
| [Noto Sans TC](https://fonts.google.com/noto/specimen/Noto+Sans+TC) | 中文字型 | SIL Open Font License 1.1 |
| [GoatCounter](https://github.com/arp242/goatcounter) | 匿名使用統計 | EUPL 1.2 |

**技術參考**

- [Automatic Document Scanner using OpenCV — LearnOpenCV](https://learnopencv.com/automatic-document-scanner-using-opencv/)：四點透視轉換的文件掃描流程
- [OpenCV: Geometric Image Transformations](https://docs.opencv.org/4.x/da/d54/group__imgproc__transform.html)：`getPerspectiveTransform`／`warpPerspective` 的定義

**做類似事情的專案**（想要自動偵測邊緣或 Python 版本，可以參考）

- [gaurapanasenko/arkush](https://github.com/gaurapanasenko/arkush)：網頁版，自動偵測四角並可手動微調
- [AVI20953/doc-scan](https://github.com/AVI20953/doc-scan)：Python + OpenCV 命令列工具
- [ArashNasrEsfahani/Python-Document-Scanner-OpenCV](https://github.com/ArashNasrEsfahani/Python-Document-Scanner-OpenCV)：Python + OpenCV，含影像增強

---

## 授權

範例頁中的霍格華茲學院、課程名稱等出自《哈利波特》系列，相關權利屬 J.K. Rowling 與 Warner Bros.；本專案為非商業開源工具，與其無任何關聯。


[MIT](LICENSE) © 2026 Yan-Ting Chen（陳彥廷）

歡迎回報問題或送 Pull Request。
