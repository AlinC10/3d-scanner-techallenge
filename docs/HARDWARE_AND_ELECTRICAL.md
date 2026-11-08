# Hardware Specifications, Kinematics, and Electrical Design

This document provides an in-depth reference for the mechatronic architecture, kinematic derivations, electrical design, and safety protocols of the autonomous photogrammetric 3D scanner.

---

## 1. Mechatronic Architecture & Kinematics

The scanner operates as a cylindrical-coordinate manipulator $(r, \theta, z)$:
* **Rotational Axis ($\theta$):** The object rests on a turntable driven by a NEMA 17 stepper motor mounted directly on an industrial axial thrust ball bearing (51124), ensuring zero axial wobble even under heavy, off-center mechanical loads.
* **Vertical Linear Axis ($z$):** The dual-camera carriage translates along parallel MGN9 linear guide rails, actuated by a T8 trapezoidal lead screw ($8\text{ mm}$ lead) equipped with a spring-loaded anti-backlash nut.
* **Structural Frame:** Constructed from rigid $20 \times 20\text{ mm}$ V-slot aluminum extrusions, die-cast corner brackets, and enclosed panels treated with a matte, diffuse white interior surface to eliminate ambient reflections and suppress background optical features.

```
                  [ Linear Rail MGN9 + T8 Lead Screw ]
                                  │
                   ▲              │  Z-Carriage Travel (0 - 475 mm)
                   │              ▼
       ┌──────────────────────────────────────┐
       │     Rigid Horizontal Carriage Bar    │
       └───┬──────────────────────────────┬───┘
           │                              │
     [ Camera 0 ]                   [ Camera 1 ]
   Sony IMX477 12.3MP             Sony IMX477 12.3MP
    (3.9mm M12 Lens)               (3.9mm M12 Lens)
    Height: ~135 mm                Height: ~135 mm
    Pitch:  14°-15° down           Pitch:  14°-15° down
           \                              /
            \       ◄── 27° ──►          /
             \  (Horizontal Azimuth)    /
              \                        /
               ▼                      ▼
       ════════════════════════════════════════
             [ Ø345 / Ø260 mm Turntable ]
             AprilTag Ring (Tag36h11 / tag16h5)
             20 stops per revolution (18° step)
```

### 1.1. Kinematic Resolution

For a stepper motor with full-step angle $\theta_s = 1.8^\circ$ and microstepping division $\mu$:
$$\delta\theta = \frac{\theta_s}{\mu}$$

* **Turntable Platter:**
  * Driver: TB6600 configured at $\mu_t = 16$ microstepping.
  * Angular resolution: $\delta\theta_t = \frac{1.8^\circ}{16} = 0.1125^\circ/\text{microstep}$.
  * For a nominal rotation step of $\alpha = 18^\circ$, the required pulse count is an exact integer:
    $$N_{18^\circ} = \frac{18^\circ}{0.1125^\circ} = 160\text{ microsteps}$$
  * At the perimeter of the $\varnothing 260\text{ mm}$ turntable ($R = 130\text{ mm}$), the arc resolution is:
    $$s = R \cdot \delta\theta_t = 0.255\text{ mm}$$
* **Vertical Axis (Z):**
  * Lead screw pitch: $p_L = 8\text{ mm/revolution}$.
  * Microstepping: $\mu_z = 4$ or $8$.
  * Linear resolution:
    $$\delta z = \frac{p_L \cdot \theta_s}{360^\circ \cdot \mu_z} = 0.01\text{ mm} \quad (10\,\mu\text{m})$$
  * A standard carriage lift of $\Delta z = 100\text{ mm}$ maps precisely to $N_z = 10^4\text{ microsteps}$.

---

## 2. Acquisition Geometry & Vernier Sampling

The two Sony IMX477 cameras are mounted at a radial distance $d \approx 352\text{ mm}$ from the platter rotation axis, with a horizontal angular separation $\beta = 27^\circ$ and a calibrated stereo baseline $B = 164.3\text{ mm}$.

