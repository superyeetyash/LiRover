# Liros — Spin-Stabilized Rocket LiDAR Terrain Mapper

*(codename: stardance + outpost)*

A model rocket that maps the terrain around its launch site in 3D during ascent. Canted fins
spin the rocket as it climbs; a single fixed LiDAR pointing out the side and angled down sweeps a
helical pattern as the rocket rises and rotates. We log range + orientation + altitude at high
rate and reconstruct a 3D point cloud on the ground afterward.

The spin does double duty: it stabilizes the rocket **and** sweeps the one-axis sensor across the
scene, so we avoid a gimbal or a multi-beam LiDAR.

- **Flight system ("stardance"):** the airframe, Pico flight computer, and sensor logger.
- **Ground system ("outpost"):** the synthetic-flight simulator, reconstruction pipeline, and the
  bench spin-rig that proves the whole pipeline without flying.

> **Status / honest framing:** The **bench spin-rig + software is the committed deliverable** — it
> is 100% in our control, needs no hardware beyond the sensors, and reconstructs a real room/yard.
> The **flight is a stretch goal**, gated on (a) parts arriving and the build finishing with
> margin and (b) a legal place to fly — a club launch *or* a suitable open field with your own
> pad (§15; no club, FAA notification, or waiver is legally required at this scale). Plan and demo
> accordingly.

---

## 1. Success criteria (calibrated expectations)

- **Minimum success:** a recognizable *coarse* point cloud — large buildings read as blobs,
  treelines as walls, the ground plane and big edges are clear. This is the realistic ceiling for
  this hardware and is genuinely impressive.
- **Not the goal:** survey-grade accuracy. That needs RTK GPS + industrial IMUs + calibrated
  scanners costing tens of thousands. Objects landing within ~1–3 m of true position is a win.
- **The bench demo is itself a success.** If the flight fails or never happens, a spin-rig on the
  ground that reconstructs a room/yard proves the entire pipeline and is a complete project.

---

## 2. How it works, end to end

1. **Spin** — canted fins induce roll during ascent. **Target ≤ 10 rev/s** (see §3.1 — this is a
   hard sensor limit, not a preference).
2. **Scan** — the LiDAR, fixed in the airframe pointing out a side window at a downward cant
   `c`, measures distance to whatever the beam hits.
3. **Know the angle** —
   - **Roll (azimuth) comes from the magnetometer**, not the gyro. The horizontal component of
     Earth's field traces one sine cycle per revolution in the body frame; its phase is the
     drift-free absolute roll angle.
   - **Tilt (pitch/yaw) comes from the accelerometer + the *transverse* gyro axes** (the roll-axis
     gyro is saturated — see §3.1).
   - **Altitude comes from the barometer** — the single most reliable position channel.
4. **Log** — the Pico timestamps **each sensor read individually** and records all streams to SD in
   binary at high rate. No processing in flight.
5. **Recover** — parachute at apogee; a ball-bearing swivel keeps residual spin from tangling the
   lines.
6. **Reconstruct** — on the ground, Python turns each `(time, roll, tilt, altitude, range)` record
   into a 3D `(x, y, z)` point, stacks them into a cloud, and visualizes in Open3D.

---

## 3. Key engineering constraints (read before building)

These facts determine whether the project actually works. Several correct earlier assumptions —
treat them as binding.

### 3.1 Spin rate is capped by gyro saturation — target ≤ 10 rev/s
Roll rate in °/s is `rev/s × 360`. The IMU gyro full-scale ranges:

| IMU | Max gyro FS | Max spin before roll-axis saturates |
|-----|-------------|-------------------------------------|
| ICM-20649 | ±4000 °/s | **11.1 rev/s** |
| MPU-6050  | ±2000 °/s | **5.5 rev/s** |

At 15 rev/s (5400 °/s) the roll-axis gyro is railed on **both** parts. Therefore:

