# stardance + outpost project

Spin-Stabilized Rocket LiDAR Terrain Mapper — Master Plan
Deadline: July 7, 2026 · Team: hardware + software split · Budget: ~$425


1. What we're building
A model rocket that maps the terrain around its launch site in 3D during ascent. Canted fins spin the rocket as it climbs. A single fixed LiDAR, pointing out the side and angled down, sweeps a helical pattern as the rocket rises and rotates. We log range + orientation + altitude at high rate, then reconstruct a 3D point cloud on the ground afterward. The spin does double duty: it stabilizes the rocket and it's what sweeps the one-axis sensor across the scene, so we avoid a gimbal or a multi-beam LiDAR.
2. Honest success criteria
Set expectations correctly so we're aiming at the right target:
Minimum success: a recognizable coarse point cloud — large buildings read as blobs, treelines as walls, the ground plane and big edges are clear. This is genuinely impressive and is the realistic ceiling for this hardware.
NOT the goal: survey-grade accuracy. That needs RTK GPS + industrial IMUs + calibrated scanners costing tens of thousands. Objects landing within ~1–3 m of true position is a win.
The bench demo is itself a success. If the flight fails, a spin-rig on the ground that reconstructs a room/yard proves the entire pipeline and is a complete project on its own.

3. How it works, end to end
Spin — canted fins induce ~10–15 rev/s roll during ascent.
Scan — the TF03 LiDAR, fixed in the airframe pointing out a side window ~30° below horizontal, measures distance to whatever the beam hits.
Know the angle — a magnetometer recovers the roll angle (which way the beam points) from the sinusoid traced by Earth's magnetic field as the rocket spins. An IMU gives pitch/tilt. A barometer gives altitude.
Log — the Pico timestamps and records all four streams to SD at high rate. No processing in flight.
Recover — parachute at apogee; a ball-bearing swivel keeps residual spin from tangling the lines.
Reconstruct — on the ground, Python turns each (time, roll, tilt, altitude, range) record into a 3D (x, y, z) point. Stack them → point cloud → visualize in Open3D.
4. Locked parts list
Already owned ($0)
Raspberry Pi Pico (flight computer), Pi Zero 2 W (ground reconstruction), DJI Action 2 (optional flight video), soldering kit, Elegoo kit (prototyping + possible MPU-6050).
Electronics to buy
Part
Source
Cost
Notes
TF03-100 LiDAR — UART version, used
eBay (returnable)
~$170
Order first; scarce. Confirm UART, not CAN/RS485.
MMC5983MA magnetometer (Qwiic Micro, I2C)
SparkFun
$18
Bench its real I2C rate before locking spin speed.
ICM-20649 IMU (or free MPU-6050 if it passes g-check)
DigiKey
$15 / $0
See Test 2.
BMP390 barometer
DigiKey
$11
Bundle with DigiKey order.
MicroSD SPI breakout + 16GB Class-10 card
DigiKey + Amazon
$10
Card: Samsung Endurance / SanDisk.
1S LiPo (500–800 mAh) + Pololu U1V11F5 5V boost
Amazon / Pololu
$16
Boost feeds TF03 5V rail.

Rocket to buy
Part
Source
Cost
Notes
Aerotech RMS 29/40-120 casing + closures
Sirius Rocketry
~$74
Verified ~$73.94.
2× E16W or F40W reloads
Apogee
$40
No certification needed (see §12).
Body tube, nose cone, centering rings, motor mount, rail buttons
Apogee
~$25
Diameter set by OpenRocket — order after sim.
Parachute, Kevlar shock cord, ball-bearing swivel
Apogee
~$12
Kevlar (not elastic); swivel is mandatory.
Filament (PETG), epoxy, wire, threadlocker, heat shrink
Hardware/Amazon
~$35
Silicone wire only. Threadlock every fastener.


Total ~$425 (~$410 if the MPU-6050 works).

Order sequence: (1) TF03 today. (2) One DigiKey order: ICM-20649 + BMP390 + SD breakout. (3) Sirius casing. (4) Everything else after OpenRocket fixes the tube diameter.
5. Physical layout (nose → tail)
Nose cone with bulkhead + threadlocked eyebolt.
Recovery bay: parachute + Kevlar shock cord, ball-bearing swivel between cord and nose eyebolt.
Electronics bay: 3D-printed PETG sled holding Pico, magnetometer, IMU, baro, SD breakout, LiPo, boost converter. The TF03 mounts here, aimed out a side window.
Motor mount: 29mm inner tube, centering rings, motor retainer. Fin roots ideally land over centering-ring positions for strength.

Two placement rules that matter:

