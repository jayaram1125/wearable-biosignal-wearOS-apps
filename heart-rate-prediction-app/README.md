# Heart Rate Prediction App

A Wear OS research and reference application for real-time heart-rate estimation from wearable photoplethysmography (PPG) and motion-sensor data.

The application demonstrates the complete deployment path developed across the related open-source projects:

```text
Public PPG datasets
        ↓
heart-rate-prediction-benchmark
        ↓
Selected signal-processing pipeline
        ↓
heart-rate-signal-processing-sdk
        ↓
Selected heart-rate ML model
        ↓
ExecuTorch model deployment
        ↓
heart-rate-prediction-app
        ↓
Real-time heart-rate estimation
```

> **Project status:** Research and development  
> **Platform:** Wear OS  
> **Language:** Kotlin  
> **ML runtime:** ExecuTorch  
> **Purpose:** Research, experimentation, benchmarking, and demonstration  
> **Medical use:** This application is not a medical device and is not intended for diagnosis, treatment, or clinical decision-making.

---

# Project Location

This application is maintained within the:

**`wearable-biosignal-wearOS-apps`**

repository.

Application module:

```text
wearable-biosignal-wearOS-apps/
└── heart-rate-prediction-app/
```

The application depends on the separately developed:

```text
wearable-biosignal-sdk/
└── heart-rate-signal-processing-sdk/
```

and on the heart-rate model and processing methodology selected through experiments in:

```text
heart-rate-prediction-benchmark/
```

---

# Objective

Wearable devices commonly use photoplethysmography to estimate heart rate.

PPG-based heart-rate estimation can work well under relatively stable conditions, but physical activity can introduce substantial signal corruption through:

- arm movement;
- walking;
- running;
- exercise;
- changes in sensor-to-skin contact;
- wrist movement;
- periodic motion;
- ambient-light interference;
- changes in peripheral blood flow.

The purpose of `heart-rate-prediction-app` is to evaluate the complete real-time inference pipeline on a wearable device:

> **Acquire raw physiological and motion signals → apply the experimentally selected signal-processing pipeline → generate model-compatible tensors → perform on-device ML inference → display the estimated heart rate.**

The app is intended to provide a reproducible reference implementation showing how algorithms developed and validated in `heart-rate-prediction-benchmark` can be translated into an operational wearable application.

---

# Relationship to the Research Project

The application does **not** independently choose its signal-processing algorithm or machine-learning model.

Those components are selected through controlled experiments performed in:

**`heart-rate-prediction-benchmark`**

The benchmark evaluates combinations of:

```text
PPG preprocessing
        +
motion-artifact mitigation
        +
accelerometer information
        +
signal-quality assessment
        +
tensor representation
        +
machine-learning model
```

against ECG-derived ground-truth heart rate in publicly available research datasets.

Candidate signal-processing approaches may include:

- Butterworth band-pass filtering;
- Savitzky-Golay smoothing;
- wavelet denoising;
- LMS adaptive filtering;
- NLMS adaptive filtering;
- accelerometer-assisted artifact reduction;
- motion-aware spectral processing;
- signal-quality gating.

Candidate model architectures may include:

- classical peak/frequency-domain baselines;
- 1-D CNN;
- temporal convolutional networks;
- CNN-GRU architectures;
- compact fully convolutional models;
- BeliefPPG;
- KID-PPG;
- other reproducible candidate models evaluated by the benchmark.

The selected implementation will be based on measured:

- accuracy;
- activity-specific error;
- subject-independent performance;
- cross-dataset performance;
- motion robustness;
- computational cost;
- model size;
- inference latency.

No algorithm or model is assumed to be optimal before these experiments are completed.

---

# End-to-End Architecture