- **Roll is recovered from the magnetometer, full stop.** The roll-axis gyro is allowed to clip.
- Set the IMU gyro to a **sensitive range (e.g. ±500 °/s)** so the small *transverse* (pitch/yaw)
  rates used for tilt are well resolved. The roll-axis clipping is intentional and harmless.
- **Cap spin at ≤ 10 rev/s** to keep the spin-up transient out of trouble, and confirm with Test 1.
- The IMU choice is therefore driven **only by the accelerometer g-range** (Test 2). Neither part
  helps the roll axis.

### 3.2 The LiDAR's usable range caps the mapping band to roughly altitude ≤ 30–50 m
With the beam canted `c` below horizontal, from altitude `h`:

```
slant range to ground  = h / sin(c)
ground footprint radius = h / tan(c)
```

At `c = 30°`: slant = `2h`, footprint = `1.73h`. The TF03-100's 100 m spec is for a cooperative
target; on **grass/asphalt at a grazing ~60° incidence the realistic reliable range is ~30–60 m.**
Consequences:

- Near apogee the ground is well beyond range → **no returns at the top** of the flight.
- Useful ground mapping is confined to the **lower altitude band (~15–50 m)**, regardless of how
  high it flies. Expect a **dense-low / empty-high** cloud.
- **Cant angle is a real design parameter**, not a fixed 30°: a steeper cant shortens slant range
  (more altitude covered) but shrinks the footprint (less ground per ring). Pick it from Test 3's
  measured real-terrain returns, then freeze it.
- **Site selection:** at 50 m altitude the footprint radius is only ~50–90 m, so the features you
  want (buildings, treelines) must be within ~60–80 m of the pad. Choose the site for close-in
  features.

### 3.3 Apogee will be several hundred feet — re-derive it, don't assume ~140 ft
A ~600 g rocket on an F40W (T/W ≈ 6.8) reaches **several hundred feet**; drag on a 66 mm tube
(~1–4 N at these speeds) cannot cap it at 140 ft. Capping that low would need ~1 kg+ of mass or a
smaller motor — but a smaller motor drops below safe rail-exit T/W for this mass. The "heavy
enough to fly straight" and "slow enough to scan densely" goals are in tension.

→ **Re-derive apogee and the velocity-vs-altitude curve in OpenRocket with a real drag model
before ordering the airframe** (expect ~250–500 ft). Feed the real curve — not a guess — into the
scan-density estimate. Higher apogee is still fine for FAA (< 400 ft AGL) and widens coverage, but
it pushes the far ground past LiDAR range (§3.2), so the two are coupled.

### 3.4 Roll recovery needs careful calibration — and a boresight step that is easy to forget
The magnetometer approach is sound. In near-vertical flight in Illinois (total field ~52 µT,
inclination ~68°) the transverse signal is the **horizontal field component, ~19 µT**. The
MMC5983MA resolves ~6 nT/LSB, so SNR is a non-issue and you are far from the unobservable
singularity (which would require the spin axis to align with the field line). The hard parts are
calibration:

- **Calibrate the *assembled* electronics bay, not the bare sensor** (§9.1). Hard/soft-iron offsets
  come from the LiPo, fasteners, and motor casing.
- **Boresight / sensor-alignment calibration** (§9.2) — recover the true cant angle and the
  rotational offsets between mag-0°, the IMU axes, and the LiDAR boresight, plus the constant LiDAR
  output-latency azimuth offset. **Skip this and the entire cloud is rotated and/or mis-scaled.**
- Exploit that **roll(t) is smooth** (near-constant rate): interpolate the mag-derived phase between
  samples to match every LiDAR shot precisely — finer than the raw mag sample spacing.

