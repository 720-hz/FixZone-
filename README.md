# FixZone 🚗🔧

> Smart Vehicle Inspection and Repair Estimator — AI-powered Android app for automated car damage detection, severity grading, and repair cost estimation.

**Graduation Project — The British University in Egypt**
Faculty of Engineering · Electrical Engineering Department · Computer Engineering Programme · June 2026

**Supervisor:** Dr. Sally Saad

**Team:**
- Rawan Osama Abdelalim Mousa — 224159
- Ranya Sherif Salah Eldin Elleithy — 226701
- Mohamed Ayman El Sayed Khedr — 219203
- Ziad Khaled Eid Abdelsalam — 208605
- Ramez Ashraf Abdulmoniem Mahmoud Nasr — 225652

---

## Overview

Vehicle damage inspection is traditionally done manually by mechanics or insurance inspectors — a process that is slow, subjective, and inconsistent. FixZone automates this by letting a user photograph their car, running an on-device YOLO instance segmentation model, and delivering a full damage report with severity grades, EGP repair cost estimates, and nearby workshop recommendations — all within seconds.

The system is not a replacement for professional inspection. It is designed as a practical first-assessment tool for vehicle owners, repair shops, and insurance workflows.

---

## Features

- **On-device AI detection** — YOLOv11m-seg runs locally via TFLite, no internet required for inference
- **Six damage classes** — dent, scratch, crack, glass shatter, broken lamp, flat tire
- **Rule-based severity grading** — Minor / Moderate / Severe, derived from segmentation mask area ratio
- **EGP cost estimation** — class-specific repair cost ranges, adjusted per severity tier
- **PDF report** — downloadable report with the scanned image, all detections, costs, and workshop details
- **Workshop locator** — four Cairo-area repair shops with one-tap Google Maps directions and call buttons
- **Rule-based chatbot** — in-app assistant covering damage types, costs, report info, and app usage
- **Firebase backend** — user authentication (email, Google, Facebook, X) and scan history storage
- **Feedback system** — in-app star rating and comment form

---

## Tech Stack

| Layer | Technology |
|---|---|
| Mobile framework | Flutter (Android) |
| IDE | Android Studio |
| UI design | Figma |
| On-device inference | flutter_vision + TFLite float16 |
| Model architecture | YOLOv11m-seg (Ultralytics) |
| Training dataset | CarDD |
| Training environment | Google Colab (NVIDIA A100) |
| Training framework | PyTorch + Ultralytics YOLO |
| Model export | TFLite float16 (`best_float16.tflite`) |
| Backend | Firebase (Auth + Firestore) |
| PDF generation | Flutter PDF library |
| Maps integration | Google Maps (deep link) |
| Build system | Kotlin DSL (`build.gradle.kts`) |

---

## Setup & Installation

### 1. Clone the repository

```bash
git clone https://github.com/your-username/FixZone.git
cd FixZone
```

### 2. Download the model file

The TFLite model is not stored in this repository due to its size. Download it from Google Drive and place it in the assets folder:

