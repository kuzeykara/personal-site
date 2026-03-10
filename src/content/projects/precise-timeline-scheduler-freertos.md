---
title: "Precise Timeline Scheduler for FreeRTOS"
description: "A deterministic, timeline-driven scheduler that replaces the standard FreeRTOS priority-based preemption with a Time-Triggered Architecture (TTA). Includes JSON-based schedule generation, kernel hooks for precise time-keeping, and UART tracing for validation."
tags: ["freertos", "scheduler", "real-time", "embedded", "c"]
repo: "https://github.com/POOYASP2/precise_timeline_scheduler_for_FreeRTOS"
featured: true
draft: false
---

## Overview

This project implements a **deterministic, timeline-driven scheduler** for FreeRTOS that replaces the standard priority-based preemption model with a **Time-Triggered Architecture (TTA)**. Designed for time-critical embedded applications, the system governs task execution through a **static schedule defined at compile time**, so operations occur within strict time windows (Sub-frames) inside a globally repeating cycle (Major-frame), giving predictable and repeatable behavior.

The architecture supports a **hybrid task model**: **Hard Real-Time (HRT)** tasks run based on absolute start and end times and are terminated if they miss their deadline; **Soft Real-Time (SRT)** tasks run sequentially during idle periods within a sub-frame. A **Supervisor task** monitors execution and resets tasks that miss deadlines. The project includes a **JSON-based configuration tool** for auto-generating schedules, **custom FreeRTOS kernel hooks** for precise time-keeping, and a **lightweight tracing system** for validating performance via UART. Full architecture, kernel modifications, and API details are documented in the repository’s PDF: *Precise_Timeline_Scheduler_for_FreeRTOS.pdf*. The work was done as part of our *Operating Systems for Embedded Systems* course, in collaboration with [Giacomo Pessolano](https://github.com/GiacomoPessolano), [Mohammad Tohidnia](https://github.com/mohammadTohidnia), [Muhammed Emir Akinci](https://github.com/emirakinci), and [Pooya Sharifi](https://github.com/POOYASP2).

## Architecture

- **Major Frame / Sub-frames:** The global cycle (Major Frame) is divided into fixed-duration Sub-frames. Each sub-frame has a configurable duration (e.g. 100 ms). Tasks are assigned to sub-frames and, for HRT tasks, to start/end times relative to the sub-frame.

- **Task types and states:** Tasks are typed as `HARD_RT` or `SOFT_RT`. States are tracked as `TASK_NOT_STARTED`, `TASK_RUNNING`, `TASK_DONE`, or `TASK_DEADLINE_MISSED`. HRT tasks must start and end on schedule; if they overrun, they are killed and the scheduler can start the next scheduled task. SRT tasks execute only when the CPU is idle within a sub-frame.

- **Schedule validation:** The scheduler validates the schedule table for invalid times (e.g. start ≥ end), out-of-bounds (end > sub-frame duration), overlapping HRT tasks, and invalid sub-frame IDs. Preprocessing converts absolute times to relative and checks frame boundaries. Errors are reported via `vApplicationScheduleErrorHook` and the system halts with interrupts disabled.

## Implementation highlights

- **Core API:** `vStartTimelineScheduler(schedule_table, table_size, subframe_duration_ms, total_subframes)` verifies the table and starts the OS. `xUpdateTimelineScheduler()` is called from the tick hook to advance the timeline. `xPreprocessSchedule()` and `xValidateSchedule()` handle validation and relative-time conversion.

- **Application hooks:** `application_hooks.c` implements `vApplicationIdleHook` to measure idle time per sub-frame (for tracing) and `vApplicationScheduleErrorHook` to log the error reason (overlap, out-of-bounds, invalid sub-frame, invalid time, or preprocessing failure) over UART and then stop the system.

- **Tracing:** A buffered trace module records scheduler behavior with tick-level precision. Trace data can be emitted via UART for offline analysis. Task names are registered from the generated schedule for readable logs.

- **Tooling:** The `tools/` directory contains `gen_schedule.py` and `schedule.json`. The Python script reads the JSON and generates the C schedule table and frame parameters (e.g. `g_major_frame_ms`, `g_minor_frame_ms`, `g_subframe_count`) so the main application uses a single generated schedule.

## Repository layout

- **FreeRTOS-Kernel:** Modified FreeRTOS kernel.
- **config:** FreeRTOS and application configuration.
- **device, drivers:** Target-specific and driver code.
- **tasks:** Application tasks that are scheduled by the timeline.
- **utils:** Shared utilities (e.g. UART, trace).
- **testing:** Test targets; the Makefile supports isolated test execution and validation.