### 3.5 Accuracy ceiling (state this up front)
There is **no horizontal position channel** — no GPS, and double-integrating accel over a ~3 s
flight drifts meters. The reconstruction places the sensor at `(0, 0, altitude)` and ignores
weathercock/wind drift, so the output is a **shape** that you then rigidly align to known ground
targets, not an absolutely georeferenced cloud. Tilt error maps strongly to ground error at long
range (5° pitch over a 75 m footprint ≈ 6.5 m). The **bench demo is genuinely easier than flight**
because tilt ≈ 0 and the sensor position is fixed — that is the right framing, not a cop-out.

---

## 4. Bill of materials (~$425; ~$410 if MPU-6050 passes Test 2)

**Already owned ($0):** Raspberry Pi Pico (flight computer), Pi Zero 2 W (field offload only —
see note), DJI Action 2 (optional flight video), soldering kit, Elegoo kit (prototyping +
possible MPU-6050).

**Electronics**

| Part | Source | Cost | Notes |
|------|--------|------|-------|
| TF03-100 LiDAR — **UART** version, used | eBay (returnable) | ~$170 | Order first; scarce. Confirm UART, **not** CAN/RS485. |
| **TFmini-S LiDAR (12 m)** — optional hedge | Amazon/SparkFun | ~$40 | **Highest-leverage spend:** enough range for the bench demo (room/yard < 12 m); unblocks the entire software pipeline before the TF03 arrives, and is a backup if the TF03 is DOA. |
| MMC5983MA magnetometer (Qwiic, I2C) | SparkFun | $18 | Bench its real I2C rate (Test 1). |
| ICM-20649 IMU (or free MPU-6050 if it passes Test 2) | DigiKey | $15 / $0 | Decide on **accel g-range only**. ICM has ±30 g; MPU ±16 g. |
| BMP390 barometer | DigiKey | $11 | Bundle with DigiKey order. |
| MicroSD SPI breakout + 16 GB Class-10 card | DigiKey + Amazon | $10 | Samsung Endurance / SanDisk. |
| 1S LiPo (500–800 mAh) + Pololu U1V11F5 5 V boost | Amazon / Pololu | $16 | Boost feeds the LiDAR 5 V rail; add bulk cap (§6). |

**Rocket**

| Part | Source | Cost | Notes |
|------|--------|------|-------|
| Aerotech RMS 29/40-120 casing + closures | Sirius Rocketry | ~$74 | Verified ~$73.94. |
| 2× E16W or F40W reloads | Apogee | $40 | No certification needed (§15). |
| Body tube, nose cone, centering rings, motor mount, rail buttons | Apogee | ~$25 | Diameter set by OpenRocket — order **after** sim. |
| Parachute, Kevlar shock cord, ball-bearing swivel | Apogee | ~$12 | Kevlar (not elastic); swivel is mandatory. |
| Filament (PETG), epoxy, **non-ferrous fasteners near the mag**, wire, threadlocker, heat shrink | Hardware/Amazon | ~$35 | Brass/aluminum/304–316 stainless near the magnetometer; silicone wire only; threadlock every fastener. |

**Order sequence:** (1) TF03 + TFmini-S today. (2) One DigiKey order: ICM-20649 + BMP390 + SD
breakout. (3) Sirius casing. (4) Everything else after OpenRocket fixes the tube diameter.

> **Pi Zero 2 W note:** it has 512 MB RAM and no clean Open3D wheels (source builds OOM / take
> hours). Run reconstruction + Open3D **on a laptop**; use the Pi only for field data offload and a
> matplotlib quicklook.

---

## 5. Physical layout (nose → tail)

1. Nose cone with bulkhead + threadlocked eyebolt.
2. Recovery bay: parachute + Kevlar shock cord; **ball-bearing swivel** between cord and nose
   eyebolt.
3. Electronics bay: 3D-printed PETG sled holding Pico, magnetometer, IMU, baro, SD breakout, LiPo,
   boost converter. The LiDAR mounts here, aimed out a side window.
4. Motor mount: 29 mm inner tube, centering rings, motor retainer. Fin roots should land over
   centering-ring positions for strength.

**Two placement rules that matter:**

