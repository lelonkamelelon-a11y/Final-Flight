# Final Flight — ESP32-S3 Flight Computer

> Review-request README. This is the project description written for an Arduino forum post, kept here as the repository README so the design intent, parts choices and open questions are all in one place.

**Board:** 4-layer EasyEDA Pro PCB · roughly circular outline · four PTR mounting holes · outward-facing connectors.

**Purpose:** a model-rocket / high-altitude payload flight computer that logs high-G acceleration, 6-axis IMU data, barometric altitude and GPS position; drives a status buzzer and LED; monitors the battery; and provides an optocoupler-isolated switched 12 V output for a deployment device.

---

## [Review Request] "Final Flight" – ESP32-S3 flight computer / telemetry & deployment board (4-layer, GPS, IMU, high-G accel, barometer, opto-isolated 12 V output)

Hi everyone,

I'd like to put my **Final Flight** PCB in front of the community for a design review before I finish the last bits of routing and order it. It's an ESP32-S3 based flight computer intended for model-rocket / high-altitude payload use: it logs acceleration (including very high-G events), altitude/pressure and GPS position, sounds a buzzer beacon, monitors the battery, and has an optocoupler-isolated switched 12 V output for a deployment device or other payload. The board is done in EasyEDA Pro, is 4 copper layers, has an approximately circular ("circual-ish") outline, four PTR-style mounting holes, and all connectors face outward.

I'm an intermediate PCB person, so please be brutal about anything I've got wrong — especially power, the buck converter, the optocoupler/MOSFET output, and anything that will bite me on a real flight.

---

### 1. What the board is supposed to do

The mission profile I designed for is:

1. **Before launch:** battery connected through the XT30, USB-C available for programming/charging debug, GPS acquiring, buzzer indicating status.
2. **During flight:** record a high-rate inertial trace from a 6-axis IMU **and** a separate ±200 g accelerometer, plus barometric altitude and GPS position/time. The ESP32-S3 logs to its 16 MB flash / 8 MB PSRAM and can stream telemetry over Wi-Fi/BLE if a receiver is in range.
3. **At apogee / on a detected event:** the ESP32 fires the optocoupler, which turns on an N-MOSFET and closes a switched 12 V output on the screw terminal — intended for a deployment charge, release mechanism, or similar payload.
4. **After landing:** GPS position + buzzer beacon to help find the board, and the logs can be pulled over USB.

So: flight data recorder + altimeter + high-G logger + GNSS tracker + one isolated switched output.

---

### 2. Main parts

| Function | Part | Notes |
|---|---|---|
| MCU | ESP32-S3-WROOM-1-N16R8 (U10) | 16 MB flash, 8 MB PSRAM, Wi-Fi/BLE, native USB |
| 6-axis IMU | LSM6DS3TR (U1) | accel + gyro, shared I²C bus |
| High-G accelerometer | ADXL375BCCZ (U2) | ±200 g, for launch/impact events |
| Barometer / altimeter | BMP388 (U3) | pressure + temperature, shared I²C bus |
| GNSS | ATGM336H-5N31 (U7) | GPS/BeiDou, 1PPS output, external active antenna via IPEX |
| Buck converter | TPS563201DDCR (U11) | 3 A synchronous step-down, 12 V → logic rail |
| Output switch | AO3400A N-MOSFET (U5) | low-side switch for the 12 V output |
| Isolation | PC817C optocoupler (U9) | ESP32 GPIO → MOSFET gate, with 10 V zener clamp |
| Battery/12 V input | XT30UPB-M (CN1) | reverse protected through a series diode |
| 12 V output | KF350-3.5-2P screw terminal (U4) | one permanent 12 V pin, one switched pin |
| USB | USB-C 16-pin (USB1) | native USB D+/D−, 5.1 k CC resistors |
| Buzzer | 2.7 kHz magnetic buzzer | driven from a GPIO through a coupling cap |
| Status LED | 0805 LED | driven from GNSS 1PPS |

Supporting parts: 4.7 k I²C pull-ups, 100 k/22 k battery sense divider, GPS reset pull-up, ESP32 EN and BOOT pull-ups/RC, 10 V zener on the MOSFET gate, LC filter/choke for GNSS antenna bias, and a pile of 0603/1206 bypass capacitors.

---

### 3. Power architecture

