# Welcome to Drone Research Center | Wiki

Toto je oficiální stránka pro dokumentaci DRC projektů

Cooked up from this guy's yt video: [-> link here <-](https://www.youtube.com/watch?v=xlABhbnNrfI)

<p align="center">
  <img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="License: MIT">
  <img src="https://img.shields.io/badge/Docs-MkDocs--Material-blue?logo=materialformkdocs" alt="MkDocs">
  <img src="https://img.shields.io/badge/Status-Internal--Active-success" alt="Build Status">
  <img src="https://img.shields.io/badge/Field-Autonomous--Robotics-orange" alt="Research">
</p>

> The **Single Source of Truth** for autonomous aerial research, hardware specifications, and flight protocols at the DRC.

This repository serves as our collective flight recorder—capturing technical breakthroughs, safety protocols, and the evolution of our aerial platforms. It is designed to be a collaborative knowledge base for researchers, engineers, and students.

[**Explore the Docs »**](https://but-drone-research-center.github.io/Documentation/)

## 🏗️ Lab Organization
The DRC is organized into three primary technical pillars. When contributing, ensure your work is categorized correctly:

| Pillar | Focus Area | Key Tech Stack |
| :--- | :--- | :--- |
| **[Software](./projects/software/)** | Autonomy & Intelligence | ROS2, Python, C++, LUA scripts, AI/CV. |
| **[Hardware](./projects/hardware/)** | Physical Engineering | PCB Design, Airframes, ESC/Motor tuning. |
| **[Hybrid](./projects/hybrid/)** | System Integration | Full builds, ArduPilot/PX4, MAVLink. |


## 📚 Navigation Guide

### 🔬 [Projects](./projects/)
Current and historical research initiatives. All projects are tracked with **weekly versioning** to maintain a clear timeline of progress from concept to flight.

### 🎓 [Workshops](./workshops/)
A directory of active lab training sessions. Each entry defines the **target audience** (High-school to Graduate), **learning objectives**, and the **curriculum**.

### 📖 [Tutorials](./tutorials/)
The final destination for completed research. Once a project reaches a stable milestone, it is distilled into a guide. This section also hosts curated external resources.

## ✍️ Documentation Protocol
To maintain accountability and organized knowledge transfer, all members must follow this workflow:

### 1. Individual Progress Logs
Every member maintains a dedicated "Dev Diary" within their specific project folder.
- **Location:** `Projects/Category/ProjectName/YourName.md` (e.g., `Projects/Software/ICUAS/Adam.md`)
- **Frequency:** Update **weekly** with tasks, bugs, and raw data logs.
- **Context:** Document the "failures" too—knowing why a $PID$ tune failed is as important as the final values.

### 2. The Project Master (`main.md`)
The official/final project record located in every project folder.
- **Workflow:** Upon reaching a major milestone, individual logs are summarized and merged into `main.md`.
- **Content:** Final architectures, installation guides, and consolidated conclusions.

## 🛠️ Content Standards
To keep this wiki clean, follow the **Signal-over-Noise** principle:

**✅ What to Write:**
* **The "Why":** Explain the logic (e.g., why we chose $LQR$ control over $PID$ for a specific frame).
* **Versioning:** Specify exact firmware, OS (e.g., Ubuntu 22.04), and hardware revisions.
* **SOPs:** **Standard Operation Procedures**--Step-by-step guides that a new member could follow without supervision.

**❌ What NOT to Write:**
- **Redundant Manuals:** Don't copy-paste the Betaflight or ArduPilot wiki. **Link to it**!
- **Ephemeral Data:** Don't document daily weather or current battery voltages, unless it is specifically required. Keep it evergreen.
- **Unverified Hacks:** If you bypassed a safety sensor, label it as a **temporary workaround**, not a standard procedure.

> **Note:** All flight logs must be uploaded within 24 hours of a field test. Boring flights are still data—document them. As well as crashes.