- **Magnetometer far from the LiPo, steel, and the motor casing**, at one end of the sled with
  ferrous parts at the other. Use non-ferrous fasteners nearby. It reads Earth's field; local iron
  is a hard-iron offset that corrupts roll recovery.
- **LiDAR (~77 g) balanced by a ~70 g counterweight opposite the window** so spin is balanced and
  the CG stays on the spin axis.

**The window:** cut a radial hole; the LiDAR aims out at downward cant `c` (set in §3.2). **Leave
the aperture open** — most plastics do not pass 905 nm cleanly and will ruin returns; the LiDAR's
IP67 housing handles the airflow.

---

## 6. Wiring & Pico pin map

```
Raspberry Pi Pico
├── UART0 (GP0 TX → LiDAR RX, GP1 RX ← LiDAR TX) ── LiDAR (powered from 5 V boost)
├── SPI0  (GP2 SCK, GP3 MOSI, GP4 MISO, GP5 CS) ── MicroSD card
├── I2C0  (GP8 SDA, GP9 SCL) ───────────────────── MMC5983MA magnetometer (own bus, high-rate)
├── I2C1  (GP14 SDA, GP15 SCL) ─────────────────── ICM-20649 IMU + BMP390 baro (shared, diff addr)
├── 3V3 OUT ── powers mag, IMU, baro, SD logic
├── VSYS  ──── LiPo 3.7 V input (Pico VSYS range 1.8–5.5 V)
└── 5 V boost converter ── LiDAR 5 V rail
```

- Magnetometer on its **own** I2C bus so its fast polling doesn't compete with IMU/baro.
- Keep all SPI/I2C wires short. Add a ~5 kΩ pull-up on SD MISO if reads are flaky.
- **Add bulk capacitance (≥ 100 µF) on the 5 V LiDAR rail** to ride out ignition/vibration
  transients. Confirm the U1V11F5 sustains the LiDAR's average + startup-peak current from 3.7 V.
- At 1000 Hz the TF03 emits ~9 KB/s; default 115200 baud (~11.5 KB/s capacity) is tight —
  **raise the TF03 baud rate** if you push frame rate to 1000 Hz.

---

## 7. Flight firmware (Pico C SDK)

Use the **Pico C SDK**, not MicroPython — MicroPython can't reliably sustain multi-sensor logging
at this rate.

- **Core 0:** tight loop reading LiDAR (UART), magnetometer (I2C0), IMU + baro (I2C1).
  **Timestamp each sensor read individually** with `time_us_64()` (not one stamp per record) so the
  reconstruction can interpolate continuous `roll(t)` and match LiDAR shots exactly — at 10 rev/s
  the rocket turns 3.6° per ms, so azimuth precision is what makes the cloud sharp. Push records
  into a RAM ring buffer.
- **Core 1:** drains the buffer to SD in binary (convert to CSV on the ground). Use the
  `carlk3/no-OS-FatFS-SD-SPI-RPi-Pico` library.
- **Record (~24–32 bytes):** per-sensor timestamps + `lidar_cm`, `mag_xyz`, `gyro_transverse`,
  `accel_z`, `baro_pressure`. ~1000 rec/s × ~30 B ≈ ~30 KB/s — trivial for the SD card.
- **Periodic flush:** call `f_sync` every ~0.25 s (not only after apogee) and **pre-allocate the
  file** so a hard landing or brown-out keeps almost all data and FAT doesn't corrupt mid-write.
- **Arming + status:** arm via switch or detect liftoff via accel threshold; flush + close the file
  a few seconds after apogee. Drive a **status LED** (armed → liftoff-detected → logging →
  flushed-and-safe) so a dead logger is visible before launch.

---

## 8. Reconstruction pipeline (laptop, Python)

Libraries: NumPy, SciPy, Open3D, Matplotlib. Build and test this against **simulated data** (§10)
before any real data lands so it's ready the moment a log arrives.

