# Brutal Q&A Prep for Poster Presentation

Here are 5 brutal, academically rigorous questions a professor is likely to ask about this project, along with the most optimal, defensive answers to protect your technical design choices.

### 1. The Synthetic Data Paradox
**Q (Brutal):** *"Your entire XGBoost model was trained on 180,000 samples of synthetic data that you generated yourself using mathematical formulas. Haven't you just trained a heavy ML model to memorize your own equations? How can you claim this will work on the chaotic, unpredictable physiology of a real autistic child?"*

**Optimal Answer:** 
"That is a highly valid concern. Our current model is indeed learning the boundaries of our mathematical generators. However, the purpose of this phase of the project was to validate the **data pipeline architecture and latency**, not clinical diagnostics. We proved that our decoupled system can ingest 10Hz data, calculate rolling 5-second variance features, normalize them via Z-scores, and run XGBoost inference in under 100 milliseconds. The architecture is fully established. The next immediate phase is clinical validation—where we will hot-swap the synthetic `model.pkl` with one trained on real-world longitudinal data from autistic children, without needing to rewrite a single line of our telemetry or mobile infrastructure."

---

### 2. The Flawed Baseline Argument
**Q (Brutal):** *"You enforce a 30-second baseline calibration to establish a Z-score. What happens if the child puts the vest on while they are already experiencing high anxiety? Your system will normalize their stressed state as the 'relaxed baseline', meaning it will fail to detect further escalation."*

**Optimal Answer:** 
"To mitigate this, we implemented strict physiological guardrails in our `baseline.py` module. While the system computes the local mean and standard deviation, it enforces absolute minimum variance floors (e.g., GSR variance cannot be $0.0$). More importantly, in a clinical deployment, this 30-second local baseline would be cross-referenced with a global, long-term historical baseline stored for that specific patient. If the initial 30-second resting heart rate is 115 BPM, the system would flag it against the historical baseline and warn the caregiver immediately upon startup, rather than blindly accepting it as 'relaxed'."

---

### 3. Battery Life vs. Streaming
**Q (Brutal):** *"You are streaming raw, 4-channel physiological data at 10Hz over WebSockets/Bluetooth to a backend server for inference. In a real-world wearable, keeping a continuous high-frequency radio stream open will drain a lithium-polymer battery in hours. Why didn't you compute this on the edge?"*

**Optimal Answer:** 
"You are exactly right; continuous raw streaming is extremely power-intensive. We designed this current iteration as a 'Research & Diagnostics' prototype, where maximum data fidelity is required on the dashboard to visualize the exact waveforms and validate the feature extraction math. In the final production wearable, as noted in our 'Future Improvements', the 5-second sliding window and the XGBoost inference will be deployed directly onto the vest's edge microcontroller (using TinyML or TensorFlow Lite). The vest will then only need to transmit a single 1Hz byte (the Stress Score), drastically reducing the radio duty cycle and extending battery life to several days."

---

### 4. The Latency of Human Physiology
**Q (Brutal):** *"You're using a 5-second sliding window for feature extraction. But Galvanic Skin Response (eccrine sweat gland activation) typically has a physiological lag of 1 to 3 seconds after the actual sympathetic nervous system trigger. Isn't your 5-second window too narrow? You risk missing the holistic picture of the stress onset."*

**Optimal Answer:** 
"While GSR does suffer from a 1-3 second physiological delay, our system relies on **multimodal sensor fusion**. Heart Rate Variability (HRV) and respiration-sinus-arrhythmia react to vagal withdrawal much faster than eccrine sweat glands, often within a single cardiac cycle. Additionally, the IMU captures instantaneous physical agitation (stimming, erratic movement). By using a 5-second window, we capture the immediate biomechanical and cardiovascular onset, and the trailing edge of the window catches the delayed GSR surge. This fusion is exactly why the XGBoost algorithm outperforms single-sensor thresholds."

---

### 5. Why XGBoost over Deep Learning?
**Q (Brutal):** *"For time-series biological data, LSTMs or 1D Convolutional Neural Networks are the gold standard. Why did you use an old decision-tree algorithm like XGBoost? It ignores the sequential temporal nature of the data."*

**Optimal Answer:** 
"While Deep Learning is excellent for raw time-series sequences, it requires massive computational overhead and makes edge-deployment significantly harder. We bypassed the need for an LSTM by performing explicit, domain-specific feature engineering in our sliding window (calculating statistical variance, min/max, and peak-to-peak amplitudes). We embedded the 'time' context into the features themselves. XGBoost is vastly superior at handling tabular, engineered features, trains orders of magnitude faster, is highly interpretable (we can see exactly which sensor triggered the alert), and crucially, its decision trees can be easily compiled into C-code for ultra-low-power microcontrollers in the future."
