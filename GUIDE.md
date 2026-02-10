# OpenBoatUtilsRealistic — User Guide

A comprehensive guide to the realistic vehicle physics system in OpenBoatUtilsRealistic.

---

## Table of Contents
- [What is this?](#what-is-this)
- [Quick Start](#quick-start)
- [Controls](#controls)
- [Vehicle Types](#vehicle-types)
- [Surface Types](#surface-types)
- [Weather Conditions](#weather-conditions)
- [Drivetrain Types](#drivetrain-types)
- [Differential Types](#differential-types)
- [Physics Behavior](#physics-behavior)
- [Vehicle Tuning Guide](#vehicle-tuning-guide)
- [Singleplayer Commands](#singleplayer-commands)
- [For Server Administrators](#for-server-administrators)

---

## What is this?

OpenBoatUtilsRealistic adds a realistic four-wheel vehicle physics simulation to Minecraft boats. When enabled by a server (or in singleplayer via commands), your boat behaves like a real car with:

- **Realistic steering** — The boat turns based on tire slip angles, not just yaw rotation
- **Weight transfer** — Braking shifts weight forward, accelerating shifts it backward, cornering shifts it sideways
- **Surface-dependent grip** — Different blocks have different friction (ice is slippery, concrete has good grip, dirt is loose)
- **Drifting** — Use the handbrake to lock rear wheels and initiate controlled drifts
- **Multiple vehicle types** — From nimble WRC cars to heavy trucks

All features are **server-controlled**. The mod does nothing unless the server activates it via packets.

---

## Quick Start

### In Singleplayer
1. Install the mod
2. Open a singleplayer world
3. Place a boat and sit in it
4. Type `/realisticphysics true` to enable the physics engine
5. Type `/vehicletype WRC_CAR` to set the vehicle type
6. Drive!

### On a Server
The server administrator enables realistic physics via their server plugin. When you join, the physics settings are automatically applied. No configuration needed on your end.

---

## Controls

When realistic physics is active, boat controls change:

| Key | Action |
|-----|--------|
| **W** | Throttle (accelerate forward) |
| **S** | Brake |
| **A** | Steer left |
| **D** | Steer right |
| **Space** | Handbrake (locks rear wheels for drifting, ground only) |

### Key Differences from Normal Boats
- **No jumping**: Space is repurposed as handbrake. Boats cannot jump in realistic mode.
- **Steering is speed-dependent**: At high speed, steering becomes less responsive (like a real car). Controlled by `speedSteeringFactor` (default 0.004).
- **You can drift**: Hold Space while turning to initiate a controlled drift.
- **Braking shifts weight**: When you brake hard, the front tires get more grip and the rear tires get less.
- **Visual tilt**: The boat visually tilts forward when braking and backward when accelerating. Pitch is scaled at 25° per radian of acceleration angle, roll contributes 8° per radian of lateral acceleration. The combined visual angle is clamped to ±30°.
- **Handbrake is ground-only**: The handbrake does not engage while airborne, even with air control enabled.
- **Self-aligning steering**: When you release steering input, the wheels gradually return to center. The return rate increases with speed (base rate 3.0 × speed / 5.0 m/s).

---

## Vehicle Types

Five preset vehicle configurations are available:

### WRC Car (`WRC_CAR`)
Modern World Rally Championship car. The default and most balanced choice.
- **Mass**: 1190 kg | **Wheelbase**: 2.53 m | **CG Height**: 0.45 m
- **Track Width**: 1.55 m | **Front Weight Bias**: 55%
- **Max Steering**: 0.50 rad (~29°) | **Steering Speed**: 5.0
- **Engine**: 5500 N | **Braking**: 8000 N | **Brake Bias**: 65% front
- **Drag**: 0.35 | **Rolling Resistance**: 0.015
- **Drivetrain**: AWD
- **Character**: Balanced, good traction, predictable handling
- **Best for**: General rally driving, beginners

### Group B (`GROUP_B`)
Legendary Group B rally car. High power, rear-wheel drive.
- **Mass**: 1100 kg | **Wheelbase**: 2.40 m | **CG Height**: 0.50 m
- **Track Width**: 1.50 m | **Front Weight Bias**: 45%
- **Max Steering**: 0.48 rad (~28°) | **Steering Speed**: 4.5
- **Engine**: 6000 N | **Braking**: 7500 N | **Brake Bias**: 60% front
- **Drag**: 0.32 | **Rolling Resistance**: 0.014
- **Drivetrain**: RWD (Rear-Wheel Drive)
- **Character**: Oversteer-prone, spectacular drifts, requires skill
- **Best for**: Experienced drivers who want excitement

### Classic Rally (`CLASSIC_RALLY`)
Classic era rally car. Lower power, higher center of gravity.
- **Mass**: 1000 kg | **Wheelbase**: 2.45 m | **CG Height**: 0.55 m
- **Track Width**: 1.45 m | **Front Weight Bias**: 50%
- **Max Steering**: 0.45 rad (~26°) | **Steering Speed**: 4.0
- **Engine**: 4000 N | **Braking**: 6000 N | **Brake Bias**: 65% front
- **Drag**: 0.38 | **Rolling Resistance**: 0.018
- **Drivetrain**: RWD
- **Character**: Gentle power delivery, forgiving but slower
- **Best for**: Casual driving, scenic routes

### Lightweight (`LIGHTWEIGHT`)
Small, nimble front-wheel drive car.
- **Mass**: 800 kg | **Wheelbase**: 2.30 m | **CG Height**: 0.42 m
- **Track Width**: 1.40 m | **Front Weight Bias**: 60%
- **Max Steering**: 0.55 rad (~32°) | **Steering Speed**: 6.0
- **Engine**: 3000 N | **Braking**: 5500 N | **Brake Bias**: 70% front
- **Drag**: 0.30 | **Rolling Resistance**: 0.012
- **Drivetrain**: FWD (Front-Wheel Drive)
- **Character**: Understeer-prone, excellent traction on climbs, very responsive
- **Best for**: Tight tracks, technical courses

### Truck (`TRUCK`)
Heavy all-terrain truck.
- **Mass**: 2000 kg | **Wheelbase**: 3.20 m | **CG Height**: 0.90 m
- **Track Width**: 1.80 m | **Front Weight Bias**: 50%
- **Max Steering**: 0.35 rad (~20°) | **Steering Speed**: 3.0
- **Engine**: 8000 N | **Braking**: 10000 N | **Brake Bias**: 60% front
- **Drag**: 0.45 | **Rolling Resistance**: 0.025
- **Drivetrain**: AWD
- **Character**: Slow steering, massive braking force, stable at speed
- **Best for**: Off-road courses, obstacles

---

## Surface Types

Different Minecraft blocks give different grip and handling characteristics. Each surface has several physics parameters:
- **Peak Friction (μ peak)** — Maximum grip before tires start sliding
- **Sliding Friction (μ slide)** — Grip when tires are fully sliding (always lower than peak)
- **Cornering Stiffness (Cs)** — How quickly lateral force builds with slip angle (higher = sharper response)
- **Relaxation Length** — Tire response lag (higher = slower to reach target force)
- **Rolling Resistance** — Speed loss from tire deformation (higher = slower on straights)
- **Peak Slip Angle** — Slip angle at maximum grip before falloff begins
- **Load Sensitivity** — How much grip decreases under heavy load

### High Grip Surfaces
| Surface | Blocks | μ peak / μ slide | Cs | Rolling Res | Characteristics |
|---------|--------|------------------|------|-------------|-----------------|
| **Asphalt (Dry)** | Stone, Polished stone variants, Obsidian, Quartz, Ores, Sandstone | 0.85 / 0.70 | 65000 | 0.012 | Best grip, fast, predictable. Peak at 8° slip. |
| **Concrete** | All colored concrete blocks | 0.80 / 0.65 | 60000 | 0.013 | Very good grip. Peak at 9° slip. |
| **Brick** | Bricks, Nether bricks, Deepslate bricks, Mud bricks | 0.75 / 0.60 | 55000 | 0.016 | Good grip. Peak at 9° slip. |

### Medium Grip Surfaces
| Surface | Blocks | μ peak / μ slide | Cs | Rolling Res | Characteristics |
|---------|--------|------------------|------|-------------|-----------------|
| **Metal** | Iron, Gold, Copper, Diamond, Emerald blocks | 0.70 / 0.55 | 55000 | 0.010 | Medium-high grip, very low rolling resistance. Peak at 8° slip. |
| **Wool** | All wool and carpet colors, Moss, Beds | 0.70 / 0.60 | 28000 | 0.045 | High grip but very slow (high rolling resistance). Peak at 14° slip. |
| **Asphalt (Wet)** | Cobblestone, Mossy variants, Andesite, Diorite, Granite, Tuff, Prismarine | 0.55 / 0.40 | 50000 | 0.015 | Reduced grip, longer braking distances. Peak at 10° slip. |
| **Terracotta** | All terracotta and glazed terracotta | 0.65 / 0.55 | 45000 | 0.018 | Medium grip. Peak at 11° slip. |
| **Wood** | All planks, logs, stripped wood, Shulker boxes | 0.50 / 0.42 | 35000 | 0.020 | Medium grip, moderate rolling resistance. Peak at 12° slip. |

### Low Grip Surfaces
| Surface | Blocks | μ peak / μ slide | Cs | Rolling Res | Characteristics |
|---------|--------|------------------|------|-------------|-----------------|
| **Gravel** | Gravel | 0.55 / 0.50 | 30000 | 0.030 | Low grip, loose, high rolling resistance. Peak at 14° slip. |
| **Nether** | Netherrack, Basalt, Magma, Glowstone, Shroomlight | 0.50 / 0.40 | 35000 | 0.025 | Medium-low grip. Peak at 13° slip. |
| **Dirt** | Dirt, Grass, Clay, Farmland, Podzol, Mycelium, Packed mud | 0.45 / 0.40 | 25000 | 0.035 | Low grip, loose. Peak at 16° slip. |
| **Vegetation** | Hay, Dried kelp, Sponge, Slime block, Honey block | 0.40 / 0.35 | 22000 | 0.040 | Soft, very slow. Peak at 15° slip. |
| **Sand** | Sand, Red sand, Concrete powder | 0.40 / 0.35 | 18000 | 0.060 | Low grip, very high rolling resistance. Peak at 15° slip. |

### Very Low Grip Surfaces
| Surface | Blocks | μ peak / μ slide | Cs | Rolling Res | Characteristics |
|---------|--------|------------------|------|-------------|-----------------|
| **Glass** | All glass and glass panes, Sea lantern | 0.35 / 0.25 | 40000 | 0.008 | Very low grip, slippery. Peak at 7° slip. |
| **Snow** | Snow, Snow block, Powder snow | 0.30 / 0.22 | 22000 | 0.025 | Very low grip. Peak at 12° slip. |
| **Mud** | Mud, Soul sand, Soul soil, Mangrove roots | 0.30 / 0.25 | 16000 | 0.050 | Very low grip, extremely slow. Peak at 18° slip. |
| **Ice** | Ice, Packed ice, Frosted ice | 0.10 / 0.07 | 10000 | 0.008 | Minimal grip, extremely slippery. Peak at 6° slip. |
| **Blue Ice** | Blue ice | 0.06 / 0.04 | 7000 | 0.008 | Almost no grip, fastest sliding. Peak at 6° slip. |

### Unmapped Blocks
Any block not in the list above uses the **default surface** (Asphalt Dry by default). The server can change this.

---

## Weather Conditions

Weather affects grip globally. The server can change this dynamically. Weather also affects tire response speed — the relaxation multiplier increases the tire lag (higher = slower tire response).

| Condition | Grip Multiplier | Tire Response (Relaxation ×) | Effect |
|-----------|----------------|------------------------------|--------|
| **Clear** | 100% | Normal (×1.0) | Normal conditions |
| **Rain** | 70% | Slower (×1.3) | Reduced grip, delayed tire response |
| **Heavy Rain** | 50% | Much slower (×1.6) | Severely reduced grip, very sluggish tire response |
| **Snow** | 40% | Slower (×1.5) | Very low grip globally, slow tire response |
| **Fog** | 95% | Slightly slower (×1.1) | Minimal physics effect (mainly visual) |

---

## Drivetrain Types

The drivetrain determines which wheels receive engine power.

### RWD (Rear-Wheel Drive)
- Only rear wheels are powered
- Prone to **oversteer** (rear slides out in corners)
- Great for drifting
- Used by Group B and Classic Rally presets

### FWD (Front-Wheel Drive)
- Only front wheels are powered
- Prone to **understeer** (car goes straight when you turn)
- Good traction going uphill
- Used by Lightweight preset

### AWD (All-Wheel Drive)
- All four wheels are powered
- Configurable front/rear torque split (default 50/50)
- Best overall traction
- Used by WRC Car and Truck presets

---

## Differential Types

Differentials control how torque is distributed between the two wheels on the same axle.

### Open Differential
- Torque goes to the wheel with least resistance
- If one wheel loses grip, that wheel spins while the other gets no power
- Realistic but can cause loss of traction

### Locked Differential
- Both wheels rotate at the same speed
- Maximum traction (both wheels always get power)
- Can cause understeer in corners (fights turning)

### LSD (Limited Slip Differential)
- Partially locks when one wheel spins faster than the other
- Configurable locking coefficient (0.0 = fully open, 1.0 = fully locked)
- Default: 0.3 (moderate locking)
- Best compromise between traction and handling

---

## Physics Behavior

Detailed description of how the four-wheel physics engine works under the hood.

### Tire Model (Fiala/Brush)

Each wheel independently computes tire forces using the Fiala/Brush tire model:

- **Slip Angle**: The angle between where the tire is pointing and where it's actually going. Computed as: `α = atan2(vy_axle, vx) - steer`, where `vx` is the vehicle's longitudinal velocity (forward/backward) and `vy_axle` is the lateral velocity at the axle position including yaw rate contribution
- **Below saturation** (slip angle < slide angle): Lateral force follows a cubic polynomial — force builds progressively with slip angle
- **Above saturation** (tire sliding): Force equals μ × Fz with progressive falloff past peak slip angle. The falloff rate is controlled by `slipAngleFalloff`
- **Slide angle threshold**: `αSlide = atan(3 × μ × Fz / Cs)` — where Cs is the cornering stiffness

### Friction Circle

The combined lateral and longitudinal forces on each tire are constrained by the friction circle:
- **Constraint**: `√(Fx² + Fy²) ≤ μ × Fz`
- This means braking reduces available cornering grip, and cornering reduces available braking grip
- If total force exceeds the limit, both forces are scaled down proportionally

### Load Sensitivity

Tires produce less grip per unit of load as load increases:
- **Formula**: `μ_effective = μ_peak × (1 - loadSensitivity × (Fz/Fz_nominal - 1))`
- This means heavily loaded wheels (e.g. outer wheels in a corner) have slightly less grip coefficient than lightly loaded wheels
- Minimum effective μ is clamped to 0.01

### Force Relaxation

Tire forces don't change instantly — there's a lag controlled by relaxation length:
- **Formula**: `blend = min(1, |speed| × dt / relaxationLength)`
- `force = currentForce + (targetForce - currentForce) × blend`
- Shorter relaxation length = faster response. Weather conditions multiply the relaxation length (rain ×1.3, heavy rain ×1.6, etc.)

### Weight Transfer

Weight shifts between wheels during acceleration, braking, and cornering:

- **Longitudinal weight transfer**: Braking shifts weight to front wheels, accelerating shifts it to rear wheels. Proportional to: `mass × acceleration × cgHeight / wheelbase`
- **Lateral weight transfer**: Cornering shifts weight to outside wheels. Proportional to: `mass × lateralAcceleration × cgHeight / trackWidth`
- **Roll stiffness distribution**: The `rollStiffnessRatioFront` parameter (default 0.55) controls how lateral weight transfer is distributed between front and rear axles

### Differential Types

How engine torque is split between left and right wheels on each axle:

- **Open**: Torque biased toward the wheel with more load (proportional to vertical load). If one wheel has no load, it gets no torque.
- **Locked**: Equal 50/50 torque split regardless of load. Maximum traction but fights cornering.
- **LSD**: Blends between open and locked based on `lsdLockingCoeff` (0.0 = fully open, 1.0 = fully locked). Default 0.3.

### Aerodynamic Downforce

Speed-dependent downforce increases grip at high speed:
- **Formula**: `downforce = 0.5 × downforceCoefficient × airDensity(1.225) × vx²`
- Distributed between front and rear axles by `downforceFrontBias` (default 0.4 = 40% front, 60% rear)
- Only significant at high speeds (force grows with speed²)

### Airborne Physics

When the vehicle is in the air (with air control enabled):
- **Tire forces are disabled** — no traction, braking, or steering grip
- **Air drag applied**: `force = -0.5 × Cd(0.35) × A(2.0 m²) × ρ(1.225 kg/m³) × v²`
- **Yaw rate dampened**: Multiplied by 0.998 each tick (slow spin-down in air)
- **Lateral velocity dampened**: Multiplied by 0.95 each tick

### Landing Impact

Hard landings after being airborne cause temporary grip loss:
- **Trigger**: Vertical velocity exceeds 1.5 m/s downward AND vehicle was airborne for at least 3 ticks
- **Maximum grip loss**: 60% (gripPenalty = 0.6)
- **Recovery**: 8% per tick (penalty decreases by 0.08 each tick until zero)
- **Effect**: Available grip on all tires is multiplied by `(1 - landingGripPenalty)` during recovery

### Handbrake

The handbrake locks rear wheels for drifting:
- **Ground only**: Handbrake does not engage while airborne, even with air control enabled
- **Force**: Applies 50% of the configured braking force (`brakingForce × 0.5`) to rear wheels only
- **Front wheels remain free**: Steering still affects front wheels during handbrake use

### Stop Speed Threshold

At very low speeds (below 0.15 m/s), the vehicle is brought to a complete stop to prevent jittering. Between 0.15 and 0.5 m/s, forces are gradually faded in to prevent abrupt force application.

---

## Vehicle Tuning Guide

For advanced users who want to fine-tune their vehicle.

### Mass
- **Effect**: Heavier = more stable but slower to accelerate/brake
- **Range**: 100 – 5000 kg
- **Default**: 1190 kg (WRC Car)
- **Tip**: Lower mass makes the car more responsive but harder to control

### Wheelbase
- **Effect**: Longer = more stable at speed, slower to turn
- **Range**: 1.0 – 5.0 m
- **Default**: 2.53 m
- **Tip**: Shorter wheelbase = sharper turns, longer = highway stability

### CG Height (Center of Gravity)
- **Effect**: Higher = more weight transfer in corners (more body roll)
- **Range**: 0.1 – 2.0 m
- **Default**: 0.45 m
- **Tip**: Lower CG = less body roll, more stable. Higher CG = more dramatic weight transfer

### Track Width
- **Effect**: Wider = more lateral stability, less body roll
- **Range**: 1.0 – 3.0 m
- **Default**: 1.55 m

### Front Weight Bias
- **Effect**: Higher = more weight on front wheels (better front grip)
- **Range**: 0.3 – 0.7
- **Default**: 0.55 (55% front)
- **Tip**: Front-heavy = understeer tendency, rear-heavy = oversteer tendency

### Max Steering Angle
- **Effect**: Larger = tighter turns possible
- **Range**: 0.1 – 1.2 radians (~6° – 69°)
- **Default**: 0.50 rad (~29°)

### Steering Speed
- **Effect**: Faster = quicker steering response
- **Range**: 0.5 – 10.0
- **Default**: 5.0

### Speed Steering Factor
- **Effect**: Higher = more steering reduction at high speed
- **Range**: 0.0 – 0.1
- **Default**: 0.004
- **Tip**: Increase if the car feels twitchy at high speed

### Engine Force
- **Effect**: More force = faster acceleration
- **Range**: 1000 – 20000 N
- **Default**: 5500 N (WRC Car)

### Braking Force
- **Effect**: More force = shorter braking distance
- **Range**: 1000 – 20000 N
- **Default**: 8000 N

### Brake Bias
- **Effect**: Higher = more braking on front wheels
- **Range**: 0.0 – 1.0
- **Default**: 0.65 (65% front)
- **Tip**: Front-biased braking is more stable. Rear-biased braking helps initiate trail-braking drifts

### Engine Braking
- **Effect**: Simulates engine resistance when not accelerating
- **Range**: 0 – 5000 N
- **Default**: 800 N

### Drag Coefficient
- **Effect**: Higher = more air resistance at high speed
- **Range**: 0.0 – 2.0
- **Default**: 0.35

### Substeps
- **Effect**: More substeps = more accurate physics (but higher CPU cost)
- **Range**: 1 – 10
- **Default**: 4
- **Tip**: 4 is good for most cases. Increase to 6-8 only if you notice physics glitches

### Roll Stiffness Ratio
- **Effect**: How much anti-roll bar stiffness is on the front vs rear
- **Range**: 0.0 – 1.0
- **Default**: 0.55 (55% front)

### Downforce
- **Effect**: Increases grip at high speed (proportional to speed²)
- **Default Coefficient**: 0.5
- **Default Front Bias**: 0.4 (40% front, 60% rear)

### LSD Locking Coefficient
- **Effect**: How aggressively the LSD locks (only matters with LSD differential type)
- **Range**: 0.0 – 1.0
- **Default**: 0.3

---

## Singleplayer Commands

All commands are available in singleplayer for testing. They send packets through the standard server→client pipeline, so behavior is identical to multiplayer.

### Basic Commands
```
/reset                            — Reset all settings to vanilla
/realisticphysics <true/false>    — Enable/disable realistic physics
/vehicletype <type>               — Set vehicle preset (WRC_CAR, GROUP_B, CLASSIC_RALLY, LIGHTWEIGHT, TRUCK)
```

### Vehicle Parameters
```
/vehiclemass <100-5000>           — Set vehicle mass (kg)
/vehiclewheelbase <1-5>           — Set wheelbase (m)
/vehiclecgheight <0.1-2>         — Set center of gravity height (m)
/vehicletrackwidth <1-3>          — Set track width (m)
/vehicleweightbias <0.3-0.7>     — Set front weight bias
/vehiclemaxsteering <0.1-1.2>    — Set max steering angle (rad)
/vehiclesteeringspeed <0.5-10>   — Set steering responsiveness
/vehicleengineforce <1000-20000>  — Set engine power (N)
/vehiclebrakingforce <1000-20000> — Set braking power (N)
/vehiclebrakebias <0-1>          — Set brake bias (0=rear, 1=front)
/vehicleenginebraking <0-5000>   — Set engine braking (N)
/vehicledrag <0-2>               — Set drag coefficient
/vehiclesubsteps <1-10>          — Set physics substeps per tick
/vehiclespeedsteeringfactor <0-0.1> — Set speed-dependent steering reduction
/vehiclerollstiffness <0-1>      — Set front roll stiffness ratio
```

### Drivetrain & Differentials
```
/vehicledrivetrain <RWD/FWD/AWD> — Set drivetrain type
```

> **Note**: The following parameters can only be set by a server plugin (no singleplayer command available):
> AWD front/rear split, front/rear differential type, LSD locking coefficient, downforce coefficient, downforce front bias, weather condition.

### Surface Configuration
```
/setsurfacetype <block> <surface> — Map a block to a surface type
/defaultsurface <surface>         — Set default surface for unmapped blocks
```

Available surface names: `ASPHALT_DRY`, `ASPHALT_WET`, `GRAVEL`, `DIRT`, `MUD`, `SNOW`, `ICE`, `BLUE_ICE`, `SAND`, `WOOD`, `CONCRETE`, `TERRACOTTA`, `METAL`, `GLASS`, `WOOL`, `BRICK`, `NETHER`, `VEGETATION`

### Standard OpenBoatUtils Commands
All original OpenBoatUtils singleplayer commands also work:
```
/stepsize <size>                  — Set boat step height
/defaultslipperiness <value>      — Set default block slipperiness
/blockslipperiness <value> <blocks> — Set slipperiness for specific blocks
/removeblockslipperiness <blocks> — Remove slipperiness override for specific blocks
/clearslipperiness                — Clear all slipperiness overrides
/aircontrol <true/false>          — Enable/disable air control
/waterelevation <true/false>      — Enable/disable water elevation
/falldamage <true/false>          — Enable/disable fall damage
/jumpforce <force>                — Set jump force
/boatmode <mode>                  — Set boat mode (see available modes below)
/exclusiveboatmode <mode>         — Set boat mode exclusively (resets other settings first)
/modeseries <modes>               — Apply multiple modes in sequence (space-separated)
/exclusivemodeseries <modes>      — Reset then apply multiple modes in sequence
/boatgravity <gravity>            — Set gravity force
/setyawaccel <accel>              — Set yaw acceleration
/setforwardaccel <accel>          — Set forward acceleration
/setbackwardaccel <accel>         — Set backward acceleration
/setturnforwardaccel <accel>      — Set turn-forward acceleration
/allowaccelstacking <true/false>  — Allow acceleration stacking
/underwatercontrol <true/false>   — Enable underwater control
/surfacewatercontrol <true/false> — Enable surface water control
/coyotetime <ticks>               — Set coyote time (ticks of grace after leaving ground)
/waterjumping <true/false>        — Enable water jumping
/swimforce <force>                — Set swim force
/collisionmode <0-5>              — Set collision mode
/stepwhilefalling <true/false>    — Allow stepping while falling
/setinterpolationten <true/false> — Set interpolation compatibility (1.21.3+)
/setcollisionresolution <1-50>    — Set collision resolution substeps
/clearcollisionfilter             — Clear collision entity type filter
/addcollisionfilter <entitytypes> — Add entity types to collision filter
/setblocksetting <setting> <value> <blocks> — Set per-block physics settings
/sendversionpacket                — Re-send version packet to the server
```

#### Available Boat Modes
The `/boatmode` command accepts any of the following mode names:

**Standard modes**: `RALLY`, `RALLY_BLUE`, `BA_NOFD`, `PARKOUR`, `BA_BLUE_NOFD`, `PARKOUR_BLUE`, `BA`, `BA_BLUE`, `JUMP_BLOCKS`, `BOOSTER_BLOCKS`, `DEFAULT_ICE`, `DEFAULT_NINE_EIGHT_FIVE`, `DEFAULT_BLUE_ICE`, `NOCOL_BOATS_AND_PLAYERS`, `NOCOL_ALL_ENTITIES`, `BA_JANKLESS`, `BA_BLUE_JANKLESS`

**Realistic physics modes**: `REALISTIC`, `REALISTIC_WRC`, `REALISTIC_GROUP_B`, `REALISTIC_CLASSIC`, `REALISTIC_LIGHTWEIGHT`, `REALISTIC_TRUCK`, `REALISTIC_ALLTERRAIN`

> The realistic modes automatically enable the physics engine, set the corresponding vehicle type, and configure rally-style settings (low slipperiness, no fall damage, air control, 1.25 step height). The base `REALISTIC` mode additionally increases backward acceleration. `REALISTIC_ALLTERRAIN` also enables step-while-falling for terrain traversal.

---

## For Server Administrators

### Enabling Realistic Physics via Plugin

To enable realistic physics for a player, your server plugin needs to send the appropriate packets over the `openboatutils:settings` channel.

#### Quick Setup (Using Mode Packet)
Send packet ID 8 (SET_MODE) with mode ID 25 (REALISTIC) to instantly configure a player with default realistic settings:
```
short packetID = 8;    // SET_MODE
short modeID = 25;     // REALISTIC
```

#### Custom Setup (Individual Packets)
For fine-tuned control, send individual configuration packets:
```
1. Send packet 33 (SET_REALISTIC_PHYSICS) with boolean true
2. Send packet 34 (SET_VEHICLE_TYPE) with short 0 (WRC_CAR)
3. Optionally send packets 35-59 to customize individual parameters
```

#### Reset
Send packet 0 (RESET) to disable all modifications and return to vanilla behavior.

### Version Detection
The Realistic mod sends a version packet with an extra boolean flag set to `true`. Your server plugin can detect this by reading the boolean after the version int if data is available in the buffer.