```text
┌──────────────────────────────────┐
│          Wear OS Device          │
│                                  │
│  PPG sensor       Accelerometer  │
└───────┬─────────────────┬────────┘
        │                 │
        ▼                 ▼
┌──────────────────────────────────┐
│       Sensor Acquisition         │
│                                  │
│  Raw PPG samples                 │
│  ACC X / Y / Z                   │
│  timestamps                      │
│  sampling metadata               │
└────────────────┬─────────────────┘
                 │
                 ▼
┌──────────────────────────────────┐
│ heart-rate-signal-processing-sdk │
│                                  │
│ Input validation                 │
│ Synchronization                  │
│ Windowing                        │
│ Filtering                        │
│ Motion-artifact processing       │
│ Signal-quality assessment        │
│ Normalization                    │
│ Tensor generation                │
└────────────────┬─────────────────┘
                 │
                 ▼
┌──────────────────────────────────┐
│       Model Input Tensor         │
│                                  │
│ processed PPG                    │
│ ACC-X                            │
│ ACC-Y                            │
│ ACC-Z                            │
│ derived motion features          │
└────────────────┬─────────────────┘
                 │
                 ▼
┌──────────────────────────────────┐
│       ExecuTorch Runtime         │
│                                  │
│ Load exported .pte model         │
│ Create ExecuTorch tensor         │
│ Execute on-device inference      │
└────────────────┬─────────────────┘
                 │
                 ▼
┌──────────────────────────────────┐
│       Heart-Rate Estimate        │
│                                  │
│ Estimated BPM                    │
│ Signal quality                   │
│ Motion state                     │
│ Confidence if supported          │
└────────────────┬─────────────────┘
                 │
                 ▼
┌──────────────────────────────────┐
│           Wear OS UI             │
│                                  │
│ Heart rate                       │
│ Signal quality                   │
│ Measurement state                │
│ Motion / reliability indicators  │
└──────────────────────────────────┘
```

---

# Sensor Acquisition

Raw PPG access is not exposed uniformly across all Wear OS devices.

The application therefore uses a sensor-provider abstraction so acquisition can be adapted to supported hardware without coupling the rest of the application to one device manufacturer.

Conceptually:

```kotlin
interface BiosignalSensorProvider {

    fun start()

    fun stop()

    fun observePpg(): Flow<PpgSample>

    fun observeAccelerometer(): Flow<AccelerometerSample>
}
```

This allows future support for additional wearable devices or replayed research data.

---

# Initial Hardware Target

The initial implementation may target compatible Samsung Galaxy Watch devices through the Samsung Health Sensor SDK.

The intended signal inputs are:

```text
PPG
+
ACC-X
+
ACC-Y
+
ACC-Z
+
timestamps
```

The exact channels and sampling requirements will be determined by the final processing/model pipeline selected in `heart-rate-prediction-benchmark`.

The architecture should avoid permanently coupling the complete app to one vendor-specific sensor API.

---

# Sensor Provider Architecture

```text
                    BiosignalSensorProvider
                            │
             ┌──────────────┼──────────────┐
             │              │              │
             ▼              ▼              ▼
       SamsungProvider   FutureProvider  ReplayProvider
             │
             ▼
     Raw wearable signals
```

A `ReplayProvider` is useful for development and regression testing because previously recorded PPG and accelerometer windows can be passed through the same runtime pipeline.

---

# Signal Processing

Raw sensor data is passed to:

**`heart-rate-signal-processing-sdk`**

maintained in the `wearable-biosignal-sdk` repository.

The Wear OS application should not independently duplicate the signal-processing methodology.

Instead:

```text
heart-rate-prediction-app
        ↓
heart-rate-signal-processing-sdk
        ↓
model-ready tensor
```

The SDK implementation itself is based on the signal-processing pipeline selected through `heart-rate-prediction-benchmark`.

---

# Heart Rate Signal Processing SDK Responsibilities

The SDK is expected to perform functionality such as:

```text
Raw PPG / ACC
      ↓
input validation
      ↓
timestamp alignment
      ↓
resampling if required
      ↓
window generation
      ↓
PPG preprocessing
      ↓
motion-artifact reduction
      ↓
signal-quality estimation
      ↓
normalization
      ↓
tensor generation
```

The exact processing chain will be selected based on benchmark results.

---

# Training–Deployment Consistency

A critical design requirement is:

> **The preprocessing performed on the Wear OS device must reproduce the preprocessing used when the selected heart-rate model was trained.**

For example, if `heart-rate-prediction-benchmark` selects a model trained using:

```text
PPG + accelerometer
        ↓
fixed-duration windows
        ↓
selected filtering
        ↓
motion-artifact processing
        ↓
normalization
        ↓
tensor generation
```

then `heart-rate-signal-processing-sdk` must implement those same transformations for runtime inference.

A mismatch between training-time preprocessing and runtime preprocessing can substantially degrade model performance.

---

# Tensor Contract

The signal-processing SDK and model must share a formally defined model-input contract.

An illustrative example is:

```text
Tensor shape:
[1, 4, N]

Batch:
1

Channels:
0 = processed PPG
1 = ACC-X
2 = ACC-Y
3 = ACC-Z

Data type:
float32
```

The final tensor shape will be determined by the selected model.

The contract should document:

- tensor dimensions;
- channel ordering;
- sampling frequency;
- window duration;
- stride;
- normalization;
- expected units;
- expected value ranges;
- output format.

