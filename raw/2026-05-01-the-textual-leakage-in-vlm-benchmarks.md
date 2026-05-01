---
title: "Why LLMs Don't Need 'Eyes' to Score High: The 'Textual Leakage' Trap in VLM Benchmarks"
date: "2026-05-01"
author: "大主管 (Grand Overseer)"
tags: ["AI", "Infrastructure", "Compute", "VLM"]
status: "draft"
---

# Why LLMs Don't Need "Eyes" to Score High: The "Textual Leakage" Trap in VLM Benchmarks

Recent blind evaluations (inspired by research from Stanford and independent AI testers) have revealed a startling fact: Frontier models like GPT-5, Gemini, and Claude can retain **70% to 80% of their scores** on visual benchmarks even when the images are completely removed.

## 1. The Root Cause: Textual Leakage
The "magic" isn't a sign of AI cheating; it's a symptom of poorly designed benchmarks. Most current Vision-Language Model (VLM) tests suffer from what we call **"Textual Leakage"**:

*   **Over-Descriptive Prompts**: The questions themselves contain so much context about the scene (e.g., "In this X-ray of a fractured tibia...") that the model can infer the answer using its massive pre-trained common sense and medical knowledge.
*   **Logical Shortcuts**: If a question asks, "Is the traffic light red or green?" and mentions a car stopping, the model will guess "red" based on language logic, not pixel analysis.

## 2. The "Small Model" Paradox
The claim that "3-billion parameter models are beating giants" needs nuance. In specialized, pixel-dense tasks (like pure OCR or industrial defect detection), dedicated small models are more efficient because they aren't distracted by the "language noise" that often leads larger models to hallucinate based on prompt cues.

## 3. Strategic Implications for Limina Labs & The Compute Market

### A. The Compute Waste Crisis
If 80% of a VLM's performance comes from text-based reasoning, then the massive GPU clusters currently allocated to "visual processing" for these tasks are significantly underutilized. 

### B. Moving Toward "Zero-Text" Benchmarks
To truly measure visual intelligence, we need benchmarks where:
1.  **Prompts are minimal** (e.g., "What's wrong here?").
2.  **Answers are hidden in pixels**, not logic.

### C. Opportunity for Limina Labs
As the industry matures, we expect a shift from "Generic Multi-modal Giant Models" toward a **Hybrid Infrastructure**:
*   **Powerful Text Reasoning Core** (Cheap, text-only).
*   **Specialized Vision Micro-services** (Lightweight, high-precision).

Limina Labs is positioning its compute routing to support this modular future, ensuring that every TFLOPS of compute we source for our clients is actually performing "work," not just processing textual redundancy.

---
*Follow Limina Labs for more insights into the intersection of AI architecture and physical infrastructure.*
