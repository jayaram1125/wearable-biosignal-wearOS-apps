# Blood Pressure Prediction App

A Wear OS research and reference application for experimental cuffless blood-pressure estimation using wearable photoplethysmography (PPG), motion-sensor data, signal processing, and on-device machine-learning inference.

The application demonstrates the final deployment stage of the broader open-source blood-pressure research pipeline:

```text
Public BP research datasets
        ↓
blood-pressure-prediction-benchmark
        ↓
Signal-processing experiments
        ↓
Model training / fine-tuning
        ↓
Subject-independent validation
        ↓
Cross-dataset validation
        ↓
Selected processing pipeline
        ↓
blood-pressure-signal-processing-sdk
        ↓
Selected PyTorch BP model
        ↓
ExecuTorch export
        ↓
blood-pressure-prediction-app
        ↓
Experimental on-device SBP / DBP estimation
```

> **Project status:** Research and development  
> **Platform:** Wear OS  
> **Language:** Kotlin  
> **ML runtime:** ExecuTorch  
> **Purpose:** Research, experimentation, benchmarking, and demonstration  
> **Medical use:** This application is not a medical device and is not intended to diagnose hypertension, guide treatment, replace a validated blood-pressure cuff, or support clinical decision-making.

---

# Project Location

This application is maintained inside the:

**`wearable-biosignal-wearOS-apps`**

repository.

Application module:

```text
wearable-biosignal-wearOS-apps/
└── blood-pressure-prediction-app/
```

The application depends on:

```text
wearable-biosignal-sdk/
└── blood-pressure-signal-processing-sdk/
```

and on the processing methodology and model selected through:

```text
blood-pressure-prediction-benchmark/
```

The three projects therefore have distinct responsibilities:

```text
blood-pressure-prediction-benchmark
        │
        │ research and validation
        ▼
blood-pressure-signal-processing-sdk
        │
        │ reusable signal processing
        ▼
blood-pressure-prediction-app
        │
        │ real-device deployment
        ▼
Wear OS experimental application
```

---

# Objective

Blood pressure is traditionally measured using an inflatable cuff.

Cuffless wearable blood-pressure systems attempt to estimate blood pressure indirectly from physiological signals such as PPG.

However, PPG does not directly measure arterial pressure.

PPG morphology may be affected by:

- blood pressure;
- arterial stiffness;
- vascular tone;
- temperature;
- autonomic state;
- medication;
- posture;
- sensor contact pressure;
- skin and tissue characteristics;
- motion;
- peripheral perfusion.

Therefore, cuffless BP estimation remains a challenging research problem.

The purpose of `blood-pressure-prediction-app` is to demonstrate an end-to-end experimental wearable pipeline:

> **Acquire raw wearable PPG and motion signals → apply the experimentally selected signal-processing pipeline → assess signal reliability → generate model-compatible tensors → perform on-device ML inference → display an experimental systolic/diastolic BP estimate when signal quality is sufficient.**

The application does not independently decide which preprocessing algorithm or machine-learning model is best.

Those decisions originate from controlled experiments in:

**`blood-pressure-prediction-benchmark`**

---

# Research-to-Deployment Architecture

```text
┌─────────────────────────────────┐
│ blood-pressure-prediction-      │
│ benchmark                       │
│                                 │
│ Public BP datasets              │
│ Signal-processing experiments   │
│ Representation experiments      │
│ Model training / fine-tuning    │
│ Subject-independent validation  │
│ Cross-dataset validation        │
└────────────────┬────────────────┘
                 │
                 │ selects
                 ▼
┌─────────────────────────────────┐
│ Reference Processing Pipeline   │
│ + Selected PyTorch Model        │
└───────────────┬─────────────────┘
                │
       ┌────────┴────────┐
       │                 │
       ▼                 ▼
blood-pressure-      ExecuTorch
signal-processing-   model export
sdk                   (.pte)
       │                 │
       └────────┬────────┘
                ▼
┌─────────────────────────────────┐
│ blood-pressure-prediction-app   │
│                                 │
│ Wearable sensor acquisition     │
│ Signal processing               │
│ Reliability assessment          │
│ Tensor generation               │
│ ExecuTorch inference            │
│ Wear OS UI                      │
└─────────────────────────────────┘
```

