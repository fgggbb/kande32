# Kanade 2.3 — 老舊 32 位晶片支援版

[Kanade](https://github.com/rcmiku/Kanade) 是一款 Jetpack Compose 架構的 Android 音樂播放器。  
原 APK 已包含 32-bit（`armeabi-v7a`）原生庫，但在 Windows 上進行魔解壓縮時因**大小寫檔案覆蓋**導致資源損壞，使老舊 32 位晶片（如 NVIDIA Tegra K1）無法安裝。  
本倉庫修復此問題，讓 Kanade 可在老舊 32 位裝置上正常安裝執行。

**原始 APK：** `Kanade_2.3.apk`（AGP 9.2.1, compileSdk 37, minSdk 26）  
**修復後簽名 APK：** `Kanade_2.3_signed.apk`（需自行以 `jarsigner` / `apksigner` 簽名）

---

## 目錄結構

```
├─ AndroidManifest.xml   # 修改 extractNativeLibs: false → true
├─ classes.dex            # 已編譯的 Kotlin/Java 字節碼
├─ resources.arsc         # 已編譯的資源表
├─ res/                   # 混淆短名資源（AndResGuard）
├─ lib/                   # 原生庫（4 個 ABI）
│  ├─ arm64-v8a/          # 64-bit ARM
│  ├─ armeabi-v7a/        # 32-bit ARM
│  ├─ x86_64/             # 64-bit x86
│  └─ x86/                # 32-bit x86
├─ assets/                # 資產（compose 資源、DexOpt profile）
├─ META-INF/              # 依賴模塊元數據
├─ kotlin/                # Kotlin builtins（編譯產物）
├─ google/                # Protobuf 定義（編譯產物）
└─ src/                   # Protobuf 源碼（編譯產物）
```

---

## 修復內容

### 1. Windows case-insensitive 檔案系統導致資源損壞

原始 APK 內有四組**僅大小寫不同**的資源檔案：

| 檔案對 | 實際大小 | 問題 |
|--------|----------|------|
| `res/VC.xml` / `res/Vc.xml` | 1800 B / 1144 B | Windows 視為同檔名，互相覆蓋 |
| `res/I8.xml` / `res/i8.xml` | 928 B / 1024 B | 同上 |
| `res/Fd.xml` | 1088 B（正確）→ 624 B（被截斷）| 內容被覆蓋損毀 |
| `res/fw.xml` | 652 B（正確）→ 1376 B（錯誤）| 內容錯亂 |
| `res/is.xml` | 1428 B（正確）→ 1232 B | 被其他文件覆蓋 |
| `res/un.xml` | 792 B（正確）→ 800 B | 內容偏移 |

**影響：** 資源解析失敗 → APK 在老舊 32 位裝置（如小米 Pad 1 / NVIDIA Tegra K1）顯示「解析套件出錯」  
**修復：** 直接從原始 APK zip 條目重建，繞過 Windows 檔案系統。

### 2. extractNativeLibs 改為 true

`android:extractNativeLibs` 從 `false` 改為 `true`，確保原生庫在安裝時被正確提取至檔案系統。

### 3. 圖標修復

損壞的 drawable 資源（`VC.xml`、`Vc.xml`、`I8.xml`、`i8.xml` 等）已從原始 APK 還原為正確內容。

---

## 適用裝置

| ABI | 位元 | 支援 |
|-----|------|------|
| `arm64-v8a` | 64-bit ARM | ✓ |
| `armeabi-v7a` | 32-bit ARM | ✓ **（重點：老舊 32 位晶片如 NVIDIA Tegra K1 / 小米 Pad 1）** |
| `x86_64` | 64-bit x86 | ✓ |
| `x86` | 32-bit x86 | ✓ |



---

## 打包與簽名

本倉庫目錄為純檔案結構，打包需重建為 zip **並確保大小寫敏感條目完整**：

### 方式一：使用本倉庫腳本（已內建於 build 流程）

```bash
# 重建 unsigned APK（保留大小寫差異的條目）
python3 final_build.py

# 使用 debug 憑證簽名
jarsigner -sigalg SHA256withRSA -digestalg SHA-256 \
  -keystore debug.keystore \
  -storepass android -keypass android \
  -signedjar Kanade_2.3_signed.apk unsigned.apk androiddebugkey
```

### 方式二：直接使用原始 APK

`Kanade_2.3.apk` 為原始未簽名 APK，可直接修改其 `AndroidManifest.xml` 後簽名：

```bash
# 解包
unzip -o Kanade_2.3.apk -d tmp/
# 修改 AndroidManifest.xml（二進制 AXML 需專用工具編輯）
# 重新打包並簽名
```

> **注意：** 打包時必須使用程式化方式（如 Python zipfile）確保 zip 內大小寫相異的檔案條目（`VC.xml` / `Vc.xml` 等）同時保留，否則 Android 資源解析器在部分裝置上會報錯。

### Windows 使用者注意事項

因 Windows 檔案系統不區分大小寫（case-insensitive），以下兩對檔案在 clone 到 Windows 時**無法同時存在**於磁碟上：

- `res/VC.xml` (1800 B) ↔ `res/Vc.xml` (1144 B)
- `res/I8.xml` (928 B) ↔ `res/i8.xml` (1024 B)

在 Linux / macOS 上 clone 則無此問題。若需在 Windows 上重建 APK，請使用 `Kanade_2.3.apk` + 上方的 Python 腳本（直接從原始 zip 條目讀取，繞過檔案系統）。

---

## 免責聲明

- 本倉庫僅為技術研究用途
- 應用程式原始版權歸 [rcmiku](https://github.com/rcmiku) 所有
- debug 憑證簽名僅供測試安裝，正式發布請使用正式 keystore
