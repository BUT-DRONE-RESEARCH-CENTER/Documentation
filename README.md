# 🚁 Drone Research Center | Documentation Hub

<p align="center">
  <img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="License: MIT">
  <img src="https://img.shields.io/badge/Docs-MkDocs--Material-blue?logo=materialformkdocs" alt="MkDocs">
  <img src="https://img.shields.io/badge/Status-Internal--Active-success" alt="Build Status">
  <img src="https://img.shields.io/badge/Field-Autonomous--Robotics-orange" alt="Research">
</p>

> The **Single Source of Truth** for autonomous aerial research, hardware specifications, and flight protocols at the DRC.

This repository hosts the technical documentation for all (in)active projects within the lab. It is designed to be a collaborative knowledge base for researchers, engineers, and students.

[**Explore the Docs »**]([https://your-github-pages-link.com](https://github.com/BUT-DRONE-RESEARCH-CENTER/Documentation))

## 🏠 [Home](./docs/index.md)
The central landing page. Contains core documentation info, lab mission statements, and general announcements.

## 🔬 [Projects](./docs/projects/)
Current and historical research initiatives. All projects are tracked with **weekly versioning** to maintain a clear timeline of progress.
* **[Software](./docs/projects/software/):** Projects focused on algorithms, AI, and flight stacks (LUA scripts, Python projects, ROS2).
* **[Hardware](./docs/projects/hardware/):** Physical engineering, PCB design, and airframe construction (FC, GPS configurations, Flash procedures).
* **[Hybrid](./docs/projects/hybrid/):** Full-system integration combining custom hardware and software (Full drone builds).

## 🎓 [Workshops](./docs/workshops/)
A directory of active lab training sessions. Each entry defines the **target audience** (High-school, Graduate etc.), **learning objectives** (ROS2, Camera Detection etc.), and **curriculum** (What will the audience learn.).

## 📖 [Tutorials](./docs/tutorials/)
The final destination for completed research. Once a project reaches a milestone, it is distilled into a tutorial. This section also hosts a curated list of essential external web resources.

## ✍️ Documentation Protocol

To maintain accountability and organized knowledge transfer, all members must follow this workflow:

### 1. Individual Progress Logs
Every researcher has a dedicated file within the specific project folder to track their private progress.
* **Location:** `Projects/Category/ProjectName/YourName.md`
* **Example:** `Projects/Software/ICUAS/Adam.md`
* **Frequency:** Update this file **weekly** with tasks, bugs, and data logs.

### 2. The Project Master (`main.md`)
The `main.md` file exists in every project folder and serves as the official/final project record.
* **Workflow:** Upon conclusion or a major milestone, individual logs are summarized and merged into `main.md`.
* **Content:** Final architectures, installations, consolidated results, and conclusions.
