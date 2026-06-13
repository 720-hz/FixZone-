# FixZone 🚗🔧

> AI-powered car damage detection and repair cost estimation — Android app built as a graduation project.

---

## Overview

FixZone is a Flutter Android application that uses a on-device AI model to automatically detect car damage from photos and estimate repair costs in Egyptian Pounds (EGP). The user photographs their vehicle, the app identifies damage types and severity in real time, and produces a detailed repair report with nearby workshop recommendations — all without an internet connection for the core detection pipeline.

---

## Features

- **On-device damage detection** — YOLO11 segmentation model running via TFLite, no server required
- **Six damage classes** — dent, scratch, crack, glass shatter, broken lamp, flat tire
- **Severity grading** — each detected region is classified as Minor, Moderate, or Severe based on its mask area relative to the image
- **EGP cost estimation** — every class maps to a price range per severity tier
- **PDF report generation** — exportable repair report with scanned images, detection results, and shop details
- **Workshop locator** — four Cairo-area repair shops with one-tap Google Maps navigation and direct phone call buttons
- **Arabic-friendly UI** — the app supports Arabic content throughout, with English-only fallback fields used where PDF rendering requires it

---

## Tech Stack

| Layer | Technology |
|---|---|
| Mobile framework | Flutter (Android) |
| On-device inference | flutter_vision + TFLite (float16) |
| Model architecture | YOLO11-seg (Ultralytics) |
| Training dataset | CarDD |
| Training environment | Google Colab |
| Model export | TFLite float16 (primary), ONNX (intermediate) |
| PDF generation | Flutter PDF library |
| Maps integration | Google Maps (deep link) |
| Build system | Kotlin DSL (`build.gradle.kts`) |

---

## Machine Learning Model

### Architecture

The detection backbone is a **YOLO11 segmentation model** trained on the **CarDD dataset**, a benchmark dataset for car damage segmentation. The model produces per-pixel segmentation masks for each detected damage region alongside bounding boxes and class confidence scores.

### Damage Classes

| Class | Description |
|---|---|
| `dent` | Panel deformations and body indentations |
| `scratch` | Surface paint scratches |
| `crack` | Structural cracks in body panels or glass |
| `glass_shatter` | Shattered windscreen or window glass |
| `lamp_broken` | Damaged headlights or tail lights |
| `tire_flat` | Flat or punctured tires |

### Severity Thresholds

Severity is determined by the detected mask area as a percentage of the total image area. Thresholds were established through percentile-based analysis, K-Means clustering, and alignment with insurance industry standards.

| Severity | Mask Area |
|---|---|
| **MINOR** | < 2% |
| **MODERATE** | 2% – 8% |
| **SEVERE** | > 8% |

### Training Configuration

| Parameter | Value |
|---|---|
| Best model size | YOLO11l-seg |
| Best mAP50 achieved | **76.3%** |
| Smaller variant mAP50 | 71.2% (YOLO11s-seg) |
| Maximum-accuracy run | yolo11x-seg, 1024 px, 250 epochs |
| Realistic accuracy ceiling | ~84% mAP50 on CarDD |

> **Note on accuracy expectations:** The CarDD dataset contains inherently difficult classes — dents, scratches, and cracks have low visual contrast and highly variable appearance. Overall mAP50 scores above ~84% are not realistic on this dataset regardless of model size. The 76.3% result with YOLO11l-seg represents a strong performance given these constraints.

### Model Deployment

The final on-device model is exported as **TFLite float16** (`best_float16.tflite`). Large ONNX files (85 MB+) cannot be reliably loaded from Android assets via flutter_vision due to AssetManager compression constraints. The TFLite format bypasses this limitation and provides faster inference on mobile hardware.

A required `noCompress` rule in `app/build.gradle.kts` prevents Android from compressing the `.tflite` asset at build time:

```kotlin
androidResources {
    noCompress += listOf("tflite")
}
```

---

## App Architecture

### Page Structure

The app uses a `res`-suffixed page architecture for the main runtime path. The pages actually executed at runtime are:

- `splashres_page.dart` — entry point, launches the res-page flow
- `homeres_page.dart` — home screen
- `reportres_page.dart` — damage detection and report display
- `historyres_page.dart` — scan history

> **Important for contributors:** Any bug fixes or UI changes must be applied to the `res`-suffixed page files. Changes made to the non-`res` counterparts have no effect at runtime.

### Detection Pipeline

1. User selects or captures a photo
2. Image is passed to the TFLite model via flutter_vision
3. Model returns bounding boxes, class IDs, confidence scores, and segmentation masks
4. A `_toDouble()` helper normalises flutter_vision output (which mixes `String` and `int` types)
5. Mask area is computed as a fraction of total image pixels to determine severity
6. Class + severity maps to an EGP price range
7. Results are displayed on the report page alongside the original image

### PDF Generation

Reports are generated using the Flutter PDF library. Key implementation notes:

- Page width is read from `ctx.page.pageFormat.availableWidth` to avoid `double.infinity` crashes
- Shop names use an English-only `pdfName` field; Arabic Unicode characters (including star symbols) cause PDF rendering failures and must be avoided in PDF-bound strings

---

## Setup & Installation

### 1. Clone the repository

```bash
git clone https://github.com/your-username/fixzone.git
cd fixzone
```

### 2. Download the model file

The TFLite model is not stored in this repository due to its size. Download it from Google Drive and place it in the correct assets folder:

📦 **[Download model assets from Google Drive](https://drive.google.com/drive/folders/1GyzjcwX8IhPwI8c4Z_L6ELREobqXC-fl?usp=sharing)**

Place the downloaded `best_float16.tflite` file into:

```
assets/models/best_float16.tflite
```

### 3. Install Flutter dependencies

```bash
flutter pub get
```

### 4. Run the app

```bash
flutter run
```

> Requires Android SDK and a connected Android device or emulator.

---

## Repair Shops

The report page surfaces four Cairo-area workshops with integrated Maps and call support:

| Shop | Integration |
|---|---|
| Mohamed Ali Mahrous PDR | Google Maps + phone |
| Auto Fit | Google Maps + phone |
| Dr Samkary | Google Maps + phone |
| Speed Pro Auto Service | Google Maps + phone |

---

## Known Constraints & Lessons Learned

**ONNX vs TFLite:** flutter_vision fails to load ONNX model files larger than ~85 MB from Android assets. Always deploy as TFLite float16.

**noCompress rule:** Without the `noCompress` build rule, Android compresses the `.tflite` file and the app silently fails to load it.

**Type casting:** flutter_vision returns detection output fields as a mix of `String` and `int`. A `_toDouble()` helper is required before any arithmetic on these values.

**PDF Arabic text:** The Flutter PDF library does not render Arabic Unicode symbols reliably. Use a plain English `pdfName` field for any text that will appear in generated PDFs.

**res-page architecture:** The `splashres_page.dart` entry point routes directly to `res`-suffixed pages. Non-`res` pages are not reached at runtime.

**mAP50 ceiling:** CarDD's class difficulty (especially dent and scratch) means mAP50 scores are structurally capped well below 90%. 76–77% with a large model is near the practical ceiling.

---

## Project Team

Built as a graduation project. Contributions span machine learning model training, Flutter app development, UI/UX design, and backend integration.

---

## License

This project was developed as an academic graduation project. Please contact the team before reusing any component.