---

# Wearable Sensor Acquisition

## Important Platform Limitation

Raw PPG is not exposed through one universal sensor interface across every Wear OS device.

The application should therefore use a sensor-provider abstraction.

The initial implementation is expected to target compatible **Samsung Galaxy Watch** devices using the **Samsung Health Sensor SDK**.

On supported Galaxy Watch hardware, Samsung provides access to raw:

```text
Green PPG
Red PPG
Infrared PPG

Accelerometer X
Accelerometer Y
Accelerometer Z
```

The final input channels used by the application will depend on the processing/model configuration selected in `blood-pressure-prediction-benchmark`.

---

# Initial Samsung Sensor Configuration

The current Samsung Health Sensor SDK supports continuous raw acquisition of:

```text
PPG:
Green
Red
Infrared
25 Hz

Accelerometer:
X
Y
Z
25 Hz
```

The research model may require a different sampling frequency.

Therefore the processing SDK may need to perform:

```text
Wearable sensor sampling rate
        ↓
validated resampling
        ↓
model-required sampling rate
```

Resampling behavior must match the model contract and must be experimentally validated.

---

# Sensor Provider Abstraction

The rest of the application should not depend directly on Samsung-specific APIs.

Conceptually:

```kotlin
interface BiosignalSensorProvider {

    fun start()

    fun stop()

    fun observePpg(): Flow<PpgSample>

    fun observeAccelerometer(): Flow<AccelerometerSample>
}
```

Possible implementations:

```text
BiosignalSensorProvider
        │
        ├── SamsungSensorProvider
        │
        ├── ReplaySensorProvider
        │
        └── FutureSensorProvider
```

This allows future support for:

- additional wearable devices;
- research sensors;
- recorded PPG datasets;
- alternative vendor APIs.

---

# Replay Sensor Provider

A replay provider is especially important for testing.

Instead of requiring live watch data for every test:

```text
Recorded PPG + ACC
        ↓
ReplaySensorProvider
        ↓
same runtime pipeline
        ↓
SDK
        ↓
ExecuTorch
        ↓
BP estimate
```

This enables reproducible Android regression testing using known input signals.

---

# Complete Runtime Architecture

```text
┌──────────────────────────────────┐
│          Wear OS Device          │
│                                  │
│ PPG sensor        Accelerometer  │
└───────┬─────────────────┬────────┘
        │                 │
        ▼                 ▼
┌──────────────────────────────────┐
│       Sensor Acquisition         │
│                                  │
│ PPG samples                      │
│ ACC-X / Y / Z                    │
│ timestamps                       │
│ sampling metadata                │
└────────────────┬─────────────────┘
                 │
                 ▼
┌──────────────────────────────────┐
│       Signal Window Buffer       │
│                                  │
│ synchronize                      │
│ collect fixed window             │
│ detect missing samples           │
└────────────────┬─────────────────┘
                 │
                 ▼
┌──────────────────────────────────┐
│ blood-pressure-signal-           │
│ processing-sdk                   │
│                                  │
│ input validation                 │
│ resampling                       │
│ filtering                        │
│ morphology processing            │
│ motion assessment                │
│ signal-quality assessment        │
│ normalization                    │
│ tensor generation                │
└────────────────┬─────────────────┘
                 │
                 ▼
┌──────────────────────────────────┐
│        Reliability Gate          │
│                                  │
│ Is signal suitable for BP        │
│ inference?                       │
└──────────────┬───────────────────┘
               │
         ┌─────┴──────┐
         │            │
        YES           NO
         │            │
         ▼            ▼
   Model tensor    Suppress
         │        prediction
         ▼
┌──────────────────────────────────┐
│        ExecuTorch Runtime        │
│                                  │
│ Load .pte model                  │
│ Create model input               │
│ Execute inference                │
└────────────────┬─────────────────┘
                 │
                 ▼
┌──────────────────────────────────┐
│ Experimental BP Output           │
│                                  │
│ SBP                              │
│ DBP                              │
│ quality / reliability            │
│ model metadata                   │
└────────────────┬─────────────────┘
                 │
                 ▼
┌──────────────────────────────────┐
│            Wear OS UI            │
└──────────────────────────────────┘
```

