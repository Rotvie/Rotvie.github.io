---
layout: page
title: Autonomous Agricultural Robot LABINM
description: Self-localizing robotic system with mapping and AI capabilities for crop monitoring (FONDECYT 171-2020)
img: assets/img/projects/labinm_robot_field.jpg
importance: 1
category: work
related_publications: false
---

## Project Overview

This government-funded research project (FONDECYT 171-2020) developed an autonomous mobile robot capable of self-localization, environment mapping, and AI-powered data processing to improve agricultural yield projections for crops in the La Libertad region of Peru.

The project addressed a critical challenge in blueberry pre-harvest operations: the sampling process for yield projections requires specialized labor and is prone to errors, leading to supply chain disruptions and economic losses when projections differ from reality.

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/labinm_robot_main.jpg" title="LABINM Mobile Robot" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/blueberry_farm_peru.jpg" title="Blueberry Farm in Trujillo, Peru" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/gazebo_simulation.jpg" title="Gazebo Simulation Environment" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Left: The LABINM mobile robot designed for agricultural environments.
    Center: Blueberry cultivation in Trujillo, Peru with sandy, uneven terrain.
    Right: Simulated blueberry farm environment in Gazebo.
</div>

---

## The Robot Platform

The LABINM robot was designed specifically for navigation in challenging agricultural terrain found in coastal agro-industrial fields of Peru and Chile. Key specifications include:

| Component | Specification |
|-----------|---------------|
| **Drive System** | Skid Steering (4-wheel differential) |
| **Wheel Diameter** | 33 cm |
| **Dimensions** | 1.40m × 1.10m × 0.62m (L × W × H) |
| **LiDAR Sensor** | Ouster OS1-32 (3D, 32-channel) |
| **Compute Units** | NVIDIA Jetson + Raspberry Pi |
| **Suspension** | Active suspension system |
| **Motor Controllers** | ODrive |

The following figure shows the robot model in the Gazebo simulator showing the dimensions and the Ouster OS1-32 LiDAR mounted on top.

<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/projects/robot_gazebo_dimensions.jpg" title="Robot Model in Gazebo" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Virtual robot model in Gazebo simulator.
</div>

---

## Navigation System Architecture

The autonomous navigation system was built on the **ROS Navigation Stack**, while also integrating and testing multiple algorithms for robust operation in multiple environments:

### Core Components

- **Mapping**: Gmapping (2D SLAM) and LeGO-LOAM (3D LiDAR Odometry)
- **Localization**: Adaptive Monte Carlo Localization (AMCL), LeGO-LOAM, and Advanced Localization System (ALS)
- **Global Planning**: A* algorithm
- **Local Planning**: Dynamic Window Approach (DWA)

---

## Navigation Demo

This is a demo of the navigation system in action.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        <div class="embed-responsive embed-responsive-16by9 rounded z-depth-1">
            <iframe class="embed-responsive-item" src="https://www.youtube.com/embed/-RRT2i8cjik" allowfullscreen></iframe>
        </div>
    </div>
    <div class="col-sm mt-3 mt-md-0">
        <div class="embed-responsive embed-responsive-16by9 rounded z-depth-1">
            <iframe class="embed-responsive-item" src="https://www.youtube.com/embed/RvNR2bxZa5o" allowfullscreen></iframe>
        </div>
    </div>
</div>
<div class="caption">
    Left: Overview of the LABINM robot's autonomous navigation capabilities in agricultural environments.
    Right: Complementary video for the Workshop on Robotics in Agriculture (WRIA) at IEEE/RSJ IROS 2023.
</div>

---

## Publications

This project resulted in the following peer-reviewed publications:

1. **Comparative Analysis of LiDAR Inertial Odometry Algorithms in Blueberry Crops** (2025)
   - R. Huaman, C. Gonzalez, and S. Prado
   - *III International Congress on Technology and Innovation in Engineering and Computing*
   - [DOI: 10.3390/engproc2025083009](https://doi.org/10.3390/engproc2025083009)

2. **Performance Evaluation of the ROS Navigation Stack Using LeGO-LOAM** (2024)
   - R. Huaman, C. Gonzalez, and S. Prado
   - *Proceedings of the 9th Brazilian Technology Symposium (BTSym'23)*, Springer
   - [DOI: 10.1007/978-3-031-66961-3_16](https://doi.org/10.1007/978-3-031-66961-3_16)

3. **Linear Quadratic Regulator (LQR) Control for the Active Suspension System of a Four-Wheeled Agricultural Robot** (2023)
   - J. A. Bazan Quispe, R. J. Huaman Kemper, and S. R. Prado Gardini
   - *IEEE XXX International Conference on Electronics, Electrical Engineering and Computing (INTERCON)*
   - [DOI: 10.1109/INTERCON59652.2023.10326049](https://doi.org/10.1109/INTERCON59652.2023.10326049)

4. **Autonomous Navigation of a Four-Wheeled Robot in a Simulated Blueberry Farm Environment** (2022)
   - R. J. Huaman Kemper, C. Gonzalez, and S. R. Prado Gardini
   - *IEEE ANDESCON 2022*, Barranquilla, Colombia
   - [DOI: 10.1109/ANDESCON56260.2022.9989865](https://doi.org/10.1109/ANDESCON56260.2022.9989865)

---

## Workshop Presentations

- **Comparative Analysis of LiDAR Odometry Algorithms in Blueberry Crops** (2023)
  - C. A. Gonzalez, R. J. Huaman Kemper, and S. R. Prado Gardini
  - *Workshop on Robotics in Agriculture: Present and Future of Agricultural Robotics and Technologies*
  - IEEE/RSJ IROS 2023, Detroit, USA
  - [Workshop Website](https://sites.google.com/view/agrobotics)

---

## Acknowledgments

This research was funded by **FONDECYT** (Fondo Nacional de Desarrollo Científico, Tecnológico y de Innovación Tecnológica) under project **171-2020-FONDECYT** and conducted at the **Laboratorio de Investigación Multidisciplinaria (LABINM)** at Universidad Privada Antenor Orrego (UPAO), Trujillo, Peru.