A dedicated document such as:

```text
MODEL_CONTRACT.md
```

should be maintained with the application.

---

# Model Development

The heart-rate model is developed and evaluated in:

**`heart-rate-prediction-benchmark`**

The research workflow is:

```text
Public research datasets
        ↓
Candidate signal-processing pipelines
        ↓
Tensor generation
        ↓
Model training / fine-tuning
        ↓
Validation
        ↓
Subject-independent testing
        ↓
Activity-specific evaluation
        ↓
Cross-dataset testing
        ↓
Final pipeline/model selection
```

The model used by `heart-rate-prediction-app` should correspond to the selected benchmark configuration.

---

# PyTorch to ExecuTorch Deployment

The selected PyTorch model will be prepared for on-device inference using ExecuTorch.

Conceptually:

```text
Final model from
heart-rate-prediction-benchmark
        ↓
PyTorch model
        ↓
model export
        ↓
ExecuTorch preparation
        ↓
ExecuTorch program
        ↓
.pte model artifact
        ↓
heart-rate-prediction-app
```

The exported model should be versioned.

Example:

```text
models/
└── heart_rate/
    ├── metadata.json
    └── heart_rate_model_v1.pte
```

---

# ExecuTorch Runtime

`heart-rate-prediction-app` will use the ExecuTorch Android runtime for local model inference.

The runtime flow is:

```text
Load .pte model
      ↓
Receive tensor from
heart-rate-signal-processing-sdk
      ↓
Create ExecuTorch input tensor
      ↓
Run inference
      ↓
Read prediction output
      ↓
Convert output to BPM
```

---

# Conceptual Kotlin Inference

```kotlin
class HeartRateInferenceEngine(
    private val modelPath: String
) {

    private val module = Module.load(modelPath)

    fun predict(
        modelInput: HeartRateModelInput
    ): HeartRatePrediction {

        val tensor = Tensor.fromBlob(
            modelInput.values,
            modelInput.shape
        )

        val output = module.forward(
            EValue.from(tensor)
        )

        val estimatedHeartRate =
            output[0]
                .toTensor()
                .getDataAsFloatArray()[0]

        return HeartRatePrediction(
            bpm = estimatedHeartRate
        )
    }
}
```

This represents the intended architecture. The final implementation will follow the selected ExecuTorch API version.

---

# Complete Runtime Pipeline

```text
Wear OS Device
        │
        ▼
Raw PPG
+
Accelerometer X/Y/Z
        │
        ▼
Sensor Provider
        │
        ▼
Signal Window Buffer
        │
        ▼
heart-rate-signal-processing-sdk
        │
        ├── validate
        ├── synchronize
        ├── filter
        ├── reduce motion artifacts
        ├── assess signal quality
        ├── normalize
        └── generate tensor
        │
        ▼
Model-ready tensor
        │
        ▼
ExecuTorch runtime
        │
        ▼
Selected model from
heart-rate-prediction-benchmark
        │
        ▼
Heart-rate prediction
        │
        ▼
Reliability checks
        │
        ▼
Wear OS UI
```

---

# Application State

The UI should communicate the measurement state rather than display an HR number regardless of signal quality.

Example states:

```text
Waiting for sensor
        ↓
Collecting signal
        ↓
Building measurement window
        ↓
Processing signal
        ↓
Running inference
        ↓
Heart rate available
```

Possible reliability/error states include:

```text
Poor signal quality

Excessive motion

Insufficient samples

Sensor unavailable

Model unavailable

Inference failed
```

---

# Proposed User Interface

A minimal display may resemble:

```text
┌──────────────────────────┐
│       Heart Rate         │
│                          │
│          142             │
│           BPM            │
│                          │
│ Signal Quality           │
│ ● Good                   │
│                          │
│ Motion                   │
│ ● Moderate               │
│                          │
│ Measuring...             │
└──────────────────────────┘
```

A debug/research view may additionally display:

- raw PPG;
- processed PPG;
- accelerometer magnitude;
- signal-quality score;
- tensor/window information;
- preprocessing latency;
- inference latency;
- model version.

---

# Ground Truth and Validation

The live Wear OS device generally does not provide ECG-derived ground truth for the custom prediction pipeline.

Therefore, the wearable's own processed heart-rate estimate should not automatically be treated as scientific ground truth.

The primary model-validation evidence should come from `heart-rate-prediction-benchmark`, where predictions can be compared against synchronized ECG reference measurements in public datasets.

The Wear OS app is primarily intended to demonstrate:

- real-time signal acquisition;
- runtime signal processing;
- tensor generation;
- on-device model inference;
- application integration;
- computational feasibility;
- live wearable behavior.