---

# Relationship to `blood-pressure-signal-processing-sdk`

Raw wearable signals are passed to:

**`blood-pressure-signal-processing-sdk`**

The application should not duplicate the selected BP preprocessing logic.

Instead:

```text
blood-pressure-prediction-app
        ↓
blood-pressure-signal-processing-sdk
        ↓
model-ready tensor
```

The SDK is responsible for converting wearable signals into exactly the representation expected by the selected model.

---

# Signal Processing Responsibilities

Depending on the results from `blood-pressure-prediction-benchmark`, the SDK may perform:

```text
Raw PPG
   ↓
input validation
   ↓
timestamp synchronization
   ↓
resampling
   ↓
detrending
   ↓
selected filtering
   ↓
motion-artifact analysis
   ↓
signal-quality evaluation
   ↓
morphology extraction
   ↓
selected representation
   ↓
normalization
   ↓
model tensor
```

Possible experimentally selected representations include:

```text
PPG

PPG + VPG

PPG + VPG + APG

PPG + accelerometer

PPG + morphology + motion

CWT representation

STFT representation
```

The application itself should not choose among these at runtime unless the selected model was explicitly designed to support multiple pipelines.

---

# Training–Deployment Consistency

This is a critical requirement.

> **The signal processing used by the Wear OS application must reproduce the processing used during model training.**

Suppose the selected model in `blood-pressure-prediction-benchmark` was trained with:

```text
10-second PPG window
        ↓
resampled to target frequency
        ↓
Butterworth filter
        ↓
PPG first derivative
        ↓
PPG second derivative
        ↓
training-set normalization
        ↓
[PPG, VPG, APG] tensor
```

The runtime application must generate the same representation.

It must not use:

```text
different filter
different normalization
different channel order
different sampling rate
different window length
```

unless the model has been retrained for those conditions.

---

# Model Contract

The interface between:

```text
blood-pressure-signal-processing-sdk
```

and:

```text
ExecuTorch BP model
```

should be defined in:

**`MODEL_CONTRACT.md`**

The contract should specify:

- model version;
- required SDK version;
- processing-pipeline version;
- input sampling rate;
- window length;
- window stride;
- channel count;
- channel ordering;
- input units;
- normalization;
- tensor dimensions;
- tensor datatype;
- calibration requirements;
- expected output format;
- output units;
- reliability requirements.

Example only:

```text
Input tensor:
[1, 3, 1250]

Channel 0:
PPG

Channel 1:
VPG

Channel 2:
APG

Datatype:
float32

Outputs:
SBP
DBP
```

The actual values must come from the final benchmark configuration.

---

# Model Development

The BP model is developed and selected in:

**`blood-pressure-prediction-benchmark`**

The research workflow is:

```text
Public BP datasets
        ↓
Signal-processing candidates
        ↓
Representation candidates
        ↓
Model candidates
        ↓
Training / fine-tuning
        ↓
Subject-independent testing
        ↓
Calibration analysis
        ↓
Cross-dataset validation
        ↓
Motion / reliability analysis
        ↓
Computational evaluation
        ↓
Selected model
```

Candidate model families may include:

- XGBoost;
- 1-D CNN;
- TCN;
- CNN-GRU;
- CNN-BiLSTM;
- U-Net waveform reconstruction;
- Transformer-based approaches;
- fine-tuned PPG foundation models.

The Wear OS application should deploy the model selected through those experiments rather than independently choosing a model.

---

# PyTorch to ExecuTorch Deployment

The selected PyTorch model will be prepared for on-device inference using ExecuTorch.

Conceptually:

```text
Final model from
blood-pressure-prediction-benchmark
        ↓
PyTorch model
        ↓
model export
        ↓
ExecuTorch lowering / preparation
        ↓
ExecuTorch exported program
        ↓
.pte model artifact
        ↓
blood-pressure-prediction-app
```

The exported model should be versioned.

Example:

```text
models/
└── blood_pressure/
    ├── metadata.json
    └── blood_pressure_model_v1.pte
```

---

# Model Metadata

The model should include metadata describing its expected runtime environment.

Conceptually:

```json
{
  "modelVersion": "1.0.0",
  "processingPipeline": "bp-pipeline-v1",
  "requiredSdkVersion": "1.x",
  "calibrationRequired": false,
  "outputs": [
    "systolic",
    "diastolic"
  ]
}
```

This helps prevent accidentally pairing a model with an incompatible SDK processing pipeline.

---

# ExecuTorch Runtime

The application will use ExecuTorch to perform local model inference on the Wear OS device.

The runtime flow is:

```text
Load BP .pte model
        ↓
Receive tensor from
blood-pressure-signal-processing-sdk
        ↓
Construct ExecuTorch input tensor
        ↓
Execute model
        ↓
Read output
        ↓
SBP / DBP
```

The initial implementation should prioritize compatibility and correctness before hardware-specific optimization.

---

# Conceptual Kotlin Inference Engine

The implementation may conceptually resemble:

```kotlin
class BloodPressureInferenceEngine(
    private val modelPath: String
) {

    private val module = Module.load(modelPath)

    fun predict(
        input: BloodPressureModelInput
    ): BloodPressurePrediction {

        val tensor = Tensor.fromBlob(
            input.values,
            input.shape
        )

        val output = module.forward(
            EValue.from(tensor)
        )

        val values =
            output[0]
                .toTensor()
                .getDataAsFloatArray()

        return BloodPressurePrediction(
            systolic = values[0],
            diastolic = values[1]
        )
    }
}
```

This illustrates the intended architecture only.

The final implementation must follow the ExecuTorch API version selected for the project.

---

# PyTorch / ExecuTorch Equivalence

Model conversion must be validated before live wearable use.

For identical tensor input:

```text
Python / PyTorch model
          │
          ├──── compare outputs
          │
ExecuTorch .pte model
```

The SBP and DBP outputs should remain within a documented numerical tolerance.

Any significant difference must be investigated before deployment.

---

# Python / Kotlin Processing Equivalence

The signal-processing implementation must also be validated.

```text
Raw input window
       │
       ├───────────────────┐
       │                   │
       ▼                   ▼
Python reference      Kotlin SDK
       │                   │
       ▼                   ▼
Reference tensor      Runtime tensor
       │                   │
       └─────────┬─────────┘
                 ▼
             Compare
```

Validation should include:

- clean signals;
- noisy signals;
- motion-corrupted signals;
- missing samples;
- abnormal sampling intervals;
- boundary cases.

---

# Device-Domain Generalization

This is one of the most important limitations of the project.

A model may have been trained using:

```text
clinical fingertip PPG
```

while the watch provides:

```text
wrist PPG
```

These are not necessarily equivalent.

Differences may include:

- sensor placement;
- LED wavelength;
- optical geometry;
- photodetector;
- sampling rate;
- skin contact;
- motion;
- signal amplitude;
- waveform morphology.

Therefore:

> **Successful benchmark performance does not automatically establish that the model will perform accurately on live Galaxy Watch PPG.**

Live-device evaluation must be treated as a separate validation stage.

If necessary, future work may include:

- wearable-domain fine-tuning;
- transfer learning;
- device-specific normalization;
- domain adaptation;
- model recalibration.

No claim of real-world BP accuracy should be made until validated against an appropriate reference measurement.

---

# Reliability Gate

Unlike a conventional application that always returns a number, `blood-pressure-prediction-app` should be designed to allow:

> **No reliable measurement available.**

For example:

```text
PPG window
      ↓
blood-pressure-signal-processing-sdk
      ↓
Signal Quality = 0.19
Motion = HIGH
      ↓
Reliability Gate
      ↓
DO NOT RUN BP INFERENCE
```

or:

```text
PPG window
      ↓
Signal Quality = 0.91
Motion = LOW
      ↓
Reliability Gate
      ↓
Run model
```

This helps prevent apparently plausible BP values from being displayed when physiological input is inadequate.

---

# Proposed Prediction States

Possible application states:

```text
WAITING_FOR_SENSOR

COLLECTING_SIGNAL

BUILDING_WINDOW

PROCESSING_SIGNAL

POOR_SIGNAL

EXCESSIVE_MOTION

RUNNING_INFERENCE

PREDICTION_AVAILABLE

MODEL_ERROR

SENSOR_ERROR
```

The UI should clearly distinguish:

```text
measurement unavailable
```

from:

```text
measured low/high BP
```

These are fundamentally different conditions.

---

# Proposed User Interface

A minimal research UI may resemble:

```text
┌─────────────────────────────┐
│     Blood Pressure          │
│                             │
│        122 / 78             │
│          mmHg               │
│                             │
│ Signal Quality              │
│ ● Good                      │
│                             │
│ Motion                      │
│ ● Low                       │
│                             │
│ Research Estimate           │
└─────────────────────────────┘
```

For unreliable signal:

```text
┌─────────────────────────────┐
│     Blood Pressure          │
│                             │
│          -- / --            │
│                             │
│ Signal Quality              │
│ ● Poor                      │
│                             │
│ Excessive movement          │
│ Try again when still        │
└─────────────────────────────┘
```

The app should not invent a BP value when reliability requirements are not met.

---

# Research / Debug UI

Development builds may expose:

- raw PPG;
- filtered PPG;
- VPG;
- APG;
- accelerometer magnitude;
- signal-quality score;
- motion score;
- model tensor dimensions;
- SDK pipeline version;
- model version;
- processing latency;
- inference latency.

For example:

```text
PPG Quality       0.87
Motion Score      0.12
Window            10.0 s
Processing        18 ms
Inference         27 ms
Model             bp-v1.0
SDK Pipeline      bp-pipeline-v1
```

These values are illustrative only.

---

# Calibration Support

The selected model may be:

```text
calibration-free
```

or:

```text
calibration-based
```

The application architecture should support either without assuming one in advance.

## Calibration-Free Flow

```text
PPG
 ↓
SDK
 ↓
tensor
 ↓
model
 ↓
SBP / DBP
```

## Calibration-Based Flow

```text
Reference cuff BP
        ↓
Calibration state
        +
current PPG
        ↓
SDK / model
        ↓
SBP / DBP
```

If calibration is required, the application must clearly distinguish:

```text
reference cuff measurement
```

from:

```text
predicted BP
```

Calibration data must be stored and handled deliberately.

---

# Calibration UI

A future calibrated model may require a setup flow such as:

```text
Step 1
Wear watch correctly

Step 2
Remain still

Step 3
Take validated upper-arm cuff BP

Step 4
Enter reference SBP / DBP

Step 5
Collect corresponding PPG window

Step 6
Create calibration state
```

This feature should only be implemented if the selected benchmark model actually requires subject-specific calibration.

---

# Ground Truth

The Galaxy Watch's own BP or HR output should not automatically be considered scientific ground truth for the custom model.

Primary model evaluation should use datasets or experiments with appropriate reference BP such as:

- invasive arterial BP;
- validated cuff BP;
- validated reference systems.

The live Wear OS application primarily demonstrates:

- sensor acquisition;
- runtime signal processing;
- reliability assessment;
- tensor generation;
- on-device inference;
- computational feasibility.

A separate validation protocol is required before making claims about live BP accuracy.

---

# Application Validation

## 1. Sensor Acquisition Validation

Verify:

- expected sensor channels;
- sampling frequency;
- timestamp consistency;
- missing samples;
- dropped events;
- watch-off-wrist behavior where detectable.

---

## 2. Signal-Processing Equivalence