### Uniform Interleaved Sampling Condition (Vernier Ratio)
To maximize viewpoint density around the object without adding physical motor stops, the camera angular separation $\beta$ and the turntable step angle $\alpha$ satisfy:
$$\beta \equiv \frac{\alpha}{2} \pmod{\alpha} \iff \beta = \left(m + \frac{1}{2}\right)\alpha, \quad m \in \mathbb{N}$$

For $\alpha = 18^\circ$ and $m = 1$:
$$\beta = \left(1 + \frac{1}{2}\right) \times 18^\circ = 27^\circ$$

* **Operational Effect:** At each motor stop, Camera 1 samples a viewpoint shifted by $9^\circ$ relative to Camera 0's previous capture.
* Across 20 mechanical turntable stops, the system acquires **40 distinct horizontal perspectives at $9^\circ$ intervals** across the full $360^\circ$ circle.
* Compared to non-interleaved or narrow baselines (e.g., $\beta = 20^\circ$), this geometry increases triangulation depth precision by over $26\%$:
  $$\mathrm{GSD} \approx \frac{d}{f_{px}} \approx 0.14\text{ mm/px}, \qquad \sigma_Z \approx \frac{\mathrm{GSD} \cdot \sigma_{px}}{2\sin(\beta/2)} \approx 0.15\text{ mm}$$

---

## 3. Real-Time Motion Control & Emergency Safety

Pulse generation is offloaded to a dedicated **Arduino Uno R3 (ATmega328P @ 16 MHz)** communicating via an isolated USB serial link at 115,200 baud.

### Why an External MCU?
Standard Linux (running on the Raspberry Pi 5) introduces process scheduling jitter in user space between $10\,\mu\text{s}$ and several hundred microseconds. Because the nominal step interval is $t_c = 1\text{ ms}$, this timing uncertainty excites mechanical resonance and causes skipped steps. The ATmega328P generates pulse trains timed down to individual clock cycles ($62.5\text{ ns}$ resolution), guaranteeing completely deterministic motion.

### Trapezoidal Velocity Profile
The firmware executes a smooth trapezoidal acceleration profile:
* Cruise period: $t_c = 1\text{ ms}$ (cruise angular velocity $\omega = 112.5^\circ/\text{s}$).
* Starting period: $t_s = 3 t_c = 3\text{ ms}$.
* Acceleration / deceleration ramps: $N_a = N_d = 20\%$ of total step count (32 steps for an $18^\circ$ move).
* A complete $18^\circ$ rotation executes in just $T = 224\text{ ms}$, enabling fast, vibration-free indexing.

### Out-of-Band Emergency Abort (`0xFF`)
To ensure operator and equipment safety:
* Between every individual motor step, the microcontroller checks the hardware UART receive register.
* Sending a single out-of-band byte `0xFF` instantly triggers a hardware interrupt that cuts driver power (de-energizing motor coils).
* The worst-case abort latency is strictly bounded under $4.1\text{ ms}$:
  $$t_{\text{abort}} \le \frac{10\text{ bits}}{115,200\text{ Bd}} + \max_i t_i \approx 0.087\text{ ms} + 4.0\text{ ms} \approx 4.09\text{ ms}$$

---

## 4. Illumination Subsystem & Zero-Current Switching (ZCS)

Studio illumination is provided by three vertical 12V LED strips (120 SMD 2835 LEDs/m) configured for uniform, shadow-free light diffusion.

* **Flicker-Free Dimming on Rolling Shutter Sensors:**
  * LED brightness is modulated via a D4184 power MOSFET driven by the RP1 hardware PWM controller at **$10\text{ kHz}$** (GPIO 12).
  * At $10\text{ kHz}$, the residual optical ripple integrated across an IMX477 rolling-shutter exposure window ($E \approx 10\text{ ms}$) is under $0.5\%$, completely eliminating horizontal banding.