1. Parse binary → NumPy arrays.
2. **Roll from magnetometer:** the horizontal-field component traces one sine cycle per revolution;
   its phase is the drift-free absolute roll. Extract phase with SciPy; interpolate `roll(t)`
   between mag samples. Apply hard/soft-iron + boresight calibration (§9).
3. **Tilt from IMU:** complementary or Madgwick filter using accel + **transverse gyro axes only**
   (the roll-axis gyro is saturated). Cleaner during coast than boost.
4. **Altitude from baro** (z-axis) — the most reliable channel.
5. **Per sample:** beam vector from `(roll = azimuth, fixed cant = elevation)`; point =
   `sensor_origin(0, 0, altitude) + range × beam_vector → (x, y, z)`.
6. Assemble into an `(N, 3)` array → Open3D for downsampling, normals, visualization. Matplotlib
   for debug plots (altitude/spin/raw mag vs time). Optionally place bright ground targets and
   rigidly align the cloud to them for a coarse georeference (§3.5).

---

## 9. Calibration procedures (do not skip)

### 9.1 Magnetometer hard/soft-iron — on the *assembled* bay
Run the figure-8 / full-rotation calibration with the **complete electronics bay** (LiPo
installed, a mass dummy for the motor, all real fasteners), not the bare sensor. Fit and store the
hard-iron offset + soft-iron matrix; apply them in §8 step 2.

### 9.2 Boresight / sensor alignment
On the bench rig, aim the LiDAR at a feature at a **known azimuth and range**, spin slowly, and
record all sensors. Solve for: the true downward cant `c`; the rotational offset between mag-0°,
IMU axes, and LiDAR boresight; and the constant LiDAR output-latency azimuth offset (a few ms ≈ a
few degrees at spin). These are constants you bake into the reconstruction. **This is what keeps
the cloud from being globally rotated or mis-scaled.**

---

## 10. Simulator + bench rig (the de-risking core)

- **Synthetic-flight simulator:** generate `(time, roll, tilt, altitude, range)` records from a
  known ground-truth scene and the real OpenRocket velocity/spin profile, including the
  dense-low/empty-high range cutoff (§3.2). Develop and validate the reconstruction against this
  before any hardware exists, then compare the reconstructed cloud to the known scene.
- **Bench spin-rig:** mount the full assembled package on a lazy-Susan / motor at 10 rev/s and
  reconstruct a recognizable room/yard cloud. This proves the whole pipeline (corrections §3.1,
  §3.2, §3.4) end-to-end **without flying** and is the committed demo.

---

## 11. OpenRocket

Free Java app. Build: nose cone → body tube → inner motor mount with centering rings →
trapezoidal canted fins → a mass component for the electronics bay where it physically sits.
Aerotech RMS reloads are in OpenRocket's motor database — swap and compare directly.

You're optimizing for the **opposite of altitude** — a slow climb through the scan band for many
revolutions per foot — but it must still leave the rail fast enough to fly straight.

Starting points to iterate from (validate in sim, do not treat as final):

- **Body tube ~2.6" (66 mm)** — roomy for LiDAR + counterweight + sled; extra drag helps slow the
  climb.
- **Liftoff mass ~550–650 g** (electronics bay ~230 g incl. counterweight).
- **Reload:** start with **F40W** (T/W ≈ 6.8, safe rail exit). E16W (T/W ≈ 2.7) is gentler/longer
  but marginal for getting a heavy build off the rail — only if mass comes out low and rail-exit
  speed still passes.
- **Fin cant:** start ~5° and tune toward ≤ 10 rev/s (the gyro-safe ceiling, §3.1).

Read off and tune:

- **Apogee** — re-derive it (§3.3); expect a few hundred feet, not 140 ft. Keep < 400 ft AGL.
- **Velocity vs altitude** — feed the real curve into the density estimate; don't guess.
- **Stability margin** — CP behind CG by 1–2 calibers; add nose mass if marginal. (Canted fins
  still give normal static stability; spin adds gyroscopic stability on top.)
