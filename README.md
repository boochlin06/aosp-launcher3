<h1 align="center">AOSP Launcher3</h1>

<p align="center">
  <a href="https://android-arsenal.com/api?level=21"><img alt="API" src="https://img.shields.io/badge/API-21%2B-brightgreen.svg?style=flat"/></a>
  <a href="https://opensource.org/licenses/Apache-2.0"><img alt="License" src="https://img.shields.io/badge/License-Apache%202.0-blue.svg"/></a>
</p>

The official Android Open Source Project (AOSP) Launcher3 source code. As the default home screen for the Android operating system, this repository showcases the highest standard of Android system-level UI development, touch event dispatching, and extreme memory optimization.

## 🚀 Features

- **Extreme UI Fluidity**: Custom `PagedView` and `CellLayout` implementations designed specifically for maintaining 60/120 FPS during complex drag-and-drop operations.
- **Global Drag and Drop**: A robust `DragController` that manages shadow views (`DragView`) floating above the entire UI hierarchy.
- **Widget Hosting**: Fully implements the Android `AppWidgetHost` to embed third-party application widgets.
- **Dynamic Device Profiling**: Replaces static `dp` dimensions with `InvariantDeviceProfile`, dynamically calculating icon margins and padding based on physical screen pixels at runtime.

## 🛠 Tech Stack & Architecture

- **State Pattern**: UI transitions (e.g., Spring-loaded workspace, App Drawer, Widget Picker) are cleanly separated using the State Design Pattern.
- **Observer / Callback Mechanism**: 
  - `LauncherModel` acts as the single source of truth, observing `PackageManager` broadcasts (install/uninstall).
  - Data is processed asynchronously and posted to the UI thread via `LauncherModel.Callbacks`.
- **System Privileges**: Declares restricted permissions like `BIND_APPWIDGET` and `FORCE_STOP_PACKAGES`, requiring system signature for full functionality.

## 📦 Getting Started

### Prerequisites
- Android Studio / IntelliJ IDEA
- Or AOSP build environment (`make Launcher3`)

### Build & Run
```bash
git clone https://github.com/boochlin06/aosp-launcher3.git
```
Import via Gradle or build directly using the provided `Android.mk` inside an AOSP tree.

## 📄 License
Licensed under the Apache License 2.0.