Compare:

```text
blood-pressure-prediction-benchmark
Python pipeline
        │
        ├── same input
        │
blood-pressure-signal-processing-sdk
Kotlin pipeline
```

---

## 3. Model Conversion Equivalence

Compare:

```text
PyTorch
   │
   ├── same tensor
   │
ExecuTorch
```

for:

- SBP;
- DBP.

---

## 4. Recorded Dataset Replay

Use known PPG windows:

```text
Recorded data
     ↓
ReplaySensorProvider
     ↓
SDK
     ↓
ExecuTorch
     ↓
Prediction
```

Compare against the benchmark reference result.

---

## 5. Live Wearable Testing

Evaluate signal behavior under conditions such as:

```text
seated rest

standing

walking

arm movement

posture changes

higher motion
```

The objective at this stage is initially to evaluate:

- signal quality;
- pipeline stability;
- runtime feasibility;
- failure modes.

Do not automatically interpret live predictions as validated BP measurements.

---

# Motion Handling

The app should explicitly monitor motion.

Example:

```text
ACC-X/Y/Z
   ↓
motion score
   ↓
┌──────────────┐
│ LOW          │
│ MODERATE     │
│ HIGH         │
└──────────────┘
```

The experimentally selected reliability logic may define:

```text
LOW
→ inference allowed

MODERATE
→ inference allowed only if PPG quality sufficient

HIGH
→ suppress prediction
```

The actual thresholds must be derived experimentally.

---

# Posture Considerations

Wrist position can influence BP-related measurements because the wrist may be above or below heart level.

Future research may therefore incorporate:

- accelerometer orientation;
- posture estimation;
- arm position;
- measurement instructions.

The initial application should document posture during test measurements where possible.

---

# Privacy-Oriented Architecture

The intended inference path is local:

```text
PPG + ACC
    ↓
On-device signal processing
    ↓
On-device tensor generation
    ↓
ExecuTorch
    ↓
Experimental BP estimate
```

A cloud connection is not inherently required for model inference.

Where practical:

```text
raw physiological data
        ↓
memory buffer
        ↓
processing
        ↓
inference
        ↓
discard
```

should be preferred unless the user explicitly enables research-data storage.

Any future cloud synchronization or research-data upload should require clear documentation and user consent.

---

# Runtime Performance

The application should measure:

```text
sensor buffering latency

signal-processing latency

tensor-generation latency

ExecuTorch inference latency

end-to-end latency

CPU utilization

memory usage

model size

battery impact

prediction coverage

rejected-window rate
```

The results should be measured rather than assumed.

---

# Proposed Repository Structure

```text
wearable-biosignal-wearOS-apps/
│
├── README.md
│
└── blood-pressure-prediction-app/
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
    │           │   └── BloodPressureProcessingManager.kt
    │           │
    │           ├── inference/
    │           │   ├── BloodPressureInferenceEngine.kt
    │           │   ├── BloodPressurePrediction.kt
    │           │   └── ModelMetadata.kt
    │           │
    │           ├── calibration/
    │           │   ├── CalibrationState.kt
    │           │   └── CalibrationManager.kt
    │           │
    │           ├── reliability/
    │           │   └── PredictionGate.kt
    │           │
    │           ├── ui/
    │           │   ├── BloodPressureScreen.kt
    │           │   ├── CalibrationScreen.kt
    │           │   └── DebugSignalScreen.kt
    │           │
    │           └── viewmodel/
    │               └── BloodPressureViewModel.kt
    │
    ├── models/
    │   └── blood_pressure_model.pte
    │
    ├── docs/
    │   ├── architecture.md
    │   ├── sensor-acquisition.md
    │   ├── MODEL_CONTRACT.md
    │   ├── model-export.md
    │   ├── calibration.md
    │   ├── reliability.md
    │   ├── validation.md
    │   └── performance.md
    │
    └── tests/
```

The exact Gradle/project layout may evolve during implementation.

---

# Dependency on `blood-pressure-signal-processing-sdk`

