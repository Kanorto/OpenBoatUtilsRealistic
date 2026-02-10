# OpenBoatUtilsRealistic — Changelog

All changes made on top of the original [OpenBoatUtils](https://github.com/o7Moon/OpenBoatUtils) (from commit `38e3097`, after CONTRIBUTING.md merge).

---

## Overview

OpenBoatUtilsRealistic is a fork of OpenBoatUtils that adds realistic four-wheel vehicle physics simulation to Minecraft boats. All new features are **server-driven** — the client does nothing unless the server sends the appropriate configuration packets. Vanilla behavior is preserved when no server packets are received.

---

## Version 1.2 (Current)

### Improved

#### Physics System Audit & Optimizations
A comprehensive audit and fix of the physics system, addressing performance, thread safety, correctness, and dead code:

- **Immutable surface presets**: All `SurfaceProperties` fields are now `final`, and all surface presets (`ASPHALT_DRY`, `ICE`, `BLUE_ICE`, etc.) are now `public static final`, preventing accidental mutation that could affect all players on a server. `BLUE_ICE` is now a proper preset (μ=0.06, even less grip than ICE).

- **Thread-safe block surface map**: `blockSurfaceMap` now uses `volatile` + double-checked locking for initialization, `ConcurrentHashMap` for thread-safe reads/writes, and `volatile` on `defaultSurface`. `resetBlockSurfaceMap()` is also `synchronized`.

- **GC optimization (SurfaceAccumulator)**: Introduced `SurfaceProperties.SurfaceAccumulator` — a reusable per-engine object that eliminates per-tick `new SurfaceProperties(...)` allocation in `detectSurface()`. When all sampled blocks resolve to the same preset instance (uniform surface), the accumulator returns the preset directly with zero allocation. Each physics engine instance holds its own accumulator.

- **Fixed yaw moment signs**: Corrected inverted signs in the longitudinal force yaw moment calculation in `FourWheelPhysicsEngine`. Left wheel forward force now correctly creates positive yaw (counterclockwise from above), and right wheel creates negative yaw. Changed from `(-fxWheel[0] + fxWheel[1])` to `(fxWheel[0] - fxWheel[1])` for both front and rear axles.

- **Removed dead code (RealisticPhysicsEngine)**: The old `RealisticPhysicsEngine` (bicycle model) was being kept in sync with `FourWheelPhysicsEngine` through ~20 setter methods in `OpenBoatUtils.java`, but was never actually used for physics computation (only `fourWheelPhysics` is referenced in `BoatMixin`). All redundant `realisticPhysics.*` calls have been removed. The `RealisticPhysicsEngine.java` class file is retained for potential future use.

### Changed Files
- `physics/SurfaceProperties.java` — `final` fields, `final` presets, `BLUE_ICE` preset, `SurfaceAccumulator`, thread-safe init
- `physics/FourWheelPhysicsEngine.java` — Uses `SurfaceAccumulator`, fixed yaw moment signs
- `physics/RealisticPhysicsEngine.java` — Uses `SurfaceAccumulator`, removed unused `Block` import
- `OpenBoatUtils.java` — Removed `realisticPhysics` field and all ~20 redundant setter calls

---

## Version 1.1

### Added

#### Realistic Physics Engine (`physics/` package)
A complete vehicle dynamics simulation with the following new source files:

- **`RealisticPhysicsEngine.java`** (598 lines) — Bicycle-model physics engine. Computes longitudinal/lateral tire forces, weight transfer, steering dynamics, surface detection, and airborne physics. Uses Fiala/Brush tire model for realistic slip angle behavior. Retained for potential future use but not actively used (see v1.2 dead code removal).

- **`FourWheelPhysicsEngine.java`** (653 lines) — Four-wheel (four-corner) physics engine that replaces the bicycle model. Each wheel (FL, FR, RL, RR) has independent:
  - Vertical load (weight transfer from acceleration/braking/cornering)
  - Slip angle and lateral force
  - Longitudinal force (drive/brake)
  - Friction circle constraint

  **Physics constants:**
  - `GRAVITY` = 9.81 m/s², `TICK_TIME` = 0.05 s (20 TPS)
  - `MIN_MU_PEAK` = 0.01 (minimum friction coefficient)
  - `YAW_RATE_DAMPING` = 0.995 (per-substep yaw damping)
  - `STOP_SPEED_THRESHOLD` = 0.15 m/s (below this speed → full stop)
  - `LOW_SPEED_FADE_THRESHOLD` = 0.5 m/s (forces fade in between 0.15–0.5 m/s)
  - `HANDBRAKE_FORCE_MULTIPLIER` = 0.5 (50% of braking force applied to rear wheels)
  - `SELF_ALIGN_BASE_RATE` = 3.0, `SELF_ALIGN_SPEED_THRESHOLD` = 5.0 m/s (steering return-to-center rate)
  - `LATERAL_VELOCITY_DAMPING` = 0.95 (per-tick lateral velocity damping)

  **Airborne physics constants:**
  - `AIR_DRAG_COEFFICIENT` = 0.35, `AIR_DENSITY` = 1.225 kg/m³, `FRONTAL_AREA` = 2.0 m²
  - `AIR_YAW_RATE_DAMPING` = 0.998 (slow spin-down in air)

  **Landing impact constants:**
  - `LANDING_IMPACT_THRESHOLD` = -1.5 m/s vertical velocity
  - `MAX_LANDING_GRIP_LOSS` = 0.6 (60% maximum grip reduction)
  - `LANDING_GRIP_RECOVERY_RATE` = 0.08 per tick
  - `MIN_AIRBORNE_TICKS_FOR_IMPACT` = 3 ticks minimum airborne time to trigger

  **Visual output constants:**
  - `VERTICAL_PITCH_FACTOR` = 0.15, `MAX_VERTICAL_PITCH` = 0.5 rad

  Supports differential types (OPEN/LOCKED/LSD), AWD torque split, aerodynamic downforce (`0.5 × coeff × ρ × vx²`), and weather-dependent grip/relaxation.

- **`TireModel.java`** (136 lines) — Fiala/Brush tire model implementation. Computes:
  - **Slip angles**: `α = atan2(vy_axle, vx) - steer` where `vx` is longitudinal velocity in vehicle frame, `vy_axle` is lateral velocity at the axle position (with `MIN_SPEED` = 1.0 m/s clamp on `vx`)
  - **Lateral forces**: Fiala cubic polynomial below slide angle (`αSlide = atan(3·μ·Fz/Cs)`), then `μ·Fz·sign(α)` with progressive falloff past peak slip angle controlled by `slipAngleFalloff`
  - **Longitudinal forces**: Clamped to `μ·Fz`, with smooth braking direction (`vx / max(|vx|, 0.5)`) to prevent oscillation near zero
  - **Friction circle**: `√(Fx² + Fy²) ≤ μ·Fz` — proportional scaling when exceeded
  - **Load sensitivity**: `μ_eff = μ_peak · (1 - loadSens · (Fz/Fz_nom - 1))`, clamped to min 0.01
  - **Force relaxation**: `blend = min(1, |speed| · dt / relaxLen)`, `force += (target - current) · blend`
  - Zero-allocation `FrictionCircleResult` reusable output object

- **`SurfaceProperties.java`** (522 lines) — Surface friction model with:
  - 18 surface presets: `ASPHALT_DRY`, `ASPHALT_WET`, `GRAVEL`, `DIRT`, `MUD`, `SNOW`, `ICE`, `BLUE_ICE`, `SAND`, `WOOD`, `CONCRETE`, `TERRACOTTA`, `METAL`, `GLASS`, `WOOL`, `BRICK`, `NETHER`, `VEGETATION`
  - Automatic block → surface mapping for 300+ Minecraft blocks (stone variants, colored concrete, terracotta, wood/logs, wool/carpets, metal/ore blocks, glass/panes, nether materials, vegetation, dirt/grass, mud/soul, snow, ice, sand, gravel, shulker boxes → WOOD, beds → WOOL)
  - Per-surface parameters: peak friction (`muPeak`), sliding friction (`muSlide`), cornering stiffness (`Cs`), relaxation length, rolling resistance, peak slip angle (degrees), slip angle falloff rate, load sensitivity
  - Server-configurable block surface type overrides via `setBlockSurfaceType(blockId, surfaceTypeName)`
  - Server-configurable default surface type via `setDefaultSurfaceType(surfaceTypeName)`
  - Default surface for unmapped blocks: `ASPHALT_DRY`

- **`VehicleConfig.java`** (62 lines) — Vehicle configuration with all tunable parameters and defaults:
  - `mass` = 1190 kg, `wheelbase` = 2.53 m, `cgHeight` = 0.45 m, `trackWidth` = 1.55 m
  - `frontWeightBias` = 0.55, `maxSteeringAngle` = 0.50 rad, `steeringSpeed` = 5.0
  - `brakingForce` = 8000 N, `engineForce` = 5500 N, `brakeBias` = 0.65, `engineBraking` = 800 N
  - `dragCoefficient` = 0.35, `rollingResistance` = 0.015
  - `substeps` = 4, `speedSteeringFactor` = 0.004
  - `rollStiffnessRatioFront` = 0.55, `drivetrain` = AWD
  - `awdFrontSplit` = 0.5, `frontDifferential` = OPEN, `rearDifferential` = OPEN
  - `lsdLockingCoeff` = 0.3, `downforceCoefficient` = 0.5, `downforceFrontBias` = 0.4

- **`VehicleType.java`** (60 lines) — 5 preset vehicle types with full configurations:
  - `WRC_CAR` — AWD, 1190 kg, wheelbase 2.53 m, CG 0.45 m, track 1.55 m, 55% front weight, steering 0.50 rad @ 5.0 speed, engine 5500 N, brake 8000 N @ 65% front, drag 0.35, RR 0.015
  - `GROUP_B` — RWD, 1100 kg, wheelbase 2.40 m, CG 0.50 m, track 1.50 m, 45% front weight, steering 0.48 rad @ 4.5 speed, engine 6000 N, brake 7500 N @ 60% front, drag 0.32, RR 0.014
  - `CLASSIC_RALLY` — RWD, 1000 kg, wheelbase 2.45 m, CG 0.55 m, track 1.45 m, 50% front weight, steering 0.45 rad @ 4.0 speed, engine 4000 N, brake 6000 N @ 65% front, drag 0.38, RR 0.018
  - `LIGHTWEIGHT` — FWD, 800 kg, wheelbase 2.30 m, CG 0.42 m, track 1.40 m, 60% front weight, steering 0.55 rad @ 6.0 speed, engine 3000 N, brake 5500 N @ 70% front, drag 0.30, RR 0.012
  - `TRUCK` — AWD, 2000 kg, wheelbase 3.20 m, CG 0.90 m, track 1.80 m, 50% front weight, steering 0.35 rad @ 3.0 speed, engine 8000 N, brake 10000 N @ 60% front, drag 0.45, RR 0.025

- **`DrivetrainType.java`** (29 lines) — Drivetrain enumeration:
  - `RWD` — Rear-wheel drive (only rear axle powered)
  - `FWD` — Front-wheel drive (only front axle powered)
  - `AWD` — All-wheel drive (configurable front/rear split via `awdFrontSplit`)

- **`DifferentialType.java`** (23 lines) — Differential types per axle:
  - `OPEN` — Torque biased by wheel load (more loaded wheel gets more torque)
  - `LOCKED` — Equal 50/50 torque split regardless of load
  - `LSD` — Blends between open and locked via `lsdLockingCoeff` (0.0 = open, 1.0 = locked)

- **`WeatherCondition.java`** (32 lines) — Weather grip and tire response modifiers:
  - `CLEAR` (id=0) — gripMultiplier=1.0, relaxationMultiplier=1.0
  - `RAIN` (id=1) — gripMultiplier=0.70, relaxationMultiplier=1.3 (30% slower tire response)
  - `HEAVY_RAIN` (id=2) — gripMultiplier=0.50, relaxationMultiplier=1.6 (60% slower tire response)
  - `SNOW` (id=3) — gripMultiplier=0.40, relaxationMultiplier=1.5 (50% slower tire response)
  - `FOG` (id=4) — gripMultiplier=0.95, relaxationMultiplier=1.1 (10% slower tire response)

- **`WheelPosition.java`** (25 lines) — Wheel position enumeration (FL, FR, RL, RR).

#### New Game Modes (`Modes.java`)
7 new modes added to the `Modes` enum:
- `REALISTIC` (25) — Default realistic mode with WRC car, rally settings, increased backward acceleration
- `REALISTIC_WRC` (26) — WRC rally car configuration
- `REALISTIC_GROUP_B` (27) — Group B rally car configuration
- `REALISTIC_CLASSIC` (28) — Classic rally car configuration
- `REALISTIC_LIGHTWEIGHT` (29) — Lightweight car configuration
- `REALISTIC_TRUCK` (30) — Truck configuration
- `REALISTIC_ALLTERRAIN` (31) — WRC car with step-while-falling enabled for terrain traversal

All realistic modes set: `allBlocksSlipperiness(0.98)`, `fallDamage(false)`, `airControl(true)`, `stepSize(1.25)`, plus the corresponding vehicle type. Additionally, `REALISTIC` (25) increases backward acceleration (`backwardsAcceleration(0.01)`), and `REALISTIC_ALLTERRAIN` (31) enables `stepWhileFalling(true)`.

#### New Network Packets (`ClientboundPackets.java`)
27 new server-to-client packets (IDs 33–59):
| ID | Packet | Data | Description |
|----|--------|------|-------------|
| 33 | `SET_REALISTIC_PHYSICS` | `boolean` | Enable/disable realistic physics engine |
| 34 | `SET_VEHICLE_TYPE` | `short` | Set vehicle preset (0-4, see VehicleType enum) |
| 35 | `SET_VEHICLE_MASS` | `float` | Vehicle mass in kg |
| 36 | `SET_VEHICLE_WHEELBASE` | `float` | Distance between axles in meters |
| 37 | `SET_VEHICLE_CG_HEIGHT` | `float` | Center of gravity height in meters |
| 38 | `SET_VEHICLE_TRACK_WIDTH` | `float` | Distance between left and right wheels |
| 39 | `SET_VEHICLE_MAX_STEERING` | `float` | Maximum steering angle in radians |
| 40 | `SET_VEHICLE_STEERING_SPEED` | `float` | How fast steering reacts |
| 41 | `SET_VEHICLE_BRAKING_FORCE` | `float` | Maximum braking force in Newtons |
| 42 | `SET_VEHICLE_ENGINE_FORCE` | `float` | Maximum engine force in Newtons |
| 43 | `SET_VEHICLE_DRAG` | `float` | Aerodynamic drag coefficient |
| 44 | `SET_VEHICLE_BRAKE_BIAS` | `float` | Front/rear brake distribution (0=rear, 1=front) |
| 45 | `SET_VEHICLE_SUBSTEPS` | `int` | Physics substeps per tick (1-10) |
| 46 | `SET_VEHICLE_FRONT_WEIGHT_BIAS` | `float` | Static weight distribution |
| 47 | `SET_BLOCK_SURFACE_TYPE` | `string, string` | Map block ID to surface type |
| 48 | `SET_VEHICLE_DRIVETRAIN` | `short` | Drivetrain type (0=RWD, 1=FWD, 2=AWD) |
| 49 | `SET_DEFAULT_SURFACE_TYPE` | `string` | Default surface for unmapped blocks |
| 50 | `SET_VEHICLE_SPEED_STEERING_FACTOR` | `float` | Speed-dependent steering reduction |
| 51 | `SET_VEHICLE_ENGINE_BRAKING` | `float` | Engine braking force |
| 52 | `SET_VEHICLE_ROLL_STIFFNESS_RATIO` | `float` | Front anti-roll bar stiffness ratio |
| 53 | `SET_AWD_FRONT_SPLIT` | `float` | AWD front/rear torque split |
| 54 | `SET_FRONT_DIFFERENTIAL` | `short` | Front axle differential type |
| 55 | `SET_REAR_DIFFERENTIAL` | `short` | Rear axle differential type |
| 56 | `SET_LSD_LOCKING_COEFF` | `float` | LSD locking coefficient |
| 57 | `SET_DOWNFORCE_COEFFICIENT` | `float` | Aerodynamic downforce |
| 58 | `SET_DOWNFORCE_FRONT_BIAS` | `float` | Downforce front/rear distribution |
| 59 | `SET_WEATHER_CONDITION` | `short` | Weather condition (0-4) |

#### Singleplayer Commands (`SingleplayerCommands.java`)
Extended from original 490 lines to 756 lines (+266 lines). All new commands send packets through the same server→client pipeline, ensuring identical behavior to multiplayer. New commands added:
- `/realisticphysics <true/false>` — Enable/disable realistic physics
- `/vehicletype <type>` — Set vehicle preset
- `/vehiclemass <mass>` — Set vehicle mass
- `/vehiclewheelbase <wheelbase>` — Set wheelbase
- `/vehiclecgheight <height>` — Set CG height
- `/vehiclemaxsteering <angle>` — Set max steering angle
- `/vehiclesteeringspeed <speed>` — Set steering responsiveness
- `/vehicleengineforce <force>` — Set engine power
- `/vehiclebrakingforce <force>` — Set braking power
- `/vehiclebrakebias <bias>` — Set brake bias
- `/vehiclesubsteps <steps>` — Set physics substeps
- `/vehicleweightbias <bias>` — Set front weight bias
- `/setsurfacetype <block> <surface>` — Map block to surface
- `/vehicledrivetrain <RWD/FWD/AWD>` — Set drivetrain
- `/defaultsurface <surface>` — Set default surface
- `/vehicletrackwidth <width>` — Set track width
- `/vehicledrag <drag>` — Set drag coefficient
- `/vehiclespeedsteeringfactor <factor>` — Set speed steering factor
- `/vehicleenginebraking <braking>` — Set engine braking
- `/vehiclerollstiffness <ratio>` — Set roll stiffness

Note: The following four-wheel model parameters have no singleplayer command and can only be set via server packets: AWD front split (53), front/rear differential (54, 55), LSD locking coefficient (56), downforce coefficient/bias (57, 58), weather condition (59).

#### CI/CD Workflows (`.github/workflows/`)
3 new workflow files:
- **`ci.yml`** — Runs `chiseledBuild` on every push/PR to `main` branch. Validates that the mod compiles for all supported Minecraft versions.
- **`prerelease-build.yml`** — Triggered on pre-release creation. Builds mod and uploads JAR artifacts to the GitHub release.
- **`release-build.yml`** — Triggered on full release creation. Builds mod, uploads artifacts, and attaches JARs to the release.

### Modified

#### `OpenBoatUtils.java`
- **VERSION**: Changed from `18` to `20` to reflect new packet additions
- **`sendVersionPacket()`**: Added `writeBoolean(true)` after version int to identify this mod as the Realistic variant to compatible server plugins
- **`resetSettings()`**: Now also resets `fourWheelPhysics` and `SurfaceProperties.resetBlockSurfaceMap()`
- **New static field**: `fourWheelPhysics` (FourWheelPhysicsEngine instance)
- **New methods** for setting all realistic physics parameters: `setRealisticPhysicsEnabled()`, `setVehicleType()`, `setVehicleConfig()`, `setVehicleMass()`, `setVehicleWheelbase()`, `setVehicleCgHeight()`, `setVehicleTrackWidth()`, `setVehicleMaxSteering()`, `setVehicleSteeringSpeed()`, `setVehicleBrakingForce()`, `setVehicleEngineForce()`, `setVehicleDragCoefficient()`, `setVehicleBrakeBias()`, `setVehicleSubsteps()`, `setVehicleFrontWeightBias()`, `setBlockSurfaceType()`, `setVehicleDrivetrain()`, `setDefaultSurfaceType()`, `setVehicleSpeedSteeringFactor()`, `setVehicleEngineBraking()`, `setVehicleRollStiffnessRatio()`, `resetRealisticPhysics()`, `setAwdFrontSplit()`, `setFrontDifferential()`, `setRearDifferential()`, `setLsdLockingCoeff()`, `setDownforceCoefficient()`, `setDownforceFrontBias()`, `setWeatherCondition()`
- **New imports**: physics package classes

#### `BoatMixin.java`
- **Merged with AbstractBoatMixin**: Now handles both `BoatEntity` (<=1.21) and `AbstractBoatEntity` (>=1.21.3) using Stonecutter conditional comments and the Stonecutter `/*$ boat >>*/` swap directive, eliminating the need for a separate `AbstractBoatMixin.java` in the 1.21.3 versions folder
- **`@Mixin` target**: Now conditionally targets `BoatEntity` (<=1.21) or `AbstractBoatEntity` (>=1.21.3) using Stonecutter
- **Shadow fields/methods**: All shadow declarations use Stonecutter conditions for the correct type per MC version
- **All hook methods**: `oncePerTick()`, `paddleHook()`, `tickHook()`, `hookCheckLocation()`, and all `@Redirect`/`@ModifyConstant` methods have version-conditional signatures for both `BoatEntity` and `AbstractBoatEntity`
- **`oncePerTick()`**: Added realistic physics engine integration:
  - Activation: when `fourWheelPhysics.isEnabled()` and boat is on ground (`ON_LAND`), or in air with `airControl` enabled (`IN_AIR`)
  - Sets `fourWheelPhysics.setAirborne(true)` when in air, `false` when on ground
  - **Input reading**:
    - Steering: `leftKey` → +1.0f, `rightKey` → -1.0f (or 0 if both/neither)
    - Throttle: `pressingForward` → 1.0f
    - Brake: `pressingBack` → 1.0f
    - Handbrake: `jumpKey.isPressed()` AND NOT airborne (ground-only)
  - **Physics update**: Calls `fourWheelPhysics.update(steeringInput, throttleInput, brakeInput, handbrake, yaw, surface)`
  - **Velocity application**: Sets entity velocity from physics engine output (`getVx()`, `getVy()`, projected to world XZ plane via yaw rotation)
  - **Yaw application**: Sets entity yaw from `getYawRate()` delta
  - **Visual effects**:
    - Pitch: `result.pitchAngle × -25.0f` (negative → nose dips on braking)
    - Roll contribution: `result.rollAngle × 8.0f` (directional lean in corners)
    - Combined: `MathHelper.clamp(visualPitch + rollContribution, -30.0f, 30.0f)` → `setPitch()`
  - **Jump suppression**: When realistic physics is active, jump key is repurposed as handbrake; original jump behavior is suppressed
- **`interpolationStepsHook()`**: Moved from the deleted `AbstractBoatMixin.java` into `BoatMixin.java` with `>=1.21.3` Stonecutter guard. Sets interpolation steps to 10 when `interpolationCompat` is enabled
- **Collision resolution**: Movement is subdivided into `collisionResolution` (1–50) sub-movements, each with `velocity / collisionResolution`, for higher wall collision accuracy
- **Water elevation**: When `waterElevation` is enabled and boat is `UNDER_WATER` or `UNDER_FLOWING_WATER`, Y position is incremented by 1.0 and Y velocity is zeroed
- **Coyote time**: Timer resets to `coyoteTime` when on ground (or water with `waterJumping`). Decrements each tick. Jump requires `coyoteTimer >= 0`. After jumping, timer is set to -1 (no double jump)
- **New import**: `FourWheelPhysicsEngine`

#### `ClientboundPackets.java`
- **27 new enum values** for realistic physics packets (SET_REALISTIC_PHYSICS through SET_WEATHER_CONDITION)
- **`handlePacket()`**: 27 new switch cases (33–59) handling all new packet types

#### `Modes.java`
- **7 new enum values**: `REALISTIC` (25) through `REALISTIC_ALLTERRAIN` (31)
- **New import**: `VehicleType`
- **`setMode()`**: 7 new switch cases configuring realistic physics with appropriate vehicle types

#### `ServerPlayNetworkHandlerMixin.java`
- **Added `remap = false`** to the `@Redirect` annotation on `preventMovedWronglyLog`. This is a correctness fix — `Logger.warn()` is not a Minecraft method and should not be remapped by the mixin processor.

#### `ServerboundPackets.java`
- **`handlePacket()`**: VERSION packet handler now reads the optional `boolean` realistic mod identifier if present in the buffer, using `buf.isReadable()` to maintain backward compatibility with standard OpenBoatUtils clients

#### `openboatutils.mixins.json5`
- **Removed Stonecutter conditionals** around `BoatMixin` — it is now included unconditionally for all versions since the version-specific logic was moved inside the mixin itself using Stonecutter comments
- **Removed `AbstractBoatMixin`** reference — no longer needed since `BoatMixin` handles all versions

### Removed

#### `versions/1.21.3/src/main/java/dev/o7moon/openboatutils/mixin/AbstractBoatMixin.java`
Deleted (263 lines). This was a separate mixin file for Minecraft 1.21.3+ that mixed into `AbstractBoatEntity` (Mojang renamed `BoatEntity` to `AbstractBoatEntity` in 1.21.3). Its functionality was merged into the main `BoatMixin.java` using Stonecutter conditional comments, keeping all version-specific logic in one file.

---

## Multiplayer Compatibility

### Server Plugin Compatibility
All new features follow the OpenBoatUtils protocol design:
- **No modifications without server packet**: The realistic physics engine is disabled by default. It only activates when the server sends packet 33 (`SET_REALISTIC_PHYSICS`) or packet 8 (`SET_MODE`) with a realistic mode.
- **`enabled` flag**: All setter methods set `OpenBoatUtils.enabled = true`. The mixin checks this flag before applying any modifications.
- **Reset on disconnect**: `resetSettings()` resets all physics state including the new engines, ensuring no state persists across server connections.

### Version Identification
- The version packet now includes an optional boolean to identify the Realistic variant
- Standard OpenBoatUtils servers will simply ignore the extra byte
- Compatible server plugins can read the boolean to detect the Realistic mod variant

### Protocol Extension
Packets 0–32 remain identical to the original OpenBoatUtils protocol. New packets 33–59 extend the protocol without breaking existing functionality.
