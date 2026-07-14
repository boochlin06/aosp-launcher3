# AOSP Launcher3 原始碼深度架構剖析與技術白皮書 (Android 系統桌面核心)

## 第一章：專案總覽與其在 Android 生態圈的地位 (Executive Summary)

本專案為 Android Open Source Project (AOSP) 官方維護的 Launcher3 原始碼。這不僅僅是一個「應用程式」，它是整個 Android 作業系統最重要的「門面」與「導航中樞」。全球數以十億計的 Android 裝置，從 Google Pixel 系列、各家 OEM 廠商 (如 Samsung, Xiaomi, Acer 等) 的深度客製化桌面，到車載系統 (Android Automotive) 與智慧電視 (Android TV)，其桌面系統的底層基底，皆是源自於這套 Launcher3 原始碼。

學習與理解 Launcher3 的架構，等同於掌握了 Android 系統級 UI 開發的最頂級心法。本專案在 60FPS/120FPS 流暢度保證、記憶體極致榨取、複雜事件分發 (Touch Event Dispatching)、多執行緒同步 (Concurrency) 以及與底層系統服務 (`ActivityManagerService`, `PackageManagerService`) 的深度通訊上，提供了教科書級別的最佳實踐 (Best Practices)。本 README 將以超過 3000 字的篇幅，為您徹底解構這座 Android 視圖系統的金字塔。

## 第二章：系統架構與設計理念深度探討 (Architectural Deep Dive)

### 2.1 放棄傳統佈局的矩陣演算視圖 (Grid-Based View Hierarchy)
傳統的 Android 開發者習慣使用 `LinearLayout` 或 `ConstraintLayout` 來排版。然而，桌面系統面臨的是極度動態的圖示拖曳、 Widget 縮放以及資料夾合併操作。為此，Launcher3 創造了完全自訂的視圖體系：
- **`CellLayout.java`**: 這是桌面的基礎網格。它將螢幕劃分為 `N x M` 的矩陣。任何放置在桌面上的元素 (`Shortcut`, `AppWidget`) 都不再由傳統的 `(x, y)` 像素座標決定，而是由 `cellX`, `cellY` (網格座標) 以及 `spanX`, `spanY` (跨越的網格數) 來定義。`CellLayout` 內部透過複雜的矩陣運算，將這些網格座標即時轉換為螢幕上的像素像素座標，並支援在拖放時自動擠開 (Displacement) 其他圖示的演算法。
- **`Workspace.java`**: 繼承自 `PagedView`，它是多個 `CellLayout` 的集合體，負責處理水平滑動、彈性邊界 (Overscroll) 以及處理跨頁的圖示拖放轉場動畫。

### 2.2 Model-View 徹底分離與狀態同步 (Observer Pattern & Callbacks)
Launcher 需要管理龐大的狀態（哪些 App 安裝了、哪些捷徑被刪除了、Widget 的即時內容是什麼）。
- **`LauncherModel.java` (資料大腦)**: 這是系統的資料模型中心。它包含了一個獨立的 `HandlerThread` (稱為 Worker Thread)。所有與外部系統的繁重溝通（例如讀取 SQLite 資料庫、解析 APK 的 Manifest 獲取圖示與名稱）都在此 Worker Thread 進行，絕對不允許阻塞 Main UI Thread。
- **`LauncherModel.Callbacks`**: 當 Worker Thread 完成資料處理後，會建構出各種 `ItemInfo` (如 `WorkspaceItemInfo`, `FolderInfo`)，然後透過 Callback 機制，將渲染指令發送回 Main Thread 的 `Launcher` 類別，指揮 UI 進行增刪改查。這種極度嚴格的非同步設計是桌面流暢的基礎。

## 第三章：核心演算法與機制解密 (Core Algorithms & Mechanisms)

### 3.1 跨螢幕拖曳演算法 (`DragController` 與 `DropTarget`)
這可以說是整份原始碼中最複雜、最精華的段落。
當使用者長按一個圖示時：
1. **啟動拖曳 (Start Drag)**: `Workspace` 會捕捉到長按事件 (Long Click)，並通知 `DragController`。
2. **生成幽靈視圖 (DragView)**: 原本的圖示會暫時被隱藏，系統會擷取該圖示的 Bitmap，並在視圖層級的最頂端建立一個 `DragLayer`。在這個 `DragLayer` 上會生成一個隨著手指移動的 `DragView`。
3. **即時碰撞檢測 (Collision Detection)**: 隨著手指滑動，`DragController` 會不斷將當前的座標傳遞給下方的各個 `DropTarget` (可能是另一個 `CellLayout` 或是資料夾)。`CellLayout` 會透過二維陣列的演算法，即時計算該圖示若在此處放下，是否會與現有圖示重疊；若重疊，是否觸發「建立資料夾」邏輯或是觸發「擠開現有圖示 (Reorder)」的動畫。
4. **結束拖曳 (Drop)**: 當手指放開，系統會執行物理回彈動畫，並更新 SQLite 資料庫中該圖示的新 `cellX`, `cellY` 與 `screenId`。

