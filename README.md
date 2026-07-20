# BFA (Business Flow Analyzer)

## Overview
In this project, you'll build **BFA (Business Flow Analyzer)**, a high-performance, edge-optimized computer vision utility implemented in **Go**, using native **OpenCV bindings (GoCV)**. This system is designed to run on resource-constrained embedded Linux environments to detect specific spatial and transactional anomalies within localized physical zones, without relying on heavy cloud computing or full-scale deep learning models during the initial verification phase.

Inspired by real-time industrial telemetry tools, this project introduces you to low-level memory management in Go/CGO, digital signal processing (DSP) for video streams, geometric Region of Interest (ROI) manipulation, and highly concurrent, non-blocking event architectures.

## Role Play
You are an embedded software and systems engineer tasked with deploying an automated monitoring and event-telemetry system inside a harsh industrial environment (subject to constant vibrations, shifting ambient light, and dust). Your goal is to build a rock-solid, zero-memory-leak video pipeline capable of isolating a specific operational workspace and raising instant alerts when unauthorized or anomalous physical interactions occur within that zone.

## Learning Objectives
* Master manual memory allocation and CGO lifecycle management within Go's garbage-collected runtime.
* Implement real-time digital image manipulation (Grayscale, Gaussian Blur, Thresholding).
* Handle spatial geometry, coordinate safety, and Region of Interest (ROI) slicing.
* Leverage Go's concurrency primitives (`goroutines`, `channels`) to decouple heavy processing from video acquisition and I/O operations.
* Build strict benchmarking systems to measure execution latency per frame ($ms/\text{frame}$).

---

## Core Requirements
Your AFM engine must be written in Go, compile natively, and execute through a Command Line Interface (CLI). It must perform the following operational phases sequentially on an incoming video stream or local camera hardware:

### 1. The CGO Memory Fortress & Stream Capture
*   Initialize and bind to a local hardware video device (e.g., `/dev/video0`) or parse a static test video file.
*   Enforce a zero-leak processing loop. Because OpenCV structures ($\text{cv::Mat}$) are allocated on the C-heap, you must manually orchestrate their allocation and deallocation lifecycle at every single iteration.
*   **Constraint:** You cannot rely on Go’s automatic garbage collection for any OpenCV matrix data.

### 2. Geometric Slicing & Boundary Safety (The ROI Engine)
*   Define a static rectangular box — the **Region of Interest (ROI)** — representing the strict physical perimeter to monitor.
*   Extraxt this dynamic sub-matrix from the main video frame without copying the entire image data in memory unless necessary.
*   Draw visual feedback overlays (bounding boxes and dynamic status text) on the primary display window.
*   **Constraint:** Implement a geometry validation layer. If the box coordinates exceed the camera’s frame boundaries (e.g., a 700x500 box on a 640x480 stream), the system must safely recut and scale the coordinates instead of causing a memory segmentation fault (`SIGSEGV`).

### 3. DSP Pipeline & Environmental Filtering
*   Convert the extracted ROI slice to a single-channel **Grayscale** matrix to eliminate redundant color data and save CPU cycles.
*   Apply a **Gaussian Blur** filter to reduce electronic sensor noise, pixel jitter caused by industrial machinery vibrations, and airborne dust artifacts.
*   Apply a **Binary Thresholding** operation to compress the matrix into strict binary format (absolute black $0$ or absolute white $255$).
*   **Constraint:** The binarization threshold must be tuned to discard gradual ambient lighting changes (shadows from outdoor structures or overhead lights) while sharply capturing physical volumes entering the zone.

### 4. Matrix Temporal Differencing (The Analytics Layer)
*   Maintain a local temporal memory state holding the cleaned matrix from the previous frame ($T_{-1}$) to compare it to the current frame ($T$).
*   Execute an **Absolute Difference** operation between $T_{-1}$ and $T$ to isolate pixels that changed state.
*   Count the non-zero (white) pixels in the resulting difference matrix. If the density of modified pixels crosses a predefined percentage threshold (representing an external volume breaching the area), trigger an internal state mutation: `ANOMALY_DETECTED`.

### 5. Concurrent I/O Isolation
*   When an anomaly state is reached, the system must invoke an alert propagation function (simulating a network JSON POST request or a local disk write log).
*   **Constraint:** This I/O invocation must run inside an isolated, non-blocking asynchronous context. The main video loop must never drop below its target processing rate because of network latency or slow storage operations.

---

## Evaluation Criteria & Stress Tests
Your submission will be evaluated based on two strict production metrics: **Stability** and **Throughput**.

### Test 1: The Infinite Loop Ram Test (Stability)
*   **Execution:** Run your AFM application continuously against a looping video file for a minimum of 15 minutes.
*   **Measurement:** Monitor the process RSS (Resident Set Size) memory using tools like `htop` or memory profilers.
*   **Passing Criteria:** The RAM profile graph must remain **perfectly flat** (e.g., locked at 40MB without a single kilobyte of variance). Any linear growth in memory consumption indicates a CGO leak, resulting in an automatic failure.

### Test 2: Edge Performance Profile (Throughput)
*   **Execution:** Inject high-frequency physical movement into the ROI box to trigger rapid, consecutive anomalies while measuring the time delta per frame loop using Go's `time.Since()`.
*   **Measurement:** Print the average processing execution time per frame in milliseconds to the console.
*   **Passing Criteria:** The processing latency must remain stable and sit comfortably under **$10\text{ ms}$ per frame** (allowing a clean $100\text{ fps}$ processing capacity). If triggering an anomaly causes the latency to spike or causes video stuttering, the concurrency model is flawed.

---

## Expected Output Behavior

```text
student$ ./afm --source=/dev/video0 --roi="200,150,240,180"
[SYS] Initializing Video Device: /dev/video0... OK
[SYS] Allocating Matrix Buffers... OK
[PERF] Pipeline target baseline: < 10ms/frame
[ENGINE] Loop started. Press 'ESC' in the window to terminate.

Frame: 00452 | OpenCV Latency: 2.34ms | Status: ZONE_OK
Frame: 00453 | OpenCV Latency: 2.11ms | Status: ZONE_OK
Frame: 00454 | OpenCV Latency: 2.45ms | Status: ZONE_OK
[EVENT] 🚨 ANOMALY: Spatial intrusion detected! Pixel variance: 14.8%
[CONCURRENCY] Spawning isolated event worker #001... OK
Frame: 00455 | OpenCV Latency: 2.51ms | Status: ANOMALY_ALERT
Frame: 00456 | OpenCV Latency: 2.22ms | Status: ANOMALY_ALERT
[WORKER #001] Dispatched remote telemetry log successfully (Status 200).
Frame: 00457 | OpenCV Latency: 2.19ms | Status: ZONE_OK
