# Smart Vest Project: Comprehensive Technical Architecture

This document serves as an in-depth technical breakdown of the Smart Vest physiological telemetry and stress detection ecosystem. It details the underlying mathematical logic, the microservice architecture, and the machine learning pipeline used to achieve real-time, low-latency stress monitoring.

---

## 1. System Overview & Objective

The **Smart Vest** is a simulated wearable ecosystem designed to monitor an individual's autonomic nervous system responses in real-time. By tracking physiological markers—such as sweat gland activity, cardiovascular variance, and biomechanical movement—the system uses an advanced Machine Learning ensemble to differentiate between normal physiological variance (e.g., walking, baseline anxiety) and acute, overwhelming stress events. 

When acute stress is detected, the ecosystem coordinates across WebSockets and Bluetooth to trigger haptic feedback and push notifications on a companion mobile device, acting as a real-time behavioral intervention tool.

---

## 2. Microservice Architecture

To ensure high performance and zero-blocking telemetry streaming, the project is decoupled into five distinct processing layers:

### A. The Simulation Engine (`simulation_engine/`)
Because testing with real human biological sensors is complex and unpredictable, this Python FastAPI service acts as a mathematically rigorous "Digital Twin" of a human body.
- **Frequency**: Broadcasts raw sensor data at exactly **10 Hz**.
- **Physiological Generators**:
  - **GSR (Galvanic Skin Response)**: Simulates skin conductance (µS). It uses a baseline of ~5.0µS, adding Perlin-like noise and occasional spontaneous fluctuations (SCRs) to mimic real eccrine sweat gland behavior.
  - **Heart Rate (HR/HRV)**: Simulates the RR-interval of the heart. Uses sinusoidal breathing-sinus-arrhythmia (RSA) coupled with the respiration wave.
  - **Respiration**: A sinusoidal waveform mimicking inhalation/exhalation chest expansion, operating around 15 breaths per minute (0.25 Hz).
  - **IMU (Inertial Measurement Unit)**: Generates 3-axis accelerometer data to track physical activity magnitude.
- **Stress Injection**: Exposes a `/stress/tap` API. When triggered, the engine smoothly transitions the mathematical bounds of the generators—raising heart rate to 120+ BPM, dropping HRV, spiking GSR voltage, and increasing IMU noise—simulating an acute panic or stress response.

### B. Machine Learning Pipeline (`ml_pipeline/`)
This FastAPI service consumes the 10Hz raw stream and runs it through an offline-trained Machine Learning inference engine to produce a **1Hz** Stress Score.
- **Feature Extraction (Sliding Window)**: The raw 10Hz data is extremely noisy. The `features.py` module maintains a rolling 5-second buffer (50 data points). It calculates statistical features (Mean, Variance, Max, Min, Peak-to-Peak amplitude) across all sensors.
- **Z-Score Calibration (The Baseline)**: Physiological baselines vary wildly between users. The pipeline features a **30-second Calibration Phase**. It records 300 data points to establish the user's current resting mean and standard deviation ($\mu, \sigma$).
  - *Guardrails*: To prevent mathematical explosions, the pipeline enforces absolute standard deviation floors (e.g., GSR variance cannot be mathematically $0.0$).
  - All incoming live features are converted to Z-Scores: $Z = \frac{X - \mu}{\sigma}$. This allows the ML model to evaluate *relative deviation* rather than absolute values.
- **The XGBoost Inference Engine**: The core intelligence. 
  - Trained offline via `notebooks/Train_Model.ipynb` using a synthetic dataset of **180,000 samples** and 1,000 decision trees utilizing GPU-accelerated Hist-Gradient Boosting.
  - The model outputs a probability density ($0.0$ to $1.0$), which the pipeline scales into a human-readable **Stress Score (0-100)**.

### C. The Dashboard Proxy & BLE Server (`dashboard/backend/`)
The routing layer that coordinates the entire ecosystem.
- **WebSocket Aggregation**: It connects to both the Simulation Engine (Port 8000) and ML Pipeline (Port 8001), merging the 10Hz raw data and 1Hz ML inferences into a single optimized payload.
- **GATT Bluetooth Server (`ble_server.py`)**: A parallel background process built using the `bless` library. It exposes a native Bluetooth Low Energy (BLE) peripheral named "SmartVest". It broadcasts the real-time stress score over Bluetooth Characteristic UUIDs, allowing nearby smartphones to read the data without a WiFi connection.

### D. The Desktop Dashboard (`dashboard/frontend/`)
A React and Vite-powered visualization suite for the desktop.
- Implements `Recharts` for high-performance, 60fps oscilloscope rendering of the biological waveforms.
- Provides the control interface to trigger the 30-second Calibration Phase and to inject synthetic Stress Events.

### E. The Mobile Ecosystem
The system features a dual-approach mobile deployment strategy to ensure continuous behavioral intervention.
1. **Mobile Web App (`mobile_app/`)**: A responsive, lightweight Vite application accessible via the local network IP. Designed for quick access and cross-platform compatibility.
2. **True Native Expo App (`native_app/`)**: A fully native Android/iOS application built with React Native and Expo. 
   - **Deep OS Integration**: This is the critical intervention tool. Because web browsers sandbox hardware access, the Native App uses `expo-notifications` and `expo-haptics` to break out of the sandbox.
   - **Trigger Logic**: When the user-defined threshold (e.g., Score > 75) is breached, the Native App triggers a high-priority system alarm, forces deep hardware vibration, and drops a persistent notification from the OS tray ("*High Stress Detected. Please take a moment to breathe.*").
   - **Multi-Connectivity**: The app supports WebSockets over Local IP, WebSockets over Cellular via Cloud Proxy (Ngrok), and direct Bluetooth LE polling.

---

## 3. Data Flow & Lifecycle Summary

1. **Initialization**: The ecosystem boots. The Simulation Engine begins generating resting biological data.
2. **Calibration**: The user presses "Start 30s Baseline". The ML Pipeline enters a strict recording state, freezing inference to calculate the user's exact physiological resting state ($\mu, \sigma$).
3. **Monitoring**: The system enters nominal tracking. Raw data streams to the React oscilloscopes. The ML pipeline computes sliding-window features, normalizes them via Z-Score against the baseline, and passes them to the XGBoost model. The model outputs a low probability, translating to a Stress Score of ~10-20.
4. **Stress Incident**: The user taps "Tap to Stress". The biological generators simulate an acute stress event. Heart rate spikes, skin conductance climbs.
5. **Detection & Alert**: The XGBoost model recognizes the distinct multivariate statistical signature of stress (high variance, high positive Z-Scores). The Stress Score spikes to 90+. 
6. **Intervention**: The Desktop Dashboard UI turns flashing red. Simultaneously, the Native Mobile App detects the breach via WebSockets/Bluetooth, vibrating the user's physical phone hardware and triggering the system alarm sound, prompting immediate behavioral intervention.
