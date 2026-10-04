# stm32-rc-car-controller
Hardware design and mainboard controller for an STM32-based RC Car. Features custom PCB, RC noise filtering, and multi-frequency power decoupling network. (EE4053 Embedded System Design (PBL))

# 🚗 STM32 RC Car Controller PCB Design & Power Filtering Architecture

> **Course Project**: EE4053 - Embedded Systems Design (PBL) / Thiết kế Hệ thống Nhúng  
> **Target Hardware**: STM32F103C8T6 (BluePill Microcontroller)  
> **Application**: Autonomous & Remote-Controlled (RC) Vehicle Main Controller Board  

---

## 📌 Project Overview

This repository contains the schematic, PCB layout, and engineering documentation for an embedded **RC Car Controller Board**. Designed around the STM32F103C8T6 microcontroller, the board integrates multi-channel sensor inputs, I2C expansion, dual motor/PWM output headers, and a robust power management circuit designed to sustain battery fluctuations and high-frequency motor noise.

```
       +-------------------------------------------------------+
       |                  12V/7.4V Battery Input               |
       +---------------------------+---------------------------+
                                   |
                         [Fuse F1 + Diode D1]
                                   |
                          [LM7805 LDO Reg]
                                   |
                         +---------+---------+
                         | 5V & 3.3V Power   |
                         | Filtering Network | (C9, C10, C11)
                         +---------+---------+
                                   |
       +---------------------------+---------------------------+
       |              STM32F103C8T6 Core MCU                   |
       +-------+-------------------+-------------------+-------+
               |                   |                   |
      [Input Sensors]       [I2C Module]        [Motor Driver]
     (Line/Encoders J1,J3,J6)  (OLED/IMU J2)      (PWM/DIR J4,J5)
```

---

## 🛠️ Hardware System Architecture

### 1. Power Regulation & Protection Stage
* **Power Input ($J7$)**: Accepts raw power ($7.4\text{V} - 12\text{V}$ LiPo/Li-ion battery).
* **Reverse Polarity Protection**: Schottky Diode $D1$ ($\text{SS36}$) protects components against inverted battery insertion with low forward voltage drop.
* **Overcurrent Protection**: Resettable PTC Fuse $F1$ ($16\text{V}, 2.5\text{A}$) prevents trace burnout during motor stalls or short circuits.
* **Voltage Regulation ($U2$)**: $\text{LM7805ACT}$ linear regulator steps down input voltage to $5\text{V}$ logic power.

### 2. Microcontroller Core ($U1A / U1B$)
* **Processor**: STM32F103C8T6 (ARM Cortex-M3).
* **Power Pins**: $3.3\text{V}$ provided directly via BluePill onboard LDO or external rail, heavily bypassed with decoupling capacitors.

### 3. Peripheral Interfaces
* **Sensor Input Connectors ($J1, J3, J6$)**: 3-pin headers for IR Line Sensors (e.g., TCRT5000), Speed Encoders, or Ultrasonic sensors. Equipped with dedicated filtering capacitor pairs ($100\text{ nF}$ ceramic + $1\ \mu\text{F}$ electrolytic/tantalum).
* **I2C Expansion ($J2$)**: Dedicated header for OLED display or MPU6050 IMU ($\text{SCL1}, \text{SDA1}$) with local power decoupling ($C_3=1\ \mu\text{F}, C_4=100\text{ nF}$).
* **Motor Control Output ($J4, J5$)**: Dedicated headers routing PWM speed controls ($\text{PWM1} - \text{PWM4}$) and direction enable signals ($\text{EN1A/B}, \text{EN2A/B}$) to H-Bridge motor drivers (e.g., L298N, TB6612FNG).
* **UART Debug Port ($P1$)**: TX/RX header for Bluetooth (HC-05/06) or telemetry logging.

---

## ⚡ Multi-Capacitor Power Supply Filtering Strategy

In an RC Car environment, DC motors generate severe high-frequency voltage spikes, electrical noise, and momentary voltage sags. A single filter capacitor is insufficient across the entire noise frequency spectrum.

```
       Power Rail (5V / 3.3V)
         o---------------+-----------------+-----------------+--- o To MCU / IC
                         |                 |                 |
                       +---+             +---+             +---+
                       |   | C9          |   | C10         |   | C11
                       |   | 1 uF        |   | 2.2 uF      |   | 100 nF
                       +---+             +---+             +---+
                         |                 |                 |
         GND o-----------+-----------------+-----------------+--- o GND
```