Magnetometer far from LiPo, steel, and the motor casing. It reads Earth's field; nearby iron/battery creates a hard-iron offset that corrupts roll recovery. Sensor at one end of the sled, ferrous parts at the other.
TF03 (~77 g) balanced by a ~70 g counterweight opposite the window. Otherwise spin is unbalanced and the CG drifts off the spin axis.
The LiDAR window
Cut a radial hole; TF03 sits inside aiming out ~30° below horizontal (ranges the ground, not the horizon). Leave the aperture open — most plastics don't pass 905 nm cleanly and will ruin returns. The TF03's IP67 housing handles the airflow.

6. Wiring & Pico pin map
Raspberry Pi Pico

├── UART0  (GP0 TX, GP1 RX) ────────── TF03-100 LiDAR  (powered from 5V boost)

├── SPI0   (GP2 SCK, GP3 MOSI, GP4 MISO, GP5 CS) ── MicroSD card

├── I2C0   (GP8 SDA, GP9 SCL) ───────── MMC5983MA magnetometer  (own bus, high-rate)

├── I2C1   (GP14 SDA, GP15 SCL) ─────── ICM-20649 IMU + BMP390 baro  (shared, diff addresses)

├── 3V3 OUT ── powers mag, IMU, baro, SD logic

├── VSYS  ──── LiPo 3.7V input

└── 5V boost converter ── TF03 5V rail (≤150 mA)

Put the magnetometer on its own I2C bus so its fast polling doesn't compete with the IMU/baro. Keep all SPI/I2C wires short. Add a ~5 kΩ pull-up on SD MISO if reads are flaky.

7. Flight firmware (Pico, C/C++)
Use the Pico C SDK, not MicroPython — MicroPython can't reliably sustain multi-sensor logging at high rate.

Core 0: tight loop reading LiDAR (UART), magnetometer (I2C0), IMU + baro (I2C1). Stamp each sample with time_us_64(). Push records into a RAM ring buffer.
Core 1: drains the buffer to SD in binary (convert to CSV on the ground). Use the carlk3/no-OS-FatFS-SD-SPI-RPi-Pico library.
Record (~20 bytes): timestamp_us, lidar_cm, mag_x, mag_y, mag_z, gyro_x/z, accel_z, baro_pressure.
Throughput: ~1000 rec/s × 20 B = ~20 KB/s. A 2–3 s flight = tiny. SD speed is not a bottleneck.
Arming: a switch or detect liftoff via accel threshold; flush + close file a few seconds after apogee so nothing is lost.

8. Reconstruction pipeline (Pi Zero 2 W or laptop, Python)
Parse binary → arrays (NumPy).
Roll from magnetometer: the transverse field traces one sine cycle per revolution; phase = absolute roll angle (drift-free). Use SciPy for phase extraction. Local note: in Illinois the field dips steeply (~68° inclination), so the transverse component is ~37% of total — usable, but calibrate hard/soft-iron carefully.
Tilt from IMU accel/gyro (complementary or Madgwick filter) over the short ascent.
Altitude from baro (z-axis), the most reliable position channel.
Per sample: beam vector from (roll = azimuth, fixed cant = elevation); point = sensor_origin(0,0,alt) + range × beam_vector → (x, y, z).
Assemble into an (N,3) array → Open3D for downsampling, normals, visualization. Matplotlib for debug plots (altitude/spin/raw mag vs time).
Optional georeferencing: place known ground targets (bright tarps) and align the cloud to them in post

Libraries: NumPy, SciPy, Open3D, Matplotlib. Build and test this against simulated data before the first flight so it's ready the moment real data lands.

9. OpenRocket — what to do
Free Java app. Build: nose cone → body tube → inner motor mount with centering rings → trapezoidal canted fins → a mass component for the electronics bay placed where it physically sits.

You're optimizing for the opposite of altitude. You want a slow climb through the scan band so the spinning LiDAR gets many revolutions per foot — but it must still leave the rail fast enough to fly straight.

