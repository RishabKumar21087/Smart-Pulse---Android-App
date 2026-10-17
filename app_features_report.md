# AirPulse: PPG Android Application - Technical & Feature Report

## Overview
AirPulse is an Android-based health and environmental monitoring application. It calculates a user's heart rate utilizing Photoplethysmography (PPG) through the smartphone camera, combines it with real-time Air Quality Index (AQI) data obtained via the user's geographic location, and provides a comprehensive, personalized health risk assessment based on the user's pre-defined health profile.

---

## 1. Core Features & Functionality
### 1.1 Heart Rate Monitoring (PPG via Camera)
*   **Feature**: Measures heart rate in Beats Per Minute (BPM) non-intrusively. The user places their finger over the smartphone's camera lens and flash. 
*   **Implementation**: Calculates the variations in blood volume by analyzing the intensity of the *Red* light channel reflected from the skin. 
*   **Flow**: The measurement starts when `MainActivity` is launched. It queues intensity data, processes it via Fast Fourier Transform (FFT), identifies the peak frequency within the expected human heart rate range (45-150 BPM), and calculates the BPM. The recording stops automatically after sampling enough frames (approx. 512 points for optimal accuracy) and saves the measurement to local storage.

### 1.2 Real-Time Air Quality Index (AQI) Tracking
*   **Feature**: Checks localized air quality to give immediate environmental context to the user.
*   **Implementation**: Utilizes the user's exact GPS coordinates to fetch AQI data from the AQICN (waqi.info) API. Returns the numeric AQI value, pollutant data, categorized risk level (e.g., "Good", "Moderate"), and actionable health tips.

### 1.3 Adaptive Risk Calculation
*   **Feature**: Calculates an individualized health risk score and provides specific recommendations.
*   **Implementation**: `RiskCalculator.kt` evaluates inputs dynamically:
    *   **Heart Rate (BPM)**: Flags bradycardia (< 50 BPM) or tachycardia (> 120 BPM).
    *   **Air Quality (AQI)**: Scales score from healthy (0-50) to hazardous (> 200).
    *   **User Health Profile**: Considers parameters recorded in `QuestionnaireActivity` (Age range, exercise frequency, pre-existing respiratory or cardiovascular conditions).
    *   **Result**: Outputs a comprehensive score categorized into Low, Mild, Moderate, or High Risk, dynamically generating tailored recommendations.

### 1.4 User Profile, History, & Authentication
*   **Feature**: Maintains a seamless user experience prioritizing local app storage, maintaining a longitudinal record of health checks. 
*   **Implementation**: Contains login/signup logic mapped purely to on-device memory (`LocalStorage.kt`), tracking the user's customized health questionnaire and keeping a historical ledger of up to 50 previous BPM and AQI measurements.

---

## 2. Technology Stack
### 2.1 Language and Environment
*   **Kotlin (v2.1.0) & Java (v1.8)**: Core application languages. Most of the UI and newer logic (AQI, Risk, Storage) leverage Kotlin for conciseness, while heavy mathematical and OpenCV legacy code utilizes Java (`MainActivity.java`).
*   **Android SDK**: Targets SDK 34 (Android 14) and requires a minimum of SDK 21 (Android 5.0).
*   **Gradle**: Build tool utilizing version `8.13.2`.

### 2.2 Frameworks & Libraries
*   **OpenCV (v3.0.0)**: Included as a library module (`openCVLibrary300`). Used heavily in `MainActivity.java` and `javaViewCameraControl.java` to interface with the camera feed (`CvCameraViewListener2`) and extract the Red-channel color intensity matrix using `Mat` arrays.
*   **Retrofit & OkHttp (v2.9.0 & v4.12.0)**: Used for robust, asynchronous RESTful API consumption in `AQIService.kt` to interact with the waqi.info API, serializing network responses via Gson.
*   **Google Play Services Location (v21.1.0)**: Responsible for obtaining fine and coarse GPS coordinates to feed into the API.
*   **Gson (v2.10.1)**: Used heavily to parse JSON responses from Retrofit, and importantly, for serializing/deserializing the custom `HistoryEntry` data class into a string format so it can be saved persistently inside Android `SharedPreferences`.

### 2.3 UI & Design
*   **XML & ViewBinding**: The app utilizes traditional XML layouts structured with `ConstraintLayout`, `CardView`, and `RecyclerView`. `ViewBinding` is enabled in Gradle for safe view interactions.
*   **Material Design (v1.11.0)**: Provides modernized UI elements like Buttons, Cards, and styling variables.

---

## 3. Deep Dive: Architectural Implementation Details

### 3.1 PPG Algorithm Flow (`MainActivity.java` & `fftLib.java`)
1.  **Camera Config:** Turns on the flash (`turnFlashOn`) and locks the framerate aggressively aiming for 30FPS.
2.  **Color Extraction:** Each camera frame is split into base channels (`Core.split()`). The Red channel (`img_comp.get(0)`) is passed to `getMatAvg()`, which aggregates pixel intensity into a single double value.
3.  **Buffering:** Values are buffered into a custom 2D queue block (`DoubleTwoDimQueue`), discarding the first uncalibrated 30 frames.
4.  **Signal Processing (FFT):** Once 512 frames are amassed, an asynchronous thread (`myFFTThread`) begins analysis. It extracts the raw values (`sample_arr`) and calculates real FPS utilizing `timestampQ`.
5.  **Frequency Analysis:** The `fftLib` computes the power spectrum. The algorithm iterates through bounds corresponding to `45 BPM` (0.75Hz) and `150 BPM` (2.5Hz), identifying the frequency with the highest magnitude. The corresponding frequency is then multiplied by 60 to output the user's BPM.

### 3.2 GPS and API Processing (`AQIService.kt` & `AQIActivity.kt`)
*   Modeled using robust data classes to strictly match the WAQI JSON layout (`AQIResponse`, `AQIData`, `AQICity`, `AQIPollutant`).
*   `AQIApiInterface` defines the `@GET` endpoint `/feed/geo:{lat};{lon}/` combining the geographic payload asynchronously using standard `Call` models instead of suspending functions.
*   Contains built-in deterministic helper methods strictly scaling standard AQI ranges 0-50, 51-100, 101-150+ into respective warning strings and hex color codes.

### 3.3 Data Persistence Strategy (`LocalStorage.kt`)
*   Acts as a facade representing state within Android `SharedPreferences` scoped privately. 
*   Avoids utilizing complex databases (like Room/SQLite). Instead, saves boolean flags (like `questionnaire_done`) and uses `TypeToken` generics alongside `Gson` to read/write JSON lists for storing historical `HistoryEntry` logs up to a predefined limit of 50.
