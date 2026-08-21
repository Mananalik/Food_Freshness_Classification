````markdown
# Real-Time Human Activity Recognition using Wi-Fi CSI

A device-free, privacy-preserving Human Activity Recognition (HAR) system that uses Wi-Fi Channel State Information (CSI) to recognize human activities in real time without cameras or wearable sensors.

## Overview

This project detects human activity by analyzing small changes in Wi-Fi signals caused by movement between a transmitter and receiver.

The system captures CSI data using ESP32-S3 hardware, processes the signal using a multi-stage preprocessing pipeline, extracts time- and frequency-domain features, and classifies activities using a Random Forest model.

The real-time inference pipeline is exposed through a FastAPI backend using Server-Sent Events (SSE), with a web dashboard displaying the predicted activity, confidence, latency, and signal metrics.

The system recognizes five activity classes:

- Empty
- Standing
- Walking
- Jumping
- Squats

## Key Features

- Device-free and camera-free human activity recognition
- Wi-Fi CSI-based sensing using commodity ESP32-S3 hardware
- Signal denoising using Median Filtering and Butterworth Low-Pass Filtering
- PCA-based dimensionality reduction
- Time-domain and frequency-domain feature extraction
- Random Forest-based multi-class classification
- Session-level grouped validation to reduce data leakage
- Real-time inference using FastAPI and Server-Sent Events
- Temporal majority-vote smoothing
- Motion-threshold guardrails for reducing transient false predictions
- Live web dashboard with activity, confidence, latency, and signal telemetry

## System Architecture