### 1. Capacitor Selection Breakdown

| Component | Value | Type | Distance to IC | Filtering Function |
| :--- | :--- | :--- | :--- | :--- |
| **$C_{11}$** | $100\text{ nF}$ ($0.1\ \mu\text{F}$) | Ceramic (MLCC) | **Directly adjacent** to IC power pin | **High-Frequency Noise (MHz)**: Low Equivalent Series Inductance (ESL) suppresses fast transient voltage spikes. |
| **$C_{10}$** | $2.2\ \mu\text{F}$ | Ceramic / Tantalum | Near regulator output | **Mid-Frequency Noise (kHz)**: Smooths switching ripple from power converters or PWM switching. |
| **$C_9$** | $1.0\ \mu\text{F}$ | Electrolytic / Ceramic | Near main power input | **Low-Frequency Bulk Filtering**: Acts as an energy reservoir during sudden motor torque bursts to prevent MCU voltage brownouts. |

### 2. Frequency Response Analysis

By paralleling capacitors of different decade orders ($100\text{ nF}, 1\ \mu\text{F}, 2.2\ \mu\text{F}$), the overall power distribution network (PDN) impedance remains low across a broad frequency spectrum:

$$\text{Total PDN Impedance } Z_{total}(f) = Z_{C11}(f) \parallel Z_{C10}(f) \parallel Z_{C9}(f)$$

---

## ⏱️ Power-On Reset (POR) & Transient Rise Time ($t_{rise}$)

### 1. The Power-On Reset (POR) Dilemma
Microcontrollers (STM32, ESP32, AVR) rely on an internal **Power-On Reset (POR)** circuit. If the supply voltage $V_{DD}$ rises too slowly during power-up ($t_{rise} > 10\text{ ms}$), the internal logic can become trapped in an indeterminate voltage state ($1.5\text{V} - 2.0\text{V}$), causing the MCU to **freeze or crash on boot**.

### 2. Rise Time Formula & Trade-off

The power rail voltage rise time is approximated by:

$$t_{rise} \approx 2.2 R C_{equivalent}$$

Where $R$ is the effective source internal resistance and $C_{equivalent}$ is the total parallel capacitance on the rail.

```
 Voltage (V)
   ^
5V |-------------/------------ (Clean, stable DC)
   |            /  \  /  \ 
   |           /    \/    \  <-- Excessive Ringing / Oscillations (Unfiltered)
   |          /
   |  /------/  <-- Ideal $t_{rise}$ (Damped noise, fast POR transition)
   | /
 0 +----------------------------------> Time (t)
```

* **If $t_{rise}$ is too fast**: The power rail suffers from severe inductive ringing ("zigzag" oscillation), introducing high voltage spikes into sensitive MCU logic.
* **If $t_{rise}$ is too slow**: MCU fails to exit reset reliably due to POR timing constraints.
* **Design Solution**: Select $C_9, C_{10}, C_{11}$ such that $t_{rise} \approx 100\ \mu\text{s}$, giving a 3dB cutoff frequency of:

$$f_c = \frac{1}{2\pi R C} \approx \frac{0.35}{t_{rise}} \approx 3.38\text{ kHz}$$

This dampens power rail oscillations while ensuring rapid, reliable MCU boot transitions.

---

## 📐 PCB Layout & Routing Guidelines

1. **Capacitor Placement**: $C_{11}$ ($100\text{ nF}$) is routed as close as physically possible to the power pin of $U1$ and $U2$ before connection to the main ground plane.
2. **Ground Plane**: Top and bottom layers utilize copper pour planes tied together with low-impedance vias to minimize loop inductance.
3. **Trace Widths**: Power distribution traces ($VIN, VCC, 5\text{V}, 3.3\text{V}$) are routed with wide copper traces ($> 0.8\text{ mm}$) to handle motor currents, while signal traces maintain $0.25\text{ mm}$ default widths.

---

## 📜 Project References

* **Microcontroller**: STM32F103C8T6 Hardware Development Manual.
* **Regulator**: LM7805 Linear Voltage Regulator Datasheet.
* **Course Context**: EE4053 Embedded Systems Design (PBL), Ho Chi Minh City University of Technology (HCMUT).