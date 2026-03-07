---
title: "Precise Timeline Scheduler for FreeRTOS"
description: "A deterministic, timeline-driven scheduler that replaces the standard FreeRTOS priority-based preemption model, designed for time-critical applications enforcing a Time-Triggered Architecture (TTA). (Source code coming soon.)"
tags: ["freertos", "scheduler", "real-time", "embedded", "c"]
# repo: "https://github.com/"
# url: "https://example.com"
# relatedPosts: ["hello-world"]
featured: true
draft: false
---

## Overview

This project implements a deterministic, timeline-driven scheduler that replaces the standard FreeRTOS priority-based preemption model. Designed for time-critical applications, it enforces a Time-Triggered Architecture (TTA) where tasks execute within strict time windows (Sub-frames) inside a repeating cycle (Major Frame).

## Features

- Deterministic timing where execution is governed by a static schedule table defined at compile time.

- Frame-based architecture utilizing a Major Frame for the global cycle duration and Sub-Frames for smaller fixed slots.

- Hybrid task model supporting Hard Real-Time (HRT) tasks with strict deadlines that are terminated if missed, and Soft Real-Time (SRT) tasks that run sequentially in the idle time of a sub-frame.

- Safety and recovery mechanisms including a Supervisor task to handle deadline misses by resetting failed tasks.

- Trace and debugging support via a lightweight, buffered tracing module designed to record scheduler behavior with tick-level precision.

- Automated testing framework driven by a dedicated Makefile for isolated test execution and validation.
