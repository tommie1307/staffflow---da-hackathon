# Workload Mathematics in Staffflow

This document explains the mathematical principles behind workload calculation, balance metrics, and the convergence algorithm in Staffflow.

## Overview

Staffflow uses a **patient-to-nurse ratio system** combined with **statistical analysis** to measure and optimize workload distribution across hospital nursing staff. The system tracks how many patients each nurse is assigned and uses standard deviation to measure balance.

---

## Core Concepts

### 1. Patient Count per Nurse

The most fundamental metric is simply counting how many patients each nurse has assigned:

```
Patient Count (nurse_i) = |Patients assigned to nurse_i|
```

**Example:**
- Nurse A: 7 patients
- Nurse B: 4 patients  
- Nurse C: 2 patients
- Nurse D: 0 patients

### 2. Patient Acuity Weighting

Each patient has an **acuity level** (1-5) representing care intensity:

| Acuity Level | Description | Care Requirement |
|--------------|-------------|------------------|
| 1 | Stable | Minimal monitoring |
| 2 | Moderate | Regular checks |
| 3 | Elevated | Frequent monitoring |
| 4 | High | Intensive care |
| 5 | Critical | Constant attention |

**Workload Score** is calculated as the sum of patient acuity levels:

```
Workload(nurse_i) = Σ acuity(patient_j) for all patients j assigned to nurse_i
```

**Example:**
- Nurse A has patients with acuity [3, 4, 2, 5, 3, 2, 4]
- Workload(A) = 3 + 4 + 2 + 5 + 3 + 2 + 4 = **23**

---

## Balance Metric Calculation

### Step 1: Calculate Patient Counts

For all N nurses in the system, create a list of patient counts:

```
patient_counts = [count₁, count₂, count₃, ..., countₙ]
```

**Example with 10 nurses:**
```
patient_counts = [7, 4, 4, 4, 3, 2, 2, 1, 0, 0]
```

### Step 2: Calculate Mean (Average)

The mean patient count represents the ideal balanced state:

```
μ = (Σ countᵢ) / N
```

**Example:**
```
μ = (7 + 4 + 4 + 4 + 3 + 2 + 2 + 1 + 0 + 0) / 10
μ = 27 / 10 = 2.7 patients per nurse
```

### Step 3: Calculate Variance

Variance measures how spread out the patient counts are from the mean:

```
σ² = Σ(countᵢ - μ)² / N
```

**Example calculation:**
```
Deviations from mean (2.7):
(7 - 2.7)² = 18.49
(4 - 2.7)² = 1.69
(4 - 2.7)² = 1.69
(4 - 2.7)² = 1.69
(3 - 2.7)² = 0.09
(2 - 2.7)² = 0.49
(2 - 2.7)² = 0.49
(1 - 2.7)² = 2.89
(0 - 2.7)² = 7.29
(0 - 2.7)² = 7.29

Sum = 42.10
Variance σ² = 42.10 / 10 = 4.21
```

### Step 4: Calculate Standard Deviation

Standard deviation is the square root of variance and represents the typical deviation from the mean:

```
σ = √(σ²)
```

**Example:**
```
σ = √4.21 = 2.05 patients
```

**Interpretation:** On average, each nurse's patient count differs from the mean by about 2 patients.

### Step 5: Find Maximum Patient Count

The maximum represents the worst-case workload:

```
max_count = max(patient_counts)
```

**Example:**
```
max_count = 7 patients (Nurse A is most overloaded)
```

---

## Balance Status Thresholds

The system classifies balance status using **two criteria**: maximum patient count and standard deviation.

### IDEAL Status

**Criteria:**
- `max_count ≤ 2` AND `σ ≤ 1.0`

**Meaning:** All nurses have 1-2 patients with minimal variation. This represents optimal staffing where everyone has a manageable, similar workload.

**Example:**
```
patient_counts = [2, 2, 2, 1, 1, 1, 1, 1, 1, 1]
μ = 1.3
σ = 0.46
max = 2
Status: IDEAL ✓
```

### SUFFICIENT Status

**Criteria:**
- `max_count ≤ 4` AND `σ ≤ 1.5`

**Meaning:** Most nurses have 3-4 patients with reasonable variation. Workload is manageable but not optimal.

**Example:**
```
patient_counts = [4, 4, 3, 3, 3, 2, 2, 2, 1, 1]
μ = 2.5
σ = 1.05
max = 4
Status: SUFFICIENT ✓
```

### INADEQUATE Status

**Criteria:**
- `max_count > 4` OR `σ > 1.5`

**Meaning:** At least one nurse has 5+ patients, or there's severe imbalance. Risk of burnout and patient safety issues.

**Example:**
```
patient_counts = [7, 4, 4, 4, 3, 2, 2, 1, 0, 0]
μ = 2.7
σ = 2.05
max = 7
Status: INADEQUATE ✗
```

---

## Convergence Algorithm

The rebalancing algorithm progressively moves patient assignments from overloaded to underutilized nurses to minimize standard deviation.

### Algorithm Steps

**1. Identify Imbalance**

Calculate current standard deviation and find nurses above/below mean:

```
overloaded = {nurse_i | count_i > μ + threshold}
underutilized = {nurse_j | count_j < μ - threshold}
```

**2. Calculate Transfer Benefit**

For each potential transfer from nurse A to nurse B, calculate the impact on standard deviation:

```
benefit = σ_current - σ_after_transfer
```

The transfer that produces the largest reduction in σ is selected.

**3. Apply Transfer**

