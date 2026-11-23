# Stick Of Power — ESP32-S3 Flight Computer

> Review-request README for the Arduino Forum post, kept here next to the EasyEDA project so the design intent, part choices and open questions are all in one place.

**GitHub:** https://github.com/lelonkamelelon-a11y/StickOfPower

**EasyEDA project:** [Final Flight_copy.eprj2](https://github.com/lelonkamelelon-a11y/StickOfPower/blob/main/Final%20Flight_copy.eprj2)

**Forum post:** https://forum.arduino.cc/t/review-request-stick-of-power-esp32-s3-flight-computer-with-gps-imu-high-g-accel-barometer-lora-and-isolated-12-v-output/1468733

**Board:** 4-layer EasyEDA Pro PCB · roughly circular ("circual-ish") outline · four PTR mounting holes · outward-facing connectors. The outline was extended twice (20 mm total) to make room for the RF and interface sections.

**Purpose:** a model-rocket / high-altitude payload flight computer that logs high-G acceleration, 6-axis IMU data, barometric altitude and GPS position; drives a status buzzer and LED; monitors the battery; and provides an optocoupler-isolated switched 12 V output for a deployment device.

---

## [Review Request] "Stick Of Power" – ESP32-S3 flight computer / telemetry & deployment board (4-layer, GPS, IMU, high-G accel, barometer, LoRa, opto-isolated 12 V output)

Hi everyone,

I'd like to put my **Stick Of Power** PCB in front of the community for a design review before I order it. It is an ESP32-S3 based flight computer intended for model-rocket / high-altitude payload use: it logs acceleration (including very high-G events), altitude/pressure and GPS position, sounds a buzzer beacon, monitors the battery, has an optocoupler-isolated switched 12 V output for a deployment device, and now also carries an 868 MHz LoRa radio, a microSD socket and a 6-pin debug header. The board is done in EasyEDA Pro, uses 4 copper layers, has an approximately circular ("circual-ish") outline, four PTR-style mounting holes, and all connectors face outward.

I'm an intermediate PCB person, so please be brutal about anything I've got wrong — especially power, the buck converter, the optocoupler/MOSFET output, the RF layout and anything that will bite me on a real flight.

---

### 1. What the board is supposed to do

1. **Before launch:** battery connected through the XT30, USB-C available for programming/debug, GPS acquiring, buzzer indicating status.
2. **During flight:** record a high-rate inertial trace from a 6-axis IMU **and** a separate ±200 g accelerometer, plus barometric altitude and GPS position/time. The ESP32-S3 logs to its 16 MB flash / 8 MB PSRAM or to microSD, and can stream telemetry over Wi-Fi/BLE or LoRa if a receiver is in range.
3. **At apogee / on a detected event:** the ESP32 fires the optocoupler, which turns on an N-MOSFET and closes a switched 12 V output on the screw terminal — intended for a deployment charge, release mechanism, or similar payload.
4. **After landing:** GPS position + buzzer beacon to help find the board; logs can be pulled over USB or microSD.

So: flight data recorder + altimeter + high-G logger + GNSS tracker + LoRa downlink + one isolated switched output.

---

### 2. Main parts

| Function | Part | Notes |
|---|---|---|
| MCU | ESP32-S3-WROOM-1-N16R8 (U10) | 16 MB flash, 8 MB PSRAM, Wi-Fi/BLE, native USB |
| 6-axis IMU | LSM6DS3TR (U1) | accel + gyro, shared I²C bus |
| High-G accelerometer | ADXL375BCCZ (U2) | ±200 g, for launch/impact events |
| Barometer / altimeter | BMP388 (U3) | pressure + temperature, shared I²C bus |
| GNSS | ATGM336H-5N31 (U7) | GPS/BeiDou, 1PPS output, external active antenna via IPEX |
| LoRa transceiver | RFM95W / SX1276 868 MHz (U14) | SPI + DIO0/RESET on ESP32 |
| LoRa antenna | SMA (ANT1) | 868 MHz, outward-facing |
| microSD socket | U15 | dedicated SPI bus |
| Debug header | J1, 1×6 2.54 mm | VCC, GND, TXD0, RXD0, BOOT_SW, EN |
| Buck converter | TPS563201DDCR (U11) | 3 A synchronous step-down, 12 V → logic rail |
| Output switch | AO3400A N-MOSFET (U5) | low-side switch for the 12 V output |
| Isolation | PC817C optocoupler (U9) | ESP32 GPIO → MOSFET gate, with 10 V zener clamp |
| Battery/12 V input | XT30UPB-M (CN1) | reverse protected through a series diode |
| 12 V output | KF350-3.5-2P screw terminal (U4) | one permanent 12 V pin, one switched pin |
| USB | USB-C 16-pin (USB1) | native USB D+/D−, 5.1 k CC resistors |
| USB 3.3 V regulator | AP2112K-3.3 (U12) | 5V+ → USB_3V3 |
| USB power OR-ing | MAX40200 (U13) | USB_3V3 → VCC, ideal-diode combined with buck rail |
| Bulk caps | C24 / C25, 10 µF | on 5V+ and VCC respectively |
| Buzzer | 2.7 kHz magnetic buzzer | driven from a GPIO through a coupling cap |
| Status LED | 0805 LED | driven from GNSS 1PPS |

Supporting parts: 4.7 k I²C pull-ups, 100 k/22 k battery sense divider, GPS reset pull-up, ESP32 EN and BOOT pull-ups/RC, 10 V zener on the MOSFET gate, LC filter/choke for GNSS antenna bias, C20/C21 1 µF on the USB supply, and 0603/1206 bypass capacitors.

---

### 3. Power architecture

- **Input:** the XT30 carries `12V+` / `12V-`. Positive goes through D3 (a 1N4007-equivalent S1M) for reverse-polarity protection; negative is the return.
- **Logic rail:** `12V+` feeds U11 (TPS563201) with L2 (10 µH) as the output inductor. The feedback divider is R10 = 33 kΩ / R11 = 10 kΩ into `BUCK_FB`, intended to set the 3V3 rail that powers the ESP32, sensors, GNSS and optocoupler LED side.
- **Switched output:** `12V+` also feeds the collector side of U9 (PC817). The ESP32 drives U9's LED through R1 (220 Ω) from `PR1`. The optocoupler's emitter drives the gate of U5 (AO3400A) through R2 (10 kΩ), with D4 (BZT52C10, 10 V) clamping Vgs. U5's source is `12V-`, its drain goes to `12V-OUT` on the screw terminal.
- **USB power:** VBUS comes in through D1/D2 (also S1M diodes) to `5V+`, bypassed by C24 (10 µF). U12 (AP2112K-3.3) regulates that to `USB_3V3` with C20/C21 (1 µF), and U13 (MAX40200) ideal-diode ORs it into `VCC` alongside the buck rail. C25 (10 µF) bypasses VCC near the regulators. CC1/CC2 have 5.1 kΩ pulls to ground; D+/D− go straight to the ESP32 native USB pins.
- **Sense:** R14 (100 kΩ) / R15 (22 kΩ) divide the 12 V rail down to `BATT_SENSE` for an ESP32 ADC input.

The 12 V nets (`12V+`, `12V-`, `12V+OUT`, `12V-OUT`) were made much beefier after the initial routing pass: 19 segments are 1.5 mm, 23 are 1.0 mm, 12 are 0.6 mm and 6 short constrained sections remain at the original ~0.25 mm. Widths were chosen segment-by-segment with DRC verification, so the widening added **zero** new clearance violations. There are no copper pours or zones yet — everything is tracks.

---

### 4. Signal/interface details

- **I²C:** SDA/SCL are shared by U1, U2, U3 and the ESP32, with 4.7 kΩ pull-ups to 3V3.
- **Interrupts:** the IMU's INT1 and the ADXL375's INT1/INT2 go to ESP32 GPIOs.
- **GNSS:** TX/RX to a UART, 1PPS to a GPIO (and to the status LED), reset controlled through R13, antenna bias fed through L1 (47 nH) to the IPEX connector.
- **LoRa (U14):** SCK = IO13, MISO = IO14, MOSI = IO21, CS = IO5, DIO0 = IO47, RST = IO48; DIO1–DIO5 are not connected. ANT1 is an outward-facing SMA.
- **microSD (U15):** dedicated SPI — SCK = IO38, MISO = IO15, MOSI = IO16, CS = IO2.
- **Debug header (J1):** 1 VCC, 2 GND, 3 TXD0, 4 RXD0, 5 BOOT_SW, 6 EN.
- **Buzzer:** driven from a GPIO through C9 as a coupling capacitor.
- **Boot:** R12 pulls IO0 high and SW1 is the boot button (see "specific things I'd like reviewed").

---

### 5. Physical / layout

4 copper layers, mostly 10 mil signal tracks and widened 12 V tracks as described above. Roughly circular outline with four symmetric PTR mounting holes; all external connections (USB-C, XT30, screw terminal, SMA antennas, IPEX, buzzer) are oriented outward toward the board edge. The outline was extended by 10 mm twice (20 mm total) to fit the antenna and connector area, keeping the rounded corners.

---

### 6. Current status

- **Schematic ERC:** 0 errors / 4 warnings — passed.
- **PCB DRC:** 0 errors.
- All routing was done by hand; no autorouter was used. Small nudges of neighbouring tracks were only used where needed.
- Current project: [Final Flight_copy.eprj2](https://github.com/lelonkamelelon-a11y/StickOfPower/blob/main/Final%20Flight_copy.eprj2)

Changes made during this cleanup:

- Added LoRa (RFM95W/SX1276 868 MHz) with an outward-facing SMA antenna, plus C22/C23.
- Added the USB power path: AP2112K-3.3 → MAX40200 → VCC, with C20/C21 and bulk caps C24/C25.
- Added a 6-pin debug/serial header (J1) and a microSD socket (U15) in the extended board area.
- Extended the board outline twice by 10 mm while preserving the rounded corners and mounting holes.
- Replaced D1/D2/D3 with S1M (SMA) 1N4007-equivalents, JLCPCB C55042651.
- Replaced L2 with CKST0503-10uH/M, JLCPCB C3002622 (2.5 A / 3.5 A Isat — please comment if that is marginal).
- Fixed a schematic/PCB inconsistency where U5.2 (MOSFET source) had ended up on both `12V+` and `12V-`; it is now correctly on `12V-`.

---

### 7. Specific things I'd like reviewed

1. **Buck output voltage:** with R10 = 33 kΩ and R11 = 10 kΩ, does the TPS563201 feedback actually give 3.3 V?
2. **L2 current rating:** 2.5 A rated / 3.5 A Isat vs the 3 A original – acceptable for ESP32-S3 + sensors + GNSS + LoRa?
3. **MOSFET output:** is the AO3400A gate drive from the PC817 with a 10 V zener and 10 kΩ pulldown sane for an inductive deployment load? Should I add a flyback diode or TVS across the output?
4. **Boot button:** SW1 currently has one side on a dead-end net (`BZ_SW_NC`). What is the cleanest fix — tie SW1.1 to GND and drop the stray net?
5. **USB power path:** is the AP2112K-3.3 → MAX40200 arrangement into VCC correct, or will it back-feed the buck rail? Any layout advice for the ideal-diode OR-ing?
6. **LoRa layout:** is the 868 MHz RF section (RFM95W, SMA, C22/C23) far enough from the buck converter and ESP32, and is the antenna keep-out adequate?
7. **GNSS antenna bias:** is the 47 nH choke plus the module's internal bias enough for a typical active patch antenna on the IPEX?
8. **General:** decoupling, 4-layer stackup, 12 V trace widths, mounting-hole keep-outs, and anything else that jumps out.

Happy to post screenshots, the netlist, or the EasyEDA project files if that helps. Thanks in advance — I'd rather find these problems on the forum than 300 m up.
