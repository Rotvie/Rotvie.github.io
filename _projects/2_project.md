---
layout: page
title: Blueberry Segmentation and Ripeness Assessment
description: Deep learning-based detection and classification of blueberries for pre-harvest ripeness assessment (FONDECYT 171-2020)
img: assets/img/projects/blueberry_segmentation.jpg
importance: 2
category: work
related_publications: false
---

## Project Overview

This government-funded research project (FONDECYT 171-2020) developed an artificial vision system for automated blueberry detection and ripeness classification in agro-industrial environments. This system enables automated fruit sampling during pre-harvest operations, addressing the labor-intensive nature of manual ripeness assessment in large blueberry fields.

Peru has become a major contributor to the international blueberry market, exporting to the UK, US, and China. However, traditional ripeness assessment methods remain time-consuming, labor-intensive, and prone to human error from worker fatigue. Our system automates this process using deep learning techniques.

<div class="row justify-content-sm-center">
    <div class="col-sm-5 mt-3 mt-md-0">
        <div class="image-equal-height">
            {% include figure.liquid 
                loading="eager" 
                path="assets/img/projects/blueberry_pipeline.jpg" 
                title="Detection and Classification Pipeline" 
                class="img-fluid rounded z-depth-1 equal-img" %}
        </div>
    </div>
    <div class="col-sm-5 mt-3 mt-md-0">
        <div class="image-equal-height">
            {% include figure.liquid 
                loading="eager" 
                path="assets/img/projects/person_count_blueberry.jpg" 
                title="Manual Blueberry Sampling and Labor Demand" 
                class="img-fluid rounded z-depth-1 equal-img" %}
        </div>
    </div>
</div>

<div class="caption">
    Left: Automated detection and ripeness classification pipeline (ZED 2i → YOLOv8 → DINOv2 embeddings → MLP classifier).  
    Right: Manual blueberry sampling in the field, highlighting labor intensity and scalability limitations.
</div>


---

## System Architecture

The system consists of two main modules:

**Object Detection Module**: YOLOv8 detects and segments blueberries in images, outputting bounding boxes and instance masks as Regions of Interest (ROIs).

**Classification Module**: DINOv2 extracts embeddings from each ROI, which are then classified by an MLP head into one of five ripeness stages.

### Ripeness Stages

| Stage | Name | Description |
|-------|------|-------------|
| 1 | **Green** | Immature, fully green |
| 2 | **Cream** | Early color change |
| 3 | **Blush** | Intermediate ripeness |
| 4 | **Pink** | Near-ripe |
| 5 | **Blue** | Fully mature |

<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/ripeness_stages.jpg" title="Five Ripeness Stages" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Blueberry ripeness stages showing color progression from immature (Green) to fully mature (Blue).
</div>

---

## Data Collection

Images were captured using a **ZED 2i stereo camera** at agro-industrial farms in Trujillo, Peru. The detection dataset contains 200 images with 1,708 labeled blueberry instances, while the classification dataset comprises 900 ROIs balanced across the five ripeness classes.

<div class="row justify-content-sm-center">
    <div class="col-sm-10 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/blueberry_field_detection.jpg" title="Field Detection Results" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Detection and classification results on field images. Each detected blueberry is labeled with its ripeness stage.
</div>

---

## Results

The system achieved **93.70% classification accuracy** across five ripeness stages, with the YOLOv8 Medium backbone achieving **91.9% mAP@50** for instance segmentation and **92.1% mAP@50** for object detection. The confusion matrix shows the classification performance across all five ripeness stages.

<div class="row justify-content-sm-center">
    <div class="col-sm-6 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/confusion_matrix.jpg" title="Confusion Matrix" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Confusion matrix showing classification performance across all five ripeness stages.
</div>

---

## Publications

1. **Artificial Vision Strategy for Ripeness Assessment of Blueberries on Images Taken During Pre-harvest Stage in Agroindustrial Environments Using Deep Learning Techniques** (2023)
   - P. B. Cubas Muñoz, R. J. Huaman Kemper, and S. R. Prado Gardini
   - *IEEE XXX International Conference on Electronics, Electrical Engineering and Computing (INTERCON)*
   - [DOI: 10.1109/INTERCON59652.2023.10326058](https://doi.org/10.1109/INTERCON59652.2023.10326058)

---

## Workshop Presentations

- **Deep Learning Approach for Accurate Pre-Harvest Blueberry Ripeness Classification** (2023)
  - P. Cubas, R. J. Huaman Kemper, and S. R. Prado Gardini
  - *Workshop on Robotics in Agriculture: Present and Future of Agricultural Robotics and Technologies*
  - IEEE/RSJ IROS 2023, Detroit, USA
  - [Workshop Website](https://sites.google.com/view/agrobotics)

---

## Acknowledgments

This research was funded by **FONDECYT** (Fondo Nacional de Desarrollo Científico, Tecnológico y de Innovación Tecnológica) under project **171-2020-FONDECYT** and conducted at the **Laboratorio de Investigación Multidisciplinaria (LABINM)** at Universidad Privada Antenor Orrego (UPAO), Trujillo, Peru.