- **Input:** the XT30 carries `12V+` / `12V-`. Positive goes through D3 (a 1N4007-equivalent S1M) for reverse-polarity protection; negative is the return.
- **Logic rail:** `12V+` feeds U11 (TPS563201) with L2 (10 µH) as the output inductor. The feedback divider is R10 = 33 kΩ / R11 = 10 kΩ into `BUCK_FB`, intended to set the 3V3 rail that powers the ESP32, sensors, GNSS and optocoupler LED side. C15/C16/C18/C19 provide bulk and bypass.
- **Switched output:** `12V+` also feeds the collector side of U9 (PC817). The ESP32 drives U9's LED through R1 (220 Ω) from `PR1`. The optocoupler's emitter drives the gate of U5 (AO3400A) through R2 (10 kΩ), with D4 (BZT52C10, 10 V) clamping Vgs. U5's source is `12V-`, its drain goes to `12V-OUT` on the screw terminal. So an external load connected between the terminal's permanent `12V+` and `12V-OUT` is switched on when the GPIO fires. The opto keeps the ESP32 galvanically separated from the 12 V side.
- **USB:** VBUS comes in through D1/D2 (also S1M diodes) to a `5V+` net; CC1/CC2 have 5.1 kΩ pulls to ground. D+/D− go straight to the ESP32's native USB pins.
- **Sense:** R14 (100 kΩ) / R15 (22 kΩ) divide the 12 V rail down to `BATT_SENSE` for an ESP32 ADC input.

The 12 V nets (`12V+`, `12V-`, `12V+OUT`, `12V-OUT`) are routed at 20 mil where there was room, with a few short local sections at 10 mil where needed to hold the 6 mil / 0.152 mm clearance rule. There are no copper pours or zones yet — everything is tracks.

---

### 4. Signal/interface details

- **I²C:** SDA/SCL are shared by U1, U2, U3 and the ESP32, with 4.7 kΩ pull-ups to 3V3.
- **Interrupts:** the IMU's INT1 and the ADXL375's INT1/INT2 go to ESP32 GPIOs.
- **GNSS:** TX/RX to a UART, 1PPS to a GPIO (and to the status LED), reset controlled through R13, antenna bias fed through L1 (47 nH) to the IPEX connector.
- **Buzzer:** driven from a GPIO through C9 as a coupling capacitor.
- **Boot:** R12 pulls IO0 high and SW1 is the boot button (see "known issues").
- **EN:** R8 pull-up plus C14 for power-on reset delay.

---

### 5. Physical / layout

- 4 copper layers (top, two inner, bottom), mostly 10 mil signal tracks and 20 mil 12 V tracks.
- Roughly circular board outline, four mounting holes placed symmetrically near the corners.
- All external connections (USB-C, XT30, screw terminal, IPEX, buzzer) are oriented outward toward the board edge.
- 54 components, 231 pads, 582 tracks, 66 vias, 0 fills/zones at the moment.

---

### 6. Current status

- Schematic ERC: **0 errors** (2 harmless native warnings).
- PCB DRC: **0 errors** after the latest cleanup.
- The board is saved and consistent between schematic and PCB.
- Remaining routing is being finished by hand; a full autorouter was deliberately **not** run.

Changes made during this cleanup that reviewers should know about:

- Removed C7 (a 10 µF/25 V bypass cap on the 12 V input) from both schematic and PCB, including its leftover copper stub and orphan net port.
- Replaced D1/D2/D3 with **S1M (SMA) 1N4007-equivalents**, JLCPCB part **C55042651** (stock ~2500), keeping the same references and nets.
- Replaced L2 with **CKST0503-10uH/M**, 5.7 × 5.1 mm, JLCPCB **C3002622**. Caveat: it's rated **2.5 A / 3.5 A Isat** versus the original 3 A part — please comment if you think that's marginal for this buck.
- Fixed a schematic/PCB inconsistency where U5.2 (MOSFET source) had ended up on both `12V+` and `12V-`; it's now correctly on `12V-`.

---

### 7. Specific things I'd like reviewed

1. **Buck output voltage:** with R10 = 33 kΩ and R11 = 10 kΩ, does the TPS563201 feedback actually give 3.3 V? I want to be sure before powering the ESP32.
2. **L2 current rating:** 2.5 A rated / 3.5 A Isat vs the 3 A original – acceptable for an ESP32-S3 + sensors + GNSS, or should I step up?
3. **MOSFET output:** is the AO3400A gate drive from the PC817 with a 10 V zener and 10 kΩ pulldown sane? Switching a deployment load would be a fairly inductive event — should I add a flyback diode or TVS across the output?
4. **Boot button:** SW1 currently has one side on a dead-end net (`BZ_SW_NC`) and the other on the IO0/BOOT net, so the button doesn't actually pull IO0 low. What's the cleanest fix — tie SW1.1 to GND and drop the stray net?
5. **5V+ net:** after D1/D2 the `5V+` net currently only exists between the two diodes — it doesn't feed anything else. Did I miss a connection to the buck input or a charge path, or is USB power intentionally debug-only?
6. **GNSS antenna bias:** is the 47 nH choke plus the module's internal bias enough for a typical active patch antenna on the IPEX?
7. **General:** decoupling, 4-layer stackup, trace widths for the 12 V paths, mounting-hole keep-outs, and anything else that jumps out.

Happy to post screenshots, the netlist, or the EasyEDA project files if that helps. Thanks in advance — I'd rather find these problems on the forum than 300 m up.

---

### Short version (for the thread title/first line)

ESP32-S3 flight computer with LSM6DS3 + ADXL375 high-G + BMP388 + GPS, opto-isolated switched 12 V output, 4 layers – looking for a sanity check on the power tree and deployment switch before I finish routing.