- **Roll rate** — from the fin-cant field; treat as approximate and confirm from flight IMU/mag.

Then: OpenRocket gives speed + spin → density estimate → if too sparse, add mass/drag or cant →
freeze tube diameter → order airframe → start CAD.

---

## 12. Build sequence

1. Breadboard everything; confirm all sensors log to SD simultaneously with clean data. Don't
   solder yet.
2. Run the open tests (§13) on the breadboard.
3. Write + test the reconstruction pipeline on simulated data **in parallel**.
4. Build the bench spin-rig — the key de-risking milestone — and do the calibrations (§9).
5. OpenRocket → freeze geometry → CAD fins, sled, LiDAR mount → 3D print.
6. Solder the final harness; mount to sled; balance with counterweight.
7. Build airframe around the finished sled, window aligned to the LiDAR boresight.
8. Ground tests → test flights → iterate.

---

## 13. Test plan

- **Test 1 — Magnetometer rate + gyro range.** Read the MMC5983MA in a bare C loop on the Pico. If
  it sustains > 500 Hz you can spin near the cap; if it tops ~200 Hz, lower spin so you keep
  ≥ 30 samples/rev. **Also spin the IMU and confirm the roll-axis gyro rails at your spin rate
  while the transverse axes stay in range** at the sensitive (±500 °/s) setting.
- **Test 2 — IMU accel g-check.** Log peak accel under hard shake / representative boost. If the
  MPU-6050 never saturates (±16 g), use it and skip the ICM-20649 (–$15). If it clips, buy the
  ICM-20649 (±30 g). (This is the *only* thing that decides the IMU — the gyro saturates on roll
  either way.)
- **Test 3 — LiDAR outdoor range test.** The day it arrives: confirm valid returns off real
  terrain (grass, pavement, a building) at your slant ranges. Use the measured reliable range to
  **set the cant angle and the usable altitude band** (§3.2). If returns are poor, steepen the
  cant.
- **Test 4 — Bench spin-rig reconstruction.** Spin the full package at 10 rev/s and reconstruct a
  recognizable cloud of the room/yard after applying the §9 calibrations. Proves the whole pipeline
  without flying.
- **Test 5 — Recovery ground test.** Verify the ejection charge deploys the chute and the swivel
  spins freely.

---

## 14. Timeline (~17 days to July 7)

**Day 0 (today):** Secure a place to fly — either a sanctioned club launch on/before July 7 *or*
access to a suitable open field (~1,000 ft clear, with permission) plus your own pad/rail/controller
(§15). This, not the law, is what gates the flight. Order long-lead parts: used TF03 + TFmini-S;
DigiKey (ICM-20649 + BMP390 + SD); Sirius casing.

**Days 1–5 (parallel, no hardware needed):** Build the synthetic-flight simulator + full
reconstruction pipeline against a known scene. Breadboard sensors as they arrive; run Test 1
(incl. gyro check) and Test 2.

**Days 5–10:** Build the bench spin-rig; run Test 4 and the §9 calibrations; run Test 3 when the
TF03 arrives. **This milestone is a complete, demoable project.**

**Days 10–17 (stretch, only if a club launch exists in-window):** OpenRocket → re-derive apogee →
freeze geometry → CAD/print → solder harness → airframe → Test 5 → flight → reconstruct → write-up.

> Keep the bench-rig and software on the critical path — they de-risk the project and are the
> fallback demo if the flight slips.

---

## 15. Launch day, recovery, legal

- **Certification:** A–G motors need no certification for purchase or use; Level 1 is only for
  H/I. The E16W and F40W are far below the high-power thresholds (160 N·s total impulse / 62.5 g
  propellant per motor), so the project is cert-free.