`blood-pressure-prediction-app` will consume:

**`blood-pressure-signal-processing-sdk`**

from the parent `wearable-biosignal-sdk` project.

After the SDK is published to Maven Central, integration may conceptually resemble:

```kotlin
dependencies {
    implementation(
        "io.github.jayaram1125:blood-pressure-signal-processing-sdk:<version>"
    )
}
```

The actual Maven coordinates will be defined when the SDK is published.

---

# ExecuTorch Dependency

ExecuTorch provides Android Java/Kotlin bindings through an Android AAR and is available through Maven Central.

Conceptually:

```kotlin
dependencies {
    implementation(
        "org.pytorch:executorch-android:<tested-version>"
    )
}
```

The project should pin a tested ExecuTorch version rather than automatically using an unspecified latest version.

---

# Samsung Health Sensor SDK

The initial Galaxy Watch implementation may use Samsung Health Sensor SDK for raw PPG and accelerometer acquisition.

The sensor-provider architecture should isolate this dependency inside:

```text
SamsungSensorProvider
```

rather than spreading Samsung-specific APIs throughout the application.

This makes later migration or additional-device support easier.

---

# Distribution Considerations

Samsung Health Sensor SDK development can be tested using developer mode on supported Galaxy Watch hardware.

Public application distribution using Samsung Health Sensor SDK may require Samsung's partner/distribution approval process.

Therefore:

> Successful local development does not automatically guarantee unrestricted public distribution of raw-PPG sensor access.

The research repository and replay mode should remain usable independently of public store distribution.

---

# Development Roadmap

## Phase 1 — Wear OS Application Skeleton

Implement:

- Wear OS application;
- Compose UI;
- application state;
- sensor-provider interface;
- replay provider.

---

## Phase 2 — Live Sensor Acquisition

Integrate:

```text
SamsungSensorProvider
```

for compatible Galaxy Watch hardware.

Acquire:

```text
PPG Green / selected PPG channels

ACC-X

ACC-Y

ACC-Z

timestamps
```

---

## Phase 3 — Signal Processing SDK Integration

Integrate:

**`blood-pressure-signal-processing-sdk`**

Verify its Kotlin output against the Python reference implementation from:

**`blood-pressure-prediction-benchmark`**

---

## Phase 4 — Model Export

Take the final experimentally selected PyTorch model from:

**`blood-pressure-prediction-benchmark`**

and export it to an ExecuTorch `.pte` model.

---

## Phase 5 — Model Integration

Load the `.pte` model within:

**`blood-pressure-prediction-app`**

Pass tensors produced by:

**`blood-pressure-signal-processing-sdk`**

to the model.

---

## Phase 6 — Reliability Gate

Before model inference:

```text
check PPG quality
      +
check motion
      +
check required calibration
      ↓
allow or suppress inference
```

---

## Phase 7 — User Interface

Display:

- experimental systolic BP;
- experimental diastolic BP;
- signal quality;
- motion state;
- measurement state;
- reliability status.

---

## Phase 8 — Runtime Validation

Perform:

- Python/Kotlin processing equivalence;
- PyTorch/ExecuTorch model equivalence;
- recorded-dataset replay;
- live-device testing;
- latency profiling;
- memory profiling;
- reliability analysis.

---

## Phase 9 — Device-Domain Evaluation

Evaluate how the selected research model behaves on real watch PPG.

If significant domain shift exists, investigate:

- fine-tuning;
- transfer learning;
- normalization;
- device-specific calibration;
- domain adaptation.

---

## Phase 10 — Open-Source Release

Publish:

- application source;
- architecture documentation;
- model metadata;
- model contract;
- setup instructions;
- validation methodology;
- reproducible runtime results;
- known limitations.

---

# Public Dissemination

The complete open-source BP project is intended to provide a transparent research-to-deployment path:

```text
blood-pressure-prediction-benchmark
        │
        │ public research
        │ reproducible experiments
        │ model selection
        ▼
blood-pressure-signal-processing-sdk
        │
        │ reusable Kotlin implementation
        │ signal quality
        │ tensor generation
        ▼
blood-pressure-prediction-app
        │
        │ Wear OS reference application
        │ ExecuTorch inference
        │ real-device experimentation
        ▼
Researchers / developers
```

