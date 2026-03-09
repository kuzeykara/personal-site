---
title: "SSH Shell Attack Session Classifier"
description: "Multi-label classifier for SSH sessions using the MITRE ATT&CK framework. Explores data cleaning, TF-IDF vectorization, Linear SVM, clustering (DBSCAN/K-Means), and BERT fine-tuning on honeypot-captured attack data."
tags: ["machine-learning", "cybersecurity", "nlp", "classification", "mitre-attack", "python"]
# repo: "https://github.com/..."
featured: true
draft: false
---

## Overview

This project classifies SSH shell sessions (sequences of commands executed on a honeypot) according to intents from the **MITRE ATT&CK** framework. Each session can exhibit multiple behaviors, so the task is framed as **multi-label classification** with intents such as Discovery, Defense Evasion, Execution, Persistence, Impact, and Harmless.

The work started as an academic project for the *Machine Learning for Networking* course (Cybersecurity Engineering MSc @ Politecnico di Torino), in collaboration with [Tiziano Radicchi](https://github.com/Tiz314), [Melissa Moioli](https://github.com/Rebel-Nightmare), and [Emir Akinci](https://github.com/emirakinci). It is not intended as a state-of-the-art benchmark but as a study of data cleaning, model selection, and the impact of semantic deduplication on real-world, noisy attack data.

## The challenge

Attack sessions are highly repetitive: automated scripts produce massive duplication with only small variations (e.g. obfuscated payloads, pseudo-random strings). A single session can be labeled with multiple intents (e.g. Discovery and Persistence), and labels are treated as independent Bernoulli trials because ground truth is derived from substring-based rules rather than a holistic session verdict.

## Data and pre-processing

- **Raw data:** ~233,000 sessions from an SSH-accessible honeypot, with timestamps and full command sequences.
- **Cleaning pipeline:** De-obfuscation (removal of pseudo-random strings), replacement of instance-specific data (IPs → `<IPV4>`/`<IPV6>`, passwords → `<PASSWORD>`), and replacement of HEX/Base64 blobs with placeholders like `<B64>`. After deduplication, the dataset reduces to **1952 unique semantic samples**, showing that the vast majority of raw records were semantic duplicates.
- **Vectorization:** TF-IDF to downweight generic shell commands (e.g. `grep`, `echo`) and emphasize discriminative tokens. Feature selection via Chi-Square (e.g. top 400 features) to limit overfitting.

## Classification

- **Naive Bayes:** High bias and underfitting; weak on minority and overlapping intents (e.g. Harmless).
- **Linear SVM:** Best-performing classical approach. After hyperparameter tuning (e.g. regularization `C`, `max_iter`), **~83% exact-match accuracy** and solid macro F1 on the cleaned dataset. Stratified multi-label train/validation/test splits and Multilabel Stratified K-Fold for tuning.
- **MLP:** Funnel-shaped architecture (400 → 256 → 128 → 7); ~79% accuracy without extensive tuning.
- **BERT (bert-base-uncased):** Fine-tuning only a final dense layer. Underperforms Linear SVM, likely due to small cleaned dataset size and BERT’s 512-token limit truncating long sessions. Suggests value in code/command-line-specific pre-trained models for future work.

Results on the **original** (uncleaned) dataset were misleadingly high due to data leakage from near-duplicate sessions; the cleaned dataset gives lower but more realistic and generalizable metrics.

## Unsupervised analysis

- **Clustering:** K-Means (e.g. \(k=7\)) and **DBSCAN** (e.g. \(\epsilon=0.05\), MinPts=200) on TF-IDF vectors reduced via TruncatedSVD (e.g. 50 components). DBSCAN gives better separation and isolates noise; clusters align with attack campaigns rather than single tactics.
- **Findings:** Distinct sub-techniques within clusters, e.g. “Polymorphic Credential Modification” campaigns where credentials are rotated per session, and “Stealthy Reconnaissance” (Discovery + Defense Evasion) scripts in `/dev/shm`.

## Deliverables

- **Report:** Full write-up with dataset analysis, cleaning rationale, classification and clustering methodology, and BERT experiments.
- **Notebooks:** Section 1 (data exploration and pre-processing), Section 2 (classification pipeline and cleaned-dataset results), Section 3 (clustering), Section 4 (transformer-based classification).