Starting numbers to iterate from (validate in the sim, don't treat as final):

Body tube: ~2.6" (66 mm) — roomy enough for TF03 + counterweight + sled, and the extra drag helps slow the climb.
Liftoff mass: ~550–650 g all-up (electronics bay ~230 g incl. counterweight; airframe + motor the rest).
Reload: start with F40W — for a draggy ~600 g rocket it gives safe rail exit (thrust-to-weight ~6) and a moderate burn. Try E16W only if mass comes out low and rail-exit speed still passes; it's gentler (more spin time) but marginal for getting off the rail on a heavy build.
Fin cant: start ~5°, tune toward 10–15 rev/s.

Read off and tune these four:

Apogee → aim ~130–150 ft, just above the 120 ft band. You then decelerate near the top, so the upper scan is denser than the bottom (expect a naturally sparse-low, dense-high cloud).
Velocity vs altitude → feed the real ascent speed through the band into the density simulation. Don't guess.
Stability margin → keep CP behind CG by 1–2 calibers; add nose mass if marginal. (Canted fins still provide normal static stability; spin adds gyroscopic stability on top.)
Roll rate → from the fin-cant field; treat as approximate, confirm real spin from flight IMU/mag data.

Aerotech RMS reloads are in OpenRocket's motor database — swap and compare directly.

Then: OpenRocket gives speed + spin → plug into the density sim → if too sparse, add mass/drag or cant → freeze tube diameter → order airframe → start CAD.

10. Build sequence
Breadboard everything. Confirm all four sensors log to SD simultaneously with clean data. Don't solder yet.
Run the two open tests (below) while still on the breadboard.
Write + test the reconstruction pipeline on simulated data in parallel.
Build the bench spin-rig (lazy-Susan/motor) — this is the key de-risking milestone.
OpenRocket → freeze geometry → CAD the fins, sled, LiDAR mount → 3D print.
Solder the final harness; mount to sled; balance with counterweight.
Build airframe around the finished sled, window aligned to TF03.
Ground tests → test flights → iterate.

11. Test plan
Test 1 — Magnetometer I2C rate. Read the MMC5983MA in a bare C loop on the Pico (not Arduino lib, not MicroPython). If you sustain >500 Hz, spin at 15 rev/s (≥30 samples/rev). If it tops ~200 Hz, cap spin at 10 rev/s. This decides motor choice and fin cant.
Test 2 — MPU-6050 g-check. Wire the Elegoo MPU-6050, log peak accel under hard shake/representative boost. If it never saturates (±16 g), use it and skip buying the ICM-20649 (–$15). If it clips, buy the ICM-20649 (±30 g).
Test 3 — TF03 outdoor range test. The day it arrives: confirm valid returns off real terrain (grass, pavement, a building) at your slant ranges. If <~30% valid returns, increase downward cant.
Test 4 — Bench spin-rig reconstruction. Spin the full package at 10–15 rev/s on the ground and reconstruct a recognizable cloud of the room/yard. This proves the whole pipeline without flying.
Test 5 — Recovery ground test. Verify ejection charge deploys the chute and the swivel spins freely.

12. Launch day, recovery, legal
Certification: A–G motors need no certification for purchase or use. Level 1 is only for H/I. Your E16W (16 N) and F40W (40 N) are far below the high-power thresholds (80 N avg thrust, 125 g propellant), so you're fully cert-free.
Where to fly: a NAR/Tripoli-sanctioned club launch is strongly recommended — established field, safety officer, launch equipment, and a waiver if needed. Find a local club; many welcome students.
Age/purchase: composite motor and igniter purchase can carry retailer/state age rules — check before ordering, and have an adult/mentor involved if anyone is under 18.
FAA: staying under 130–150 ft is far below the 400 ft AGL model-rocket ceiling. No FAA notification needed at this scale, but don't fly near airports/controlled airspace — check a sectional or app for your site.
Recovery hardware: threadlock the eyebolt; ball-bearing swivel on the shock cord; bias ejection delay slightly short of apogee.

13. Timeline to July 7
Week 1 (now): order TF03 today; place DigiKey + Sirius orders; install OpenRocket + start the model; breadboard sensors; start firmware and the reconstruction pipeline on simulated data; run Tests 1 & 2.
Week 2: finalize OpenRocket → freeze geometry → CAD + print parts; solder final harness; build the bench spin-rig; Test 4 (reconstruct from spin-rig); Test 3 when TF03 arrives.
Week 3: build airframe; integrate; ground tests; first test flight; iterate; reconstruct flight data; prep the demo + writeup.

Keep the bench-rig and software on the critical path — they de-risk the project and are the fallback demo if flights slip.

14. Risk register (highest first)
LiDAR returns garbage (low reflectivity, beam too shallow). → Downward cant; Test 3 outdoors early.
Roll reconstruction fails (mag rate too low, bad calibration, weak transverse field). → Test 1; careful hard/soft-iron calibration; Test 4 proves it on the ground.
Timing skew across streams smears the cloud. → Single MCU timestamps everything.
Mechanical imbalance / fin flutter / window failure. → Counterweight; solid PETG fin roots; open aperture.
Recovery tangle from spin. → Ball-bearing swivel; threadlock.
Overshoot (too high/fast → thin cloud). → Add mass/drag in OpenRocket; aim 130–150 ft.
Parts delays (TF03 + OOS Adafruit items). → Order immediately; buy returnable.

15. Open items to close
Confirm hackathon budget covers ~$425 (or pursue Hack Club hardware reimbursement / borrow a LiDAR — the highest-leverage cost move).
Test 1 result → final spin target (10 vs 15 rev/s).
Test 2 result → ICM-20649 or free MPU-6050.
OpenRocket → final tube diameter, mass, reload, fin cant → then order airframe.
Identify a sanctioned launch site / club.
Confirm purchase age rules with the motor retailer.