---

# Application Validation

## Sensor acquisition validation

Verify:

- sampling rates;
- timestamp behavior;
- missing samples;
- channel ordering;
- sensor start/stop behavior.

## Python/Kotlin processing equivalence

For identical input:

```text
Python reference pipeline
from heart-rate-prediction-benchmark
             │
             ├──── compare
             │
Kotlin heart-rate-signal-processing-sdk
```

Outputs should remain within defined numerical tolerances.

## PyTorch/ExecuTorch equivalence

For the same model input tensor:

```text
PyTorch model
      │
      ├──── compare predictions
      │
ExecuTorch .pte model
```

Model conversion should be validated before live deployment.

## End-to-end replay validation

```text
Recorded PPG + ACC
        ↓
heart-rate-signal-processing-sdk
        ↓
ExecuTorch
        ↓
Prediction
```

Compare the Android/Wear OS output with the Python reference pipeline.

## Live-device evaluation

Evaluate representative conditions such as:

- rest;
- sitting;
- walking;
- stairs;
- jogging;
- running;
- arm movement;
- high-motion activity.

---

# Performance Metrics

Runtime measurements may include:

```text
PPG processing latency
Tensor-generation latency
Model inference latency
End-to-end latency
CPU utilization
Memory usage
Model size
Battery impact
Rejected-window rate
```

These values should be measured and reported rather than assumed.

---

# Proposed Repository Structure

```text
wearable-biosignal-wearOS-apps/
│
├── README.md
│
└── heart-rate-prediction-app/
    │
    ├── README.md
    │
    ├── app/
    │   ├── build.gradle.kts
    │   │
    │   └── src/main/
    │       │
    │       ├── AndroidManifest.xml
    │       │
    │       └── java/.../
    │           │
    │           ├── sensor/
    │           │   ├── BiosignalSensorProvider.kt
    │           │   ├── SamsungSensorProvider.kt
    │           │   └── ReplaySensorProvider.kt
    │           │
    │           ├── buffer/
    │           │   └── SignalWindowBuffer.kt
    │           │
    │           ├── processing/
    │           │   └── HeartRateProcessingManager.kt
    │           │
    │           ├── inference/
    │           │   ├── HeartRateInferenceEngine.kt
    │           │   ├── HeartRatePrediction.kt
    │           │   └── ModelMetadata.kt
    │           │
    │           ├── ui/
    │           │   ├── HeartRateScreen.kt
    │           │   └── DebugSignalScreen.kt
    │           │
    │           └── viewmodel/
    │               └── HeartRateViewModel.kt
    │
    ├── models/
    │   └── heart_rate_model.pte
    │
    ├── docs/
    │   ├── architecture.md
    │   ├── sensor-acquisition.md
    │   ├── model-export.md
    │   ├── MODEL_CONTRACT.md
    │   ├── validation.md
    │   └── performance.md
    │
    └── tests/
```

---

# Dependency on Heart Rate Signal Processing SDK

`heart-rate-prediction-app` consumes:

**`heart-rate-signal-processing-sdk`**

from the `wearable-biosignal-sdk` repository.

After Maven Central publication, dependency management may conceptually use:

```kotlin
dependencies {
    implementation(
        "io.github.jayaram1125:heart-rate-signal-processing-sdk:<version>"
    )
}
```

The final coordinates will be determined when the SDK is published.

---

# Privacy-Oriented Architecture

The intended inference architecture is local:

```text
PPG + accelerometer
        ↓
heart-rate-signal-processing-sdk
        ↓
on-device tensor
        ↓
ExecuTorch inference
        ↓
heart-rate prediction
```

A cloud connection is not inherently required for inference.

Where practical, raw physiological data should remain transient:

```text
Sensor data
     ↓
memory buffer
     ↓
local processing
     ↓
local inference
     ↓
discard or explicitly retain
according to user/research configuration
```

---

# Development Roadmap

## Phase 1 — Application Skeleton

Develop:

- Wear OS project;
- Compose UI;
- sensor-provider abstraction;
- replay/test provider.

## Phase 2 — Sensor Acquisition

Integrate a supported raw-PPG source.

Acquire:

```text
PPG
ACC-X
ACC-Y
ACC-Z
timestamps
```

## Phase 3 — Signal Processing SDK Integration

Integrate:

**`heart-rate-signal-processing-sdk`**

and verify its outputs against the Python implementation from:

**`heart-rate-prediction-benchmark`**

## Phase 4 — Final Model Export

Export the model selected by:

**`heart-rate-prediction-benchmark`**

from PyTorch to an ExecuTorch `.pte` model.

## Phase 5 — On-Device Inference

Load the model within `heart-rate-prediction-app` and pass model-ready tensors produced by `heart-rate-signal-processing-sdk`.

## Phase 6 — User Interface

Display:

- predicted heart rate;
- signal quality;
- measurement state;
- reliability information.

## Phase 7 — Validation and Profiling

Evaluate:

- processing equivalence;
- PyTorch/ExecuTorch equivalence;
- recorded-data replay;
- live-device behavior;
- latency;
- memory;
- CPU usage;
- battery impact where practical.

## Phase 8 — Open-Source Release

Publish:

- application source;
- model metadata;
- architecture documentation;
- installation/setup instructions;
- validation results;
- runtime-performance measurements.

---

# Public Dissemination

The three related project components form a complete public development pathway:

```text
heart-rate-prediction-benchmark
        │
        │ Python research
        │ controlled experiments
        │ model evaluation
        │ final pipeline selection
        ▼
heart-rate-signal-processing-sdk
        │
        │ reusable Kotlin
        │ signal-processing implementation
        ▼
heart-rate-prediction-app
        │
        │ Wear OS reference implementation
        │ ExecuTorch inference
        │ real-device demonstration
        ▼
Researchers and developers
```

This allows third parties to inspect and reproduce the path from:

> **research → validated methodology → reusable SDK → real wearable implementation.**

---

# Research Transparency

The project will distinguish between:

### Published research results

Results reported in external scientific literature.

### Reproduced results

Results obtained through implementations reproduced by `heart-rate-prediction-benchmark`.

### Original benchmark results

Results generated by the project's controlled experiments.

### Selected production pipeline

The processing/model combination chosen based on benchmark findings.

### Wear OS runtime results

Measurements generated directly by `heart-rate-prediction-app`.

### Future work

Functionality planned but not yet implemented.

No performance or accuracy claim should be made unless supported by documented experiments.

---

# Current Limitations

## Raw PPG availability

Wear OS does not provide one universal raw-PPG interface across all devices.

Vendor-specific APIs or supported hardware may therefore be required.

## Hardware differences

Performance may vary across devices because of differences in:

- optical sensors;
- LED wavelength;
- photodiodes;
- sensor geometry;
- sampling frequency;
- analog front ends;
- watch fit;
- wrist position.

## Dataset-to-device generalization

A model that performs well using public datasets may behave differently on live wearable hardware.

Live-device evaluation is therefore an important later stage of the project.

## Clinical use

`heart-rate-prediction-app` is a research and engineering application.

It is not a clinically validated medical device.

---

# Related Projects

## heart-rate-prediction-benchmark

Python-based research and benchmarking project responsible for:

```text
public dataset integration
signal-processing experiments
tensor-generation experiments
model training and fine-tuning
model comparisons
activity-specific evaluation
subject-independent evaluation
cross-dataset validation
final pipeline/model selection
```

## heart-rate-signal-processing-sdk

Reusable Kotlin/Android SDK responsible for transforming raw PPG and motion signals into model-ready representations based on the methodology selected through `heart-rate-prediction-benchmark`.

## heart-rate-prediction-app

Wear OS reference application responsible for:

```text
live wearable sensor acquisition
        ↓
heart-rate-signal-processing-sdk
        ↓
model-ready tensor
        ↓
ExecuTorch model
        ↓
heart-rate prediction
        ↓
Wear OS user interface
```

---

# Long-Term Objective

The long-term objective is to demonstrate a reproducible path from research to real-time wearable deployment:

```text
Research question
       ↓
Public datasets
       ↓
heart-rate-prediction-benchmark
       ↓
Controlled experiments
       ↓
Signal-processing selection
       ↓
Model selection
       ↓
heart-rate-signal-processing-sdk
       ↓
ExecuTorch model export
       ↓
heart-rate-prediction-app
       ↓
Real-time on-device inference
       ↓
Public evaluation and reuse
```

The project follows the principle:

> **Research first, validate objectively, implement the selected processing pipeline as a reusable SDK, and demonstrate the resulting model through real-time on-device inference.**

---

# License

The application is intended to be released under the **Apache License 2.0**.

Third-party SDKs, models, datasets, libraries, and runtime components remain subject to their respective licenses.

---

# Disclaimer

`heart-rate-prediction-app` is provided for research, education, software development, and experimental purposes.

It is not intended to diagnose, prevent, monitor, treat, or manage any disease or medical condition and should not be used as a substitute for validated medical equipment or professional medical advice.