This enables third parties to inspect and reproduce:

> **research → preprocessing selection → model selection → reusable SDK → wearable implementation.**

---

# Research Transparency

The project will distinguish between:

## Published Research Results

Results reported by external researchers.

## Reproduced Results

Results obtained by reproducing published methods.

## Original Benchmark Results

Results generated by:

**`blood-pressure-prediction-benchmark`**

## SDK Validation Results

Numerical validation of:

**`blood-pressure-signal-processing-sdk`**

against the Python reference pipeline.

## ExecuTorch Validation Results

Comparison between:

```text
PyTorch model
```

and:

```text
ExecuTorch model
```

## Wearable Runtime Results

Measurements produced using:

**`blood-pressure-prediction-app`**

## Clinical Accuracy

Not claimed unless independently established through an appropriate clinical-validation process.

---

# Current Limitations

## PPG is an indirect BP signal

PPG does not directly measure arterial pressure.

## Raw PPG availability

Wear OS does not expose raw PPG through one universal interface across every device.

## Sensor-domain differences

Public datasets and smartwatch PPG can differ substantially.

## Model generalization

A model performing well on public datasets may not perform equally well on live watch data.

## Motion

The application may suppress prediction when motion or poor signal quality makes inference unreliable.

## Calibration

Some selected models may require user-specific cuff calibration.

## Ground Truth

The app itself does not create medical-grade ground truth.

## Clinical Use

This application is not intended for diagnosis or treatment.

---

# Related Projects

## `blood-pressure-prediction-benchmark`

Responsible for:

```text
public BP datasets
        ↓
signal-processing experiments
        ↓
representation experiments
        ↓
model training / fine-tuning
        ↓
calibration evaluation
        ↓
subject-independent testing
        ↓
cross-dataset validation
        ↓
motion / reliability analysis
        ↓
final model selection
```

---

## `blood-pressure-signal-processing-sdk`

Responsible for:

```text
raw PPG / motion data
        ↓
validated preprocessing
        ↓
signal-quality assessment
        ↓
selected representation
        ↓
model-ready tensor
```

---

## `blood-pressure-prediction-app`

Responsible for:

```text
live Wear OS signals
        ↓
blood-pressure-signal-processing-sdk
        ↓
reliability gate
        ↓
model tensor
        ↓
ExecuTorch BP model
        ↓
experimental SBP / DBP
        ↓
Wear OS UI
```

---

# Long-Term Objective

The long-term objective of `blood-pressure-prediction-app` is to demonstrate a complete and reproducible path from cuffless blood-pressure research to real-time wearable deployment:

```text
Research question
       ↓
Public BP datasets
       ↓
blood-pressure-prediction-benchmark
       ↓
Controlled experiments
       ↓
Selected signal-processing methodology
       ↓
Selected model
       ↓
blood-pressure-signal-processing-sdk
       ↓
ExecuTorch export
       ↓
blood-pressure-prediction-app
       ↓
Wearable runtime experimentation
       ↓
Public evaluation and reuse
```

The project follows the principle:

> **Research first, validate on unseen subjects and independent datasets, preserve physiologically meaningful signal information, reject unreliable measurements when necessary, and only then deploy the selected methodology on a wearable device.**

---

# License

The application is intended to be released under the:

**Apache License 2.0**

Third-party SDKs, datasets, models, runtimes, and dependencies remain subject to their respective licenses.

---

# Disclaimer

`blood-pressure-prediction-app` is provided solely for research, education, software development, and experimental purposes.

It is not a medical device.

It has not been clinically validated for diagnosing, monitoring, treating, or managing hypertension or any other medical condition.

Predictions generated by this application should not be used to make medical, medication, treatment, or emergency-care decisions.

Use a clinically validated blood-pressure measurement device and qualified healthcare professional for medical decisions.
