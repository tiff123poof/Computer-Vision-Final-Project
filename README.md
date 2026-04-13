# Single-Camera Bird’s-Eye View (BEV) Scene Reconstruction with Object Detection

## Overview

This project builds a pipeline that takes images captured while rotating in place and reconstructs a scene in a bird’s-eye view (BEV). The goal is to approximate a surround-view system using only a single uncalibrated camera, without depth information or multi-camera calibration.

The main idea is to first stitch the images into a panorama, detect objects in the original images, and then map those detections into a top-down representation of the scene.

---

### Pipeline

1. Load images  
2. Build panorama (two methods)  
3. Detect objects (YOLO)  
4. Map detections to panorama  
5. Deduplicate detections  
6. Generate BEV map and place objects  

---

## Panorama Stitching

### Attempt #1: Cylindrical Warp (Custom)

In this approach, we manually construct a panorama by projecting each image onto a cylindrical surface and aligning them using feature matching.

- Each image is warped using a cylindrical projection based on an estimated focal length  
- Feature matching is used to estimate relative offsets between images  
- Offsets are accumulated to place all images into a shared coordinate system  
- Images are blended together using distance based weighting, which gives higher weight to the center of each image and reduces blurry blending at the edges.  

Because this method is fully manual, several issues had to be addressed:

- Vertical drift: small alignment errors caused the panorama to gradually shift up/down -> fixed using linear drift correction  
- Black borders: warping creates empty regions -> removed using cropping  
- Wraparound overlap: first and last images often overlap -> detected and trimmed  
- Slanted panoramas: misalignment created a parallelogram shape -> corrected with a perspective warp  
- Incorrect orientation: panoramas could be slightly rotated -> fixed by aligning dominant horizontal lines  

This method is more robust when automatic stitching fails, but it is less geometrically accurate.

---

### Attempt #2: OpenCV Stitcher

This approach uses OpenCV’s built-in stitching pipeline, which handles:

- feature detection and matching  
- homography estimation  
- image warping and blending  

To improve reliability:

- Stitching is run on both:
  - original images  
  - preprocessed images (Gaussian blur + CLAHE for better feature detection)  
- If stitching all images fails, smaller overlapping subsets are stitched and merged  

---

### Selecting the Best Panorama

To determine which result to use:

- Compute the aspect ratio (width / height)  
- A complete panorama should be significantly wider than it is tall  

Selection logic:
- If either OpenCV result passes the ratio threshold -> keep the better one  
- If both fail -> fall back to the cylindrical panorama  

Cylindrical results are cleaned before use (cropping, trimming, alignment).

---

## Object Detection

Object detection is performed using a pretrained YOLO model:

- Model: `yolov8.pt`  
- Each original image is processed independently  
- For each detection, we store:
  - class label  
  - confidence  
  - bounding box  

Annotated images are saved to: 
outputs/detections/scene*/

---

### Mapping

Each detection is mapped from image coordinates into the panorama:

- The center of the bounding box is used  
- Homographies are computed between each image and the panorama  
- Detection points are projected into panorama coordinates  

---

### Deduplication

Since objects appear in multiple overlapping images, duplicates must be removed.

- Detections are sorted by confidence  
- Only detections of the same class are compared  
- Distance is computed using: d = sqrt(dx^2 + dy^2)
- Horizontal wraparound is handled (panorama is circular)  
- Nearby detections are grouped  
- The highest-confidence detection is kept  

---

## BEV Map

### Background

The BEV map is constructed by transforming the lower portion of the panorama:

- The bottom part of the panorama is assumed to correspond to the ground  
- A perspective warp is applied to map this region into a top-down view  

This does not produce true metric depth, but it gives a reasonable approximation of spatial layout.

---

### Object Placement

For each detection:

1. Map to panorama coordinates  
2. Convert to BEV coordinates  
3. Draw a labeled circle on the map  

Only detections that fall within the BEV region are included.

---

## Challenges

This project involved several non-trivial challenges:

- Stitching without calibration: Without known camera parameters, aligning images required approximations and iterative fixes.

- Drift and misalignment: Small errors in pairwise alignment accumulated across images, causing visible distortion.

- Panorama geometry issues: Cylindrical warping introduced slanting and curvature that required additional correction steps.

- Choosing between methods: OpenCV stitching worked well when it succeeded, but failed unpredictably, so a fallback strategy was necessary.

- Duplicate detections: Objects appearing in multiple images required deduplication, especially due to panorama wraparound.

- No depth information: The BEV map relies on heuristics (e.g., object size and position), so placement is approximate rather than exact.

- Balancing realism and simplicity: The goal was to produce a visually reasonable BEV without overcomplicating the pipeline.

---

## Summary

This project shows that it is possible to approximate a BEV scene representation using a single rotating camera by combining panorama stitching, object detection, and geometric transformations. While the result is not perfectly accurate, it captures the structure of the scene and places detected objects in a consistent top-down layout.
