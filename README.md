# AOSP Launcher3 (Android 系統桌面)

這是 Android 開源計畫 (AOSP) 官方的 Launcher3 原始碼。作為 Android 系統的預設啟動器，它展示了最高標準的 Android 系統級 UI 開發與架構設計。

---

## 程式功能分析 (Program Feature Analysis)

1. **高效能視圖系統與排版機制**
   * 實作了客製化的 `CellLayout` 與 `PagedView`，捨棄了標準的 Android Layout，以矩陣 (Grid) 座標系來精確計算圖示與 Widget 的擺放位置。
2. **全域拖放框架 (Drag and Drop)**
   * 實作了極度複雜的 `DragController` 與 `DropTarget` 機制，支援跨螢幕的圖示移動、資料夾合併、以及刪除動畫。
3. **資料庫與 ContentProvider**
   * 透過 `LauncherProvider` 建立私有的 SQLite 資料庫，儲存使用者的桌面配置（包含 Icon 位置、Widget ID 等），並支援資料庫升級與還原。

---

## 檔案與目錄結構 (Directory Structure)

```text
Launcher3/
├── Android.mk & CleanSpec.mk        # Android 原始碼樹 (Source Tree) 的編譯腳本
├── build.gradle                     # 支援 Gradle 的編譯配置
├── src/                             # 系統桌面原始碼 (Launcher, Workspace, Folder 等)
├── res/                             # 包含多維度的資源配置 (sw600dp, sw720dp 等平板適配)
├── WallpaperPicker/                 # 負責桌布設定與裁切的子專案
├── tests/                           # 包含系統級別的 Instrumentation Tests
└── update_system_wallpaper_cropper.py # 輔助開發的 Python 腳本
```

---

## 使用到的 Design Pattern 與架構決策

1. **State Pattern (狀態模式)**
   * Launcher 擁有不同的狀態（例如：預設桌面、開啟應用程式抽屜、進入 Widget 選擇模式）。透過狀態模式管理，使 UI 轉場動畫與返回鍵邏輯能清晰分離。
2. **Thread-Safe Data Loading (執行緒安全的資料載入)**
   * **決策**：在 `LauncherModel` 中使用了專屬的 Background Handler Thread 來處理所有的資料庫 I/O 與 App 解析 (`PackageManager` 查詢)。
   * **實作**：所有的資料載入任務 (`LoaderTask`) 都會放入這個背景執行緒排隊，完成後再發送 Runnable 到 Main Thread 更新 UI，徹底避免 ANR (Application Not Responding)。

---

## 專案亮點介紹 (Highlights)

* 📏 **精密的記憶體與圖形管理**
  * 由於桌面常駐於系統背景，專案內對 `Bitmap` 的快取 (Cache) 與資源回收有著教科書等級的實作，極大程度減少了 Garbage Collection (GC) 引發的卡頓。
* 🛡 **系統級應用開發範本**
  * 這份程式碼是學習如何編寫高效能 Android UI、處理複雜 Touch Event (手勢攔截) 以及與 Android Framework 核心服務 (`ActivityManager`, `PackageManager`) 互動的最佳範例。
