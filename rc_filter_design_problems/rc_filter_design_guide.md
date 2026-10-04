# 🔌 RC Low-Pass Filter Design & Practical Selection Guide for Embedded Systems

> **Subject**: EE4053 - Embedded Systems Design (PBL) / Thiết kế Hệ thống Nhúng  
> **Topic**: Practical RC Hardware Filtering for Digital & Analog Inputs (Buttons, TCRT5000, Encoders)

---

## 📌 Overview

In embedded hardware design, raw digital signals from mechanical switches, optical line sensors, or high-speed encoders often suffer from electrical noise, high-frequency interference, or mechanical bouncing. 

An **RC Low-Pass Filter (LPF)** is a simple yet vital passive circuit used to smooth out noise before the signal reaches microcontrollers (MCUs). However, selecting an incorrect capacitor value can either fail to suppress noise or severely distort the square wave signal, causing dropped pulses or logic state detection failures ($V_{IH} / V_{IL}$).

This repository provides the theoretical foundations, practical sizing formulas, lookup tables, and real-world failure analyses for choosing $R$ and $C$ values in embedded systems.

---

## 📐 Mathematical Foundations

An RC low-pass filter consists of a resistor $R$ in series with the signal path and a capacitor $C$ connected to ground.

### 1. Cutoff Frequency ($f_c$)
The frequency at which the output signal amplitude drops to approximately $70.7\%$ ($-3\text{ dB}$) of the input:

$$f_c = \frac{1}{2\pi R C}$$

* **Rule of Thumb**: Increasing $C$ reduces $f_c$, attenuating lower-frequency noise components.

### 2. Signal Rise Time ($t_{rise}$)
The time required for the signal to rise from $10\%$ to $90\%$ of its final step response value:

$$t_{rise} \approx 2.2 R C$$

* **Rule of Thumb**: Increasing $C$ increases $t_{rise}$, making the pulse edges less steep (shaping square waves into curved/sloped edges).

### 3. Edge Integrity Constraint
To avoid deforming a digital pulse into a triangular wave, the rise time $t_{rise}$ should not exceed **$1\%$ of the signal period ($T$)**:

$$t_{rise} \le \frac{T}{100}$$

---

## 📊 Application Lookup Table ($R = 1\text{ k}\Omega$)

| Input Application | Operating Frequency ($f$) | Period ($T$) | Capacitance Range ($C$) | Filtering Strength & Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Push Button / Reset Switch** | $< 10\text{ Hz}$ | $> 100\text{ ms}$ | $100\text{ nF} - 470\text{ nF}$ | **Heavy Filtering**: Debouncing mechanical contact chatter. |
| **Line Sensor (TCRT5000)** | $100\text{ Hz} - 1\text{ kHz}$ | $1\text{ ms} - 10\text{ ms}$ | $4.7\text{ nF} - 47\text{ nF}$ (Typ. $10\text{ nF}$) | **Moderate Filtering**: Preserves pulse shape while removing IR noise. |
| **High-Speed Encoder** | $1\text{ kHz} - 10\text{ kHz}$ | $100\text{ }\mu\text{s} - 1\text{ ms}$ | $470\text{ pF} - 4.7\text{ nF}$ | **Light Filtering**: Fast response without pulse distortion. |

---

## 🧮 Practical Design Examples

### Example 1: High-Speed Wheel Encoder ($1\text{ kHz} - 10\text{ kHz}$)

#### 1. Input Specification:
* Frequency Range: $f \in [1\text{ kHz}, 10\text{ kHz}]$
* Period Range: $T \in [10^{-4}\text{ s}, 10^{-3}\text{ s}]$
* Fixed Resistor: $R = 1\text{ k}\Omega$

#### 2. Calculation:
Target rise time constraint:
$$10^{-6}\text{ s} \le t_{rise} \le 10^{-5}\text{ s}$$

Using $t_{rise} = 2.2 R C$:
$$\frac{10^{-6}}{2.2 \times 1000} \le C \le \frac{10^{-5}}{2.2 \times 1000}$$

$$0.454\text{ nF} \le C \le 4.54\text{ nF}$$

#### 3. Standard Practical Value Choice:
$$470\text{ pF} \le C \le 4.7\text{ nF}$$

---

### Example 2: Line Follower Sensor TCRT5000 ($100\text{ Hz} - 1\text{ kHz}$)

#### 1. Input Specification:
* Scanning Frequency: $f \in [100\text{ Hz}, 1\text{ kHz}]$
* Period Range: $T \in [1\text{ ms}, 10\text{ ms}]$
* Fixed Resistor: $R = 1\text{ k}\Omega$

#### 2. Calculation:
Target rise time constraint:
$$10^{-5}\text{ s} \le t_{rise} \le 10^{-4}\text{ s}$$

Using $t_{rise} = 2.2 R C$:
$$4.545\text{ nF} \le C \le 45.45\text{ nF}$$

#### 3. Standard Practical Value Choice:
Selection of $C = 10\text{ nF}$ satisfies the range and provides an optimal trade-off between noise suppression and rise-time speed.

---

## ⚠️ Case Study: Failure Analysis of Over-Filtering

### Scenario:
A engineer mistakenly uses a button debouncing capacitor ($C = 470\text{ nF}$) on a $1\text{ kHz}$ TCRT5000 line sensor signal path with $R = 1\text{ k}\Omega$.

### Analysis:
1. **Calculate $t_{rise}$**:
   $$t_{rise} = 2.2 \times 1000\text{ }\Omega \times 470 \times 10^{-9}\text{ F} \approx 1.034\text{ ms}$$

2. **Comparison with Signal Period ($T$)**:
   For $f = 1\text{ kHz}$, the total signal period is $T = 1\text{ ms}$. Here, $t_{rise} = 1.034\text{ ms} > T$.

3. **Technical Consequences**:
   * **Severe Pulse Distortion**: The capacitor charges too slowly to reach full voltage within a pulse period.
   * **Voltage Clipping**: The square wave degenerates into a low-amplitude triangular wave.
   * **Logic Detection Failure**: The voltage never reaches the microcontroller's logic HIGH input threshold ($V_{IH}$). Consequently, the MCU misses line transitions completely.

---

## 💡 Summary & Best Practices

1. **Always calculate $t_{rise}$ relative to period $T$** before placing passive components on high-frequency digital lines.
2. **Use higher $C$ values ($100\text{ nF} - 470\text{ nF}$)** strictly for slow, human-operated inputs (switches/buttons).
3. **Use lower $C$ values ($\text{pF}$ to low $\text{nF}$ range)** for high-speed sensor streams (optical encoders, PWM signals).
4. If passive filtering alters the edge slope too much for MCU GPIO interrupts, consider adding a **Schmitt Trigger (e.g., 74HC14)** after the RC network to restore clean digital edges.

---

## 📜 References
* Course: EE4053 - Embedded System Design (PBL)
* Components Analyzed: Push buttons, TCRT5000 IR reflective sensor, Quadrature Encoder.