Move one patient from the most overloaded nurse to the most underutilized nurse (respecting skill qualifications).

**4. Repeat**

Continue until σ drops below threshold or no beneficial transfers remain.

### Convergence Example

**Initial State (Tick 0):**
```
Assignments: [7, 4, 4, 4, 3, 2, 2, 1, 0, 0]
μ = 2.7, σ = 2.05, max = 7
Status: INADEQUATE
```

**After Tick 1:**
Transfer 1 patient from Nurse A (7→6) to Nurse I (0→1)
```
Assignments: [6, 4, 4, 4, 3, 2, 2, 1, 1, 0]
μ = 2.7, σ = 1.79, max = 6
Status: INADEQUATE (improved)
```

**After Tick 5:**
```
Assignments: [4, 4, 3, 3, 3, 2, 2, 2, 2, 2]
μ = 2.7, σ = 0.78, max = 4
Status: SUFFICIENT
```

**After Tick 10:**
```
Assignments: [3, 3, 3, 3, 2, 2, 2, 2, 2, 2]
μ = 2.4, σ = 0.49, max = 3
Status: SUFFICIENT
```

**After Tick 15:**
```
Assignments: [2, 2, 2, 2, 2, 2, 2, 2, 2, 1]
μ = 1.9, σ = 0.30, max = 2
Status: IDEAL ✓
```

---

## Mathematical Properties

### Why Standard Deviation?

Standard deviation is used because it:

1. **Penalizes outliers**: A nurse with 7 patients contributes more to σ than one with 3
2. **Scale-invariant**: Works regardless of total patient count
3. **Differentiable**: Enables optimization algorithms
4. **Interpretable**: Measured in same units as patient counts

### Convergence Guarantee

The algorithm is guaranteed to converge because:

1. Each transfer reduces σ (by design)
2. σ has a lower bound of 0 (perfect balance)
3. Number of possible states is finite

**Convergence Rate:**
- Typically reaches SUFFICIENT within 5-10 ticks
- Reaches IDEAL within 10-20 ticks
- Rate depends on initial imbalance severity

### Computational Complexity

- **Balance calculation**: O(N) where N = number of nurses
- **Transfer selection**: O(N²) to evaluate all pairs
- **Per tick**: O(N²) overall
- **Total convergence**: O(T × N²) where T = ticks to converge

For typical hospital units (N ≈ 10-20 nurses), this is very efficient.

---

## Practical Example: Full Calculation

Let's walk through a complete calculation for a 4-nurse scenario:

**Initial Assignment:**
- Nurse A: 5 patients (acuity: [3, 4, 2, 3, 5])
- Nurse B: 3 patients (acuity: [2, 3, 4])
- Nurse C: 1 patient (acuity: [2])
- Nurse D: 0 patients

### Step-by-Step Calculation

**1. Patient Counts:**
```
counts = [5, 3, 1, 0]
```

**2. Mean:**
```
μ = (5 + 3 + 1 + 0) / 4 = 9 / 4 = 2.25
```

**3. Deviations:**
```
(5 - 2.25)² = 7.5625
(3 - 2.25)² = 0.5625
(1 - 2.25)² = 1.5625
(0 - 2.25)² = 5.0625
```

**4. Variance:**
```
σ² = (7.5625 + 0.5625 + 1.5625 + 5.0625) / 4 = 14.75 / 4 = 3.6875
```

**5. Standard Deviation:**
```
σ = √3.6875 = 1.92
```

**6. Maximum:**
```
max = 5
```

**7. Status:**
```
max = 5 > 4 → INADEQUATE
```

### After One Transfer

Transfer 1 patient from A to D:

**New Counts:**
```
counts = [4, 3, 1, 1]
μ = 2.25 (unchanged, total patients same)
```

**New Deviations:**
```
(4 - 2.25)² = 3.0625
(3 - 2.25)² = 0.5625
(1 - 2.25)² = 1.5625
(1 - 2.25)² = 1.5625
```

**New Variance:**
```
σ² = 6.75 / 4 = 1.6875
```

**New Standard Deviation:**
```
σ = √1.6875 = 1.30
```

**Improvement:**
```
Δσ = 1.92 - 1.30 = 0.62 (32% reduction!)
```

---

## Visualization Mathematics

The workload chart plots patient counts over time for each nurse, creating convergence lines.

### Chart Data Structure

For each tick t and nurse i:
```
point(t, i) = (tick: t, patients: count_i(t))
```

### Convergence Visualization

As ticks progress, the lines should:
1. **Cluster together**: All lines approach mean value
2. **Reduce spread**: Vertical distance between lines decreases
3. **Stabilize**: Lines become horizontal (no more changes)

**Mathematical representation:**
```
lim(t→∞) σ(t) → 0
lim(t→∞) count_i(t) → μ for all i
```

---

## Summary

The Staffflow workload system uses straightforward statistical principles:

1. **Count patients** per nurse (simple counting)
2. **Calculate mean** (average workload)
3. **Calculate standard deviation** (measure of imbalance)
4. **Find maximum** (worst-case workload)
5. **Apply thresholds** (IDEAL/SUFFICIENT/INADEQUATE)
6. **Iteratively transfer** (reduce σ each tick)
7. **Converge to balance** (minimize σ)

The mathematics is intentionally simple and interpretable, making it easy for hospital administrators to understand and trust the system's recommendations.

---

**Key Insight:** The entire system is based on minimizing one number: **standard deviation (σ)**. When σ is low, workload is balanced. When σ is high, rebalancing is needed. The algorithm simply moves patients to reduce σ until it reaches an acceptable level.