```text
Wi-Fi Signal Source
        │
        ▼
   ESP32-S3 CSI Receiver
        │
        │ USB Serial
        ▼
  CSI Data Acquisition
        │
        ▼
   Signal Preprocessing
   ├── Median Filtering
   ├── Butterworth Low-Pass Filter
   └── PCA
        │
        ▼
  Feature Extraction
   ├── Time-Domain Features
   └── Frequency-Domain Features
        │
        ▼
    Random Forest
     Classifier
        │
        ▼
 Temporal Majority Vote
        │
        ▼
 Motion Threshold Guardrails
        │
        ▼
    FastAPI + SSE
        │
        ▼
    Web Dashboard
````

The project uses a modular four-stage architecture: CSI data acquisition, signal preprocessing, feature extraction/classification, and real-time inference/visualization.

## Hardware

The system uses:

* ESP32-S3 development boards
* Wi-Fi mobile hotspot as the signal source
* Host computer for processing and inference
* USB serial connection between the ESP32-S3 receiver and host computer

The intended operating range is approximately 1–3 meters between the transmitter and receiver in an indoor environment.

The original implementation experimented with ESP-NOW transmission, but a mobile hotspot was subsequently used to increase the CSI packet rate.

## Signal Processing Pipeline

Raw CSI amplitude data passes through three main preprocessing stages:

### 1. Median Filtering

Used to suppress artifacts caused by packet drops and other short-duration disturbances.

### 2. Butterworth Low-Pass Filtering

Used to isolate motion-related frequency components while reducing higher-frequency noise.

### 3. Principal Component Analysis

PCA reduces the dimensionality of the CSI subcarrier data while retaining the components carrying the most relevant variance.

This preprocessing pipeline was derived from the STAR framework and adapted for the project.

## Feature Engineering

Features are extracted from sliding windows of preprocessed CSI data.

The feature set contains both:

### Time-Domain Features

Statistical characteristics of the CSI signal over each window.

### Frequency-Domain Features

Frequency characteristics derived from the CSI signal, including:

* Spectral entropy
* Low-band energy ratio (0–2 Hz)
* Mid-band energy ratio (2–4 Hz)

## Machine Learning Model

The final classifier is a Random Forest model.

Random Forest was selected after evaluating alternative approaches including:

* k-Nearest Neighbours
* Support Vector Machines
* Decision Trees
* CNN + GRU based deep learning approaches

The final Random Forest configuration used 120 decision trees with Gini impurity, with hyperparameters optimized through randomized cross-validation search.

A session-level grouped split was used during validation so that windows from the same recording session could not appear in both training and validation sets. This reduces the risk of data leakage from adjacent windows and provides a more realistic estimate of generalization.

## Real-Time Inference

The real-time inference pipeline continuously receives CSI data from the ESP32-S3 receiver over USB serial.

The system:

1. Reads the incoming CSI stream.
2. Maintains a sliding window of 450 samples with a 50-sample hop.
3. Applies the fitted preprocessing pipeline.
4. Extracts features from the current window.
5. Generates a prediction using the Random Forest classifier.
6. Applies temporal majority voting to smooth transient predictions.
7. Applies motion-threshold guardrails.
8. Publishes the final prediction through a FastAPI SSE endpoint.

The majority-vote layer uses the three most recent predictions to reduce flickering during activity transitions. Static and dynamic motion thresholds are also used to constrain physically inconsistent predictions.

## Web Dashboard

The real-time dashboard displays:

* Current activity
* Prediction confidence
* Inference latency
* Signal telemetry

The dashboard receives updates through the FastAPI Server-Sent Events stream.

## Results

The final system achieved an offline validation accuracy of approximately **85% across five activity classes**.

The deployed real-time system achieved approximately **32 ms inference-to-SSE broadcast latency** under normal operating conditions.

The system maintained stable operation during live testing, with temporal smoothing helping to reduce brief misclassifications during activity transitions.

## Challenges and Solutions

### Limited CSI Sampling Rate

Standalone ESP-NOW transmission provided approximately 50–60 transmissions per second, which limited temporal feature extraction.

**Solution:** Mobile hotspot-assisted active pinging was introduced to increase the packet rate.

### Hardware Range Limitations

The ESP32-S3 hardware lacked an external high-gain antenna, causing CSI quality to degrade with distance and obstacles.

**Solution:** Testing was maintained within a controlled 1–3 meter transmitter-receiver separation.

### Deep Learning Generalization

CNN+GRU models trained on the custom dataset showed poor generalization when tested in different environments.

**Solution:** The project moved to Random Forest with engineered features and grouped session-level validation.

### Transient False Predictions

Activity transitions occasionally produced brief incorrect classifications.

**Solution:** Three-frame majority voting and motion-threshold guardrails were introduced.

## Applications

Potential applications include:

* Elderly activity monitoring
* Smart home automation
* Indoor occupancy monitoring
* Privacy-preserving sensing
* Security and intrusion detection

The project specifically explores applications in elderly care, smart home automation, and intrusion detection.

## Limitations

The current implementation is primarily designed for:

* Indoor environments
* Single-occupant scenarios
* A 1–3 meter transmitter-receiver range
* Five predefined activity classes

The report identifies cross-room testing, multi-person sensing, improved antenna hardware, and larger/more diverse datasets as future directions.

## Future Work

Planned improvements include:

* Expanding the dataset across different rooms and subjects
* Supporting multi-person activity recognition
* Testing ESP32 variants with external antenna support
* Improving cross-environment generalization
* Exploring self-supervised approaches such as AutoFi

## Technology Stack

**Hardware**

* ESP32-S3
* Wi-Fi / Mobile Hotspot
* USB Serial

**Embedded**

* ESP-IDF
* ESP-NOW

**Machine Learning**

* Python
* Scikit-learn
* Random Forest
* PCA
* Statistical feature engineering

**Backend**

* FastAPI
* Server-Sent Events (SSE)

**Signal Processing**

* Median Filtering
* Butterworth Low-Pass Filtering
* FFT-based frequency analysis

## Project Structure

```text
wifi-csi-har/
│
├── firmware/
│   └── esp32-s3/
│       └── ...
│
├── data/
│   ├── raw/
│   └── processed/
│
├── preprocessing/
│   ├── median_filter.py
│   ├── butterworth_filter.py
│   └── pca.py
│
├── features/
│   └── feature_extraction.py
│
├── model/
│   ├── train.py
│   ├── validate.py
│   └── random_forest/
│
├── inference/
│   └── realtime_inference.py
│
├── api/
│   └── main.py
│
├── dashboard/
│   └── ...
│
└── README.md
```

> **Note:** Update the directory names above to match the actual structure of your GitHub repository. The project report documents the functional architecture and source code in the appendix, but it does not provide enough information to verify these exact repository filenames/directories.

## Authors

**Manan Malik**
Vellore Institute of Technology, Chennai

**Ralf Paul Victor**
Vellore Institute of Technology, Chennai

**Saumil Singh Rana**
Vellore Institute of Technology, Chennai

## Project Report

The project was completed as part of the Bachelor of Technology in Electronics and Communication Engineering at Vellore Institute of Technology, Chennai, in April 2026.

```
```