📦 **[Download model assets from Google Drive](https://drive.google.com/drive/folders/1GyzjcwX8IhPwI8c4Z_L6ELREobqXC-fl?usp=sharing)**

Place the downloaded file at:

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

## Machine Learning Model

### Architecture & Selection

The model is **YOLOv11m-seg** — the medium variant of the YOLO11 instance segmentation family. It was chosen as the practical balance between accuracy and computational cost. Smaller variants miss fine-grained damage like thin cracks and shallow scratches; larger variants demand significantly more memory and training time. YOLOv11m-seg was found to be the ideal middle ground for this task.

The model produces per-pixel segmentation masks alongside bounding boxes and class confidence scores. Masks are used for severity estimation in the Python pipeline; the Flutter app displays bounding boxes only (lighter for mobile rendering).

### Dataset — CarDD

The Car Damage Detection Dataset (CarDD) is a large-scale public benchmark for vehicle damage analysis. It contains approximately 4,000 real-world vehicle images with 9,163 annotated damage instances across six classes, each annotated at the pixel level for instance segmentation.

| Split | Images | Annotated Instances |
|---|---|---|
| Train | 2,816 | — |
| Validation | 810 | 1,744 |
| Test | 374 | 785 |

### Damage Classes

| Class | Description |
|---|---|
| `dent` | Panel deformations and body indentations |
| `scratch` | Surface paint scratches |
| `crack` | Structural cracks in body panels |
| `glass_shatter` | Shattered windscreen or window glass |
| `lamp_broken` | Damaged headlights or tail lights |
| `tire_flat` | Flat or punctured tires |

### Training Configuration

| Parameter | Value |
|---|---|
| Model | YOLOv11m-seg |
| Task | Instance segmentation |
| Approach | Transfer learning |
| Input image size | 832 × 832 px |
| Epochs | 100 |
| Batch size | 4 |
| Early stopping patience | 25 epochs |
| Hardware | Google Colab (NVIDIA A100) |
| Random seed | 42 |

### Results

| Metric | Validation | Test |
|---|---|---|
| Box mAP50 | 0.770 | **0.769** |
| Mask mAP50 | 0.763 | **0.763** |
| Box Precision | 0.798 | 0.803 |
| Box Recall | 0.735 | 0.731 |

#### Per-class mask mAP50 (test set)

| Class | mAP50 |
|---|---|
| glass_shatter | 0.989 |
| tire_flat | 0.916 |
| lamp_broken | 0.901 |
| dent | 0.636 |
| scratch | 0.607 |
| crack | 0.530 |

The model performs strongest on visually distinctive classes (glass shatter, tire flat, lamp broken) and weaker on subtle surface damage (dent, scratch, crack), which is expected given the inherent visual difficulty of these classes and their tendency to blend with normal car body textures.

### Post-Processing

Class-specific confidence thresholds are applied after inference to filter out low-quality detections before severity and cost estimation.

| Class | Confidence Threshold |
|---|---|
| Dent | 0.35 |
| Scratch | 0.35 |
| Crack | 0.25 |
| Glass Shatter | 0.30 |
| Broken Lamp | 0.40 |
| Flat Tire | 0.40 |

On the test set, this reduced 879 raw predictions to 641 final detections (238 removed).

---

## Severity Estimation

Severity is determined by the ratio of the segmentation mask area to the full image area. Thresholds are applied per class, since different damage types carry different risk levels at the same visual extent. Thresholds are set conservatively — the system has no separate car-part segmentation model, so severity is scored against the full image, not the specific panel or component.

| Class | Minor → Moderate | Moderate → Severe | Additional Rule |
|---|---|---|---|
| Scratch | 3.0% | 18.0% | Severe only if widespread |
| Dent | 2.5% | 12.0% | Severe only if visibly large |
| Crack | 2.0% | 10.0% | Serious but not instantly severe |
| Glass Shatter | 4.0% | 20.0% | Severe if large glass region affected |
| Broken Lamp | 3.0% | 15.0% | Minimum severity is Moderate |
| Flat Tire | — | — | **Always Severe** |

When multiple damage instances are detected, the overall scan severity is taken from the highest severity level found.

---

## Repair Cost Estimation

Cost ranges were derived from field surveys and direct consultation with Egyptian auto repair workshops. Each class maps to a min–max EGP range; the severity level determines where within that range the estimate falls (lower end = Minor, mid = Moderate, upper end = Severe).

| Class | Min (EGP) | Max (EGP) |
|---|---|---|
| Scratch | 300 | 2,000 |
| Dent | 600 | 3,000 |
| Crack | 1,200 | 2,200 |
| Glass Shatter | 750 | 2,000 |
| Broken Lamp | 1,500 | 2,300 |
| Flat Tire | 10,000 | 20,000 |

Cost outputs are estimates and not exact quotations. Actual repair prices vary by vehicle brand, spare part availability, workshop location, and market conditions.

---

## App Architecture

### Page Structure

The app uses a `res`-suffixed page architecture for the main runtime path. The entry point is `splashres_page.dart`, which routes directly into the `res`-suffixed pages. The non-`res` counterparts are not reached at runtime.

| Page | Role |
|---|---|
| `splashres_page.dart` | App entry point and routing |
| `homeres_page.dart` | Home screen (camera / upload / navigation) |
| `reportres_page.dart` | Damage detection result and cost display |
| `historyres_page.dart` | Previous scan history |

> **For contributors:** All fixes and UI changes must be applied to the `res`-suffixed pages. Changes to non-`res` pages have no effect at runtime.

### Detection Pipeline (Flutter)

1. User selects or captures an image
2. Image is passed to `best_float16.tflite` via flutter_vision
3. A `_toDouble()` helper normalises the mixed `String`/`int` output from flutter_vision
4. Mask area is computed as a fraction of total image pixels → determines severity
5. Class + severity maps to an EGP cost range
6. Bounding boxes (not masks) are displayed on the result page alongside the original image

### Model Deployment

The on-device model is `best_float16.tflite`. ONNX files above ~85 MB cannot be reliably loaded from Android assets via flutter_vision due to AssetManager compression limits. TFLite float16 bypasses this and provides faster mobile inference.

A required `noCompress` rule in `app/build.gradle.kts` prevents Android from compressing the asset at build time:

```kotlin
androidResources {
    noCompress += listOf("tflite")
}
```

### Firebase Integration

Firebase handles user authentication and data persistence. Each user's scan history, generated reports, and account information are stored and retrieved per session. The authentication system supports email/password, Google, Facebook, and X sign-in.

### PDF Generation

Reports are generated using the Flutter PDF library. Key implementation constraints:

- Page width is read from `ctx.page.pageFormat.availableWidth` — using `double.infinity` causes a crash
- Shop names use an English-only `pdfName` field — Arabic Unicode characters (including star symbols) cause PDF rendering failures

---

## Repair Shops

Four Cairo-area workshops are embedded in the app with Google Maps directions and direct phone call buttons:

| Shop | Area |
|---|---|
| Mohamed Ali Mahrous PDR | Old Cairo, Salah Salem St |
| Auto Fit | El-Demerdash, Cairo |
| Dr Samkary | Nasr City, Cairo |
| Speed Pro Auto Service | New Cairo |

---

## Known Constraints & Lessons Learned

**ONNX vs TFLite** — flutter_vision cannot load ONNX files above ~85 MB from Android assets. Always deploy as TFLite float16.

**noCompress rule** — Without this Kotlin DSL rule, Android silently compresses the `.tflite` asset and the model fails to load.

**Type casting** — flutter_vision returns detection output as a mix of `String` and `int`. A `_toDouble()` helper is required before any arithmetic on these values.

**Arabic text in PDFs** — Arabic Unicode and star symbols fail in the Flutter PDF library. Use a plain English `pdfName` field for all PDF-bound strings.

**res-page routing** — Non-`res` pages are never reached at runtime. All development work targets the `res`-suffixed files.

**mAP50 ceiling on CarDD** — Dent, scratch, and crack are inherently difficult classes. Realistic mAP50 on CarDD is structurally capped well below 90%; 76–77% with YOLOv11m-seg is near the practical ceiling for this dataset.

---

## Demo

A full walkthrough of the FixZone application is available in the demo video included in this repository. It covers the complete user flow — from login and image upload through damage detection, severity grading, cost estimation, PDF report generation, and the chatbot interface.

---

## Project Report

The full dissertation — *Smart Vehicle Inspection and Repair Estimator* — is available in this repository as `report.pdf`.

---

## Contact

For questions, collaboration, or any inquiries about this project, feel free to reach out:

📧 **ziadkhaledshatah@gmail.com**

---

## License

Developed as an academic graduation project at The British University in Egypt. Contact the team before reusing any component.