* **Zero-Current Switching (ZCS) Relay Isolation:**
  * To prevent mechanical relay contact arcing and contact welding:
  * **Turn-on:** The relay contacts close while PWM duty cycle is held at $0\%$. The target brightness duty cycle is applied only after contact settling ($50\text{ ms}$).
  * **Turn-off:** The PWM duty cycle is returned to $0\%$ before the mechanical relay opens. The relay never switches live load current.

---

## 5. Time-of-Flight Laser Telemetry (VL53L1X)

To automate scan boundaries and eliminate wasted captures, an STMicroelectronics VL53L1X Class 1 eye-safe laser rangefinder ($940\text{ nm}$ VCSEL) monitors the turntable center:
* **The Cone Overlap Challenge:** Generic ToF sensors (e.g., VL53L0X) feature an unconfigurable $25^\circ$ field-of-view cone, causing spurious distance returns from the turntable rim or enclosure back wall ($4 - 8\text{ cm}$ errors).
* **Programmed SPAD ROI:** The VL53L1X features a programmable Single-Photon Avalanche Diode (SPAD) array. The scanner restricts the optical aperture to a custom $4 \times 4$ ROI (centered at index 58), narrowing the field of view to $\approx 15^\circ$ and tilting the optical axis $5-6^\circ$ downwards.
* **Adaptive Height Termination:** Following each $360^\circ$ revolution, the laser inspects the platter center. When the measured distance clears the object threshold ($> 350\text{ mm}$, hitting the background), the scanner terminates the vertical climb automatically.

---

## 6. Electrical Architecture, Power Budget, and Pinout

The complete KiCad electrical schematic is available in:
📄 **[info/3d_scanner_electrical_scheme.pdf](../info/3d_scanner_electrical_scheme.pdf)**

### Power Distribution & Consumption Budget
* 2× TB6600 Stepper Drivers: $\approx 36\text{ W}$ peak
* Raspberry Pi 5 + Active Cooler: $\le 27\text{ W}$ peak
* 12V Studio LED Arrays (100% duty cycle): $\approx 48\text{ W}$
* 11.6" Capacitive Touch Display: $\approx 10\text{ W}$
* **Total Peak Power:** $\approx 121\text{ W}$
* **Installed Power Supply:** Industrial 12V / 30A switched-mode PSU ($360\text{ W}$), providing a robust $3\times$ safety headroom.

### Controller Pinout Mapping

| Signal / Peripheral | Controller Pin | Protocol / Signal Type |
|---|---|---|
| **VL53L1X ToF (SDA / SCL)** | RPi GPIO 2 / GPIO 3 | I2C Bus 1 (Address `0x29`) |
| **Z-Axis Limit Switches (Min / Max)** | RPi GPIO 4 / GPIO 22 | Active-LOW digital interrupts, internal pull-up |
| **LED Relay Cut-Off** | RPi GPIO 11 | Digital Output (ZCS sequencing) |
| **LED Dimmer MOSFET** | RPi GPIO 12 | RP1 Hardware PWM @ 10 kHz |
| **Camera 0 & 1 CSI Ports** | RPi CSI0 / CSI1 | Dual 4-lane MIPI CSI-2 |
| **Camera VCM Autofocus** | I2C-10 / I2C-11 | I2C (Addresses `0x0c` / `0x0e`) |
| **Audible Feedback Buzzer** | RPi GPIO 23 | Digital Output (piezo audio alerts) |
| **Turntable (EN / STEP / DIR)** | Arduino Pin D2 / D3 / D4 | Optoisolated signals to TB6600 |
| **Z-Axis (EN / STEP / DIR)** | Arduino Pin D5 / D6 / D7 | Optoisolated signals to TB6600 |
| **RPi to Arduino Link** | USB Port `/dev/ttyACM0` | Serial UART @ 115,200 baud |