- **You do NOT need a club (legally).** This rocket is an FAA *Class 1 model rocket* (≤ 1,500 g
  total and ≤ 125 g propellant — you're at ~600 g / ~30 g), which requires **no FAA notification or
  waiver regardless of altitude**. Altitude is not the trigger; you're in the no-paperwork regime
  whether you fly to 150 ft or 500 ft. A sanctioned club launch is **recommended, not required.**
- **What you actually need to fly (a club just bundles all of this):**
  - **A legal, open field.** NAR safety-code minimum site dimension for E/F motors is **~1,000 ft**
    (a roughly quarter-mile-wide clear area). A backyard won't do; a farm or large field with
    landowner permission will. Keep clear of people, buildings, and dry vegetation.
  - **Airspace + conditions:** away from airports/controlled airspace, not under clouds,
    wind < 20 mph, launch within 30° of vertical. Check a sectional/app for the site.
  - **Your own launch equipment:** a launch rail + a controller able to fire a composite igniter.
    The BOM has rail buttons but no pad/rail/controller — add these (~$40–80) if self-launching.
  - **State/local + fire rules** (you appear to be in Illinois): model rocketry is legal; check the
    State Fire Marshal and any municipal permit.
  - **No NAR insurance** when self-launching — clubs carry liability coverage; you don't.
- **Net effect on schedule:** with field access + a pad/controller, **self-launching removes the
  monthly-club-launch gate** and makes the flight schedulable inside the 17 days. The trade is that
  you take on the field, equipment, safety, and insurance yourself.
- **Age/purchase:** composite motor + igniter purchase can carry retailer/state age rules — check
  before ordering; involve an adult/mentor if anyone is under 18.
- **Recovery hardware:** threadlock the eyebolt; ball-bearing swivel on the shock cord; bias the
  ejection delay slightly short of apogee.

---

## 16. Risk register (highest first)

1. **No place to fly in the window** (parts lead time + no club launch, or no legal field, before
   July 7). → Self-launching from a suitable field (§15) removes the club-schedule gate; either way
   the bench rig + software stays the committed deliverable.
2. **LiDAR returns garbage** (low reflectivity, grazing beam). → Downward cant; Test 3 outdoors
   early; expect a dense-low/empty-high cloud.
3. **Roll reconstruction fails** (mag rate too low, bad calibration, missing boresight). → Test 1;
   assembled-bay + boresight calibration (§9); Test 4 proves it on the ground.
4. **Gyro saturation mishandled.** → Roll from mag only; gyro on a sensitive range for transverse
   tilt; spin ≤ 10 rev/s.
5. **Timing skew across streams smears the cloud.** → Single MCU, per-sensor `time_us_64()` stamps,
   interpolated `roll(t)`.
6. **Data loss on landing/brown-out.** → Periodic `f_sync`, pre-allocated file, status LED.
7. **Mechanical imbalance / fin flutter / window failure.** → Counterweight; solid PETG fin roots;
   open aperture.
8. **Recovery tangle from spin.** → Ball-bearing swivel; threadlock.
9. **Apogee/range mismatch** (flies higher than assumed → far ground out of LiDAR range). →
   Re-derive apogee in OpenRocket (§3.3); tune cant + mass.

---

## 17. Open decisions to close

- Final spin target = `min(mag-rate limit from Test 1, ~10 rev/s gyro-safe ceiling)` — likely cap
  at ~10 rev/s.
- IMU — decided by Test 2 accel g-check alone.
- Cant angle — decided by Test 3 real-terrain range vs the footprint trade-off.
- Real apogee/velocity — from the OpenRocket re-derivation → drives the density estimate and
  airframe order.
- Launch site with close-in features (≲ 60–80 m) **and** a way to fly it in the window — a club
  launch *or* a ~1,000 ft open field + your own pad/controller (§15).
- Budget sign-off for ~$425 (or pursue Hack Club hardware reimbursement / borrow a LiDAR — the
  highest-leverage cost move; TF03 is the swing cost, TFmini-S hedge ~$40).