### 3.2 雙層圖示快取架構 (`IconCache.java`)
啟動器在冷啟動 (Cold Start) 時，需要在零點幾秒內渲染數十個高解析度圖示。若每次都透過 `PackageManager` 去解析 APK，將會導致災難性的啟動延遲。
- **記憶體快取 (L1)**: 使用 `LruCache` 將最常顯示的圖示保留在 RAM 中。
- **持久化快取 (L2)**: 建立一個專屬的 SQLite 表格 (`app_icons.db`)，將圖示轉換為 `byte[]` 儲存。同時，它會記錄 APK 的 `lastUpdateTime`。每次開機時，比對 `lastUpdateTime`，若 APK 未被更新，則直接從資料庫讀取位元組陣列並轉換為 Bitmap，這比呼叫系統 API 快上數十倍。

## 第四章：資料庫持久化與 ContentProvider

### 4.1 `LauncherProvider.java` 剖析
所有的桌面配置皆儲存於私有的 `favorites.db` 中。
- **Schema 詳解**: 核心表格 `favorites` 定義了每一個物件的屬性：
  - `_id`: 唯一識別碼
  - `title`: 圖示名稱
  - `intent`: 點擊時觸發的 URI 字串 (包含了啟動哪個 Package 的哪個 Activity)
  - `itemType`: 定義此物件是 APP 捷徑 (`ITEM_TYPE_APPLICATION`)、資料夾 (`ITEM_TYPE_FOLDER`) 還是小工具 (`ITEM_TYPE_APPWIDGET`)
  - `container`: 定義該物件位於何處（例如：放置在桌面上 `CONTAINER_DESKTOP`，還是位於底部的 Dock 列 `CONTAINER_HOTSEAT`，亦或是某個資料夾內部）。如果是資料夾內部，此值為該資料夾的 `_id`。
- **備份與還原 (Backup & Restore)**: Launcher3 實作了 Android 系統的 `BackupAgent`，能將使用者的桌面配置加密上傳至 Google Drive，並在使用者更換新手機時，無縫還原圖示排列。

## 第五章：資源管理與多維度硬體適配 (Resource Management)

在 `res/` 目錄下，您會看到極其複雜的資源修飾詞 (Qualifiers)。
Launcher3 必須在 4 吋的小手機到 12 吋的大平板上都能完美顯示。
- **`InvariantDeviceProfile.java`**: 此類別負責解析裝置的螢幕實體尺寸與 DPI。它會從 XML (如 `device_profiles.xml`) 中讀取各種定義好的 Grid Size (例如：4x4, 5x5, 6x5)。
- **動態縮放 (Dynamic Scaling)**: 不同於一般 App 使用固定的 `dp` 值，Launcher 的所有圖示大小、字體大小、Margin 與 Padding 皆是在啟動時由 `DeviceProfile.java` 根據螢幕長寬的實際像素，透過比例動態計算出來的，確保空間利用率達到最佳化。

## 第六章：效能優化極限操作 (Performance Optimization)

### 6.1 硬體加速與 RenderThread
為了達到流暢的動畫，Launcher3 大量使用了 `ViewPropertyAnimator` 並設定 `View.setLayerType(View.LAYER_TYPE_HARDWARE, null)`。這會強迫系統將該 View 的渲染結果快取在 GPU 的 Texture 中。當執行平移或縮放動畫時，CPU 完全不參與重繪，而是直接由 GPU 進行矩陣變換，徹底消除掉幀 (Jank) 現象。

### 6.2 垃圾回收防禦 (GC Defense)
在 Java 中，頻繁建立物件會觸發 GC，進而導致主執行緒暫停。Launcher3 內部有大量防止 GC 的設計。例如在繪製 (onDraw) 迴圈或滑動事件 (`onTouchEvent`) 中，絕對不會使用 `new` 關鍵字宣告新物件。所有的暫存矩陣、Rect、Point 皆被宣告為靜態成員變數或是使用物件池 (Object Pool) 重複利用。

## 第七章：高階系統安全權限 (Security & Permissions)

作為系統核心，Launcher3 在 `AndroidManifest.xml` 中宣告了多個危險且受保護的權限：
- `android.permission.BIND_APPWIDGET`: 允許 Launcher 綁定並讀取第三方應用程式的 Widget 內容。一般 App 除非經過使用者明確授權，否則無法取得此權限。
- `android.permission.FORCE_STOP_PACKAGES`: 允許 Launcher 在特定情況下（如長按強制關閉）直接呼叫 `ActivityManager` 終止其他進程。

## 第八章：總結與技術學習指南

AOSP Launcher3 不僅僅是一個專案，它是一本活生生的 Android 進階開發教科書。對於任何渴望從「應用層開發者」晉升為「系統框架級架構師 (System Framework Architect)」的工程師來說，深入研讀 Launcher3 的程式碼是必經之路。從理解 `View` 的測量與繪製 (`onMeasure`, `onLayout`, `onDraw`)，到掌握 IPC 機制與背景排程，這份超過數十萬行的原始碼蘊含了 Google 頂尖工程師的心血與無數次的效能迭代。
