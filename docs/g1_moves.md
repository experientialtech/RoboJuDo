# g1-moves Deployment Guide

Deployment configurations for BeyondMimic motion policies on the Unitree G1 robot using the LocoMimic pipeline. Supports both MuJoCo sim2sim validation and real robot deployment.

## Configs

Two configs are defined in [`g1_loco_mimic_cfg.py`](../robojudo/config/g1/g1_loco_mimic_cfg.py):

| Config | Environment | Use case |
|--------|-------------|----------|
| `g1_moves` | `G1MujocoEnvCfg` | Sim2sim validation on workstation |
| `g1_moves_real` | `G1RealEnvCfg` (UnitreeCppEnv) | Real robot deployment |

Both use **AMOPolicy** for locomotion and **BeyondMimicPolicy** for motion playback, with smooth interpolation transitions between the two.

## Available Policies

ONNX models are stored in `assets/models/g1/beyondmimic/`:

| Policy | State Estimator | Max Steps | Description |
|--------|:-:|:-:|-------------|
| `B_Fence1` | Yes | - | Fencing lunge — stationary upper-body motion |
| `Dance_wose` | No | - | Dance motion |
| `Violin` | Yes | 500 | Violin playing — stationary upper-body, safe indoors |
| `Waltz` | Yes | 850 | Waltz dance |
| `Jump_wose` | - | - | Jump (test only, included with RoboJuDo) |

## Controls

### Sim2sim — Keyboard + Joystick

```
python scripts/run_pipeline.py -c g1_moves
```

**Keyboard:**

| Key | Action |
|-----|--------|
| `w/a/s/d` | Walk forward/left/backward/right |
| `q/e` | Turn left/right |
| `]` | Switch to locomotion mode |
| `p` | Switch to mimic mode (standing, paused) |
| `z` | Play motion |
| `x` | Pause motion |
| `c` | Reset motion to start |
| `;` | Previous mimic policy |
| `'` | Next mimic policy |
| `i` | Reset simulation |
| `o` | Shutdown |

**Xbox Joystick:**

| Button | Action |
|--------|--------|
| Left stick | Walk |
| Right stick | Turn |
| Select/Back | Locomotion mode |
| Start | Mimic mode |
| X | Play motion |
| B | Pause motion |
| Y | Reset motion |
| LB | Previous policy |
| RB | Next policy |
| A | Shutdown |

### Real Robot — Unitree Controller

```
python scripts/run_pipeline.py -c g1_moves_real
```

| Button | Action |
|--------|--------|
| Left stick | Walk |
| Right stick | Turn |
| Select | Locomotion mode |
| Start | Mimic mode |
| X | Play motion |
| B | Pause motion |
| Y | Reset motion |
| L1 | Previous policy |
| R1 | Next policy |
| A | Emergency shutdown (damping mode) |

## Pipeline Commands

Commands are triggered by controller inputs and handled by the pipeline and/or the active policy:

| Command | Handled by | Effect |
|---------|------------|--------|
| `[POLICY_LOCO]` | Pipeline | Interpolate to locomotion mode |
| `[POLICY_MIMIC]` | Pipeline | Interpolate to mimic mode (paused) |
| `[POLICY_SWITCH],NEXT` | Pipeline | Switch to next mimic policy |
| `[POLICY_SWITCH],LAST` | Pipeline | Switch to previous mimic policy |
| `[MOTION_FADE_IN]` | BeyondMimicPolicy | Start/resume motion playback |
| `[MOTION_FADE_OUT]` | BeyondMimicPolicy | Pause motion playback |
| `[MOTION_RESET]` | BeyondMimicPolicy | Reset motion to beginning |
| `[MOTION_DONE]` | Pipeline (callback) | Auto-switch to locomotion when motion ends |
| `[SHUTDOWN]` | Pipeline | Emergency stop |
| `[SIM_REBORN]` | Pipeline | Reset simulation (sim only) |

## Deployment Workflow

### Sim2sim (Local Workstation)

1. Install RoboJuDo and the `mujoco_viewer` submodule:
   ```bash
   cd RoboJuDo
   pip install -e .
   python submodule_install.py mujoco_viewer
   ```

2. Run with display:
   ```bash
   DISPLAY=:0 python scripts/run_pipeline.py -c g1_moves
   ```

3. Walk with WASD/QE, press `p` to enter mimic mode, then `z` to play.

### Real Robot

**Prerequisites:**
- Robot in **locked standing mode** (L2+A from Unitree app, feet on ground)
- Network connected (find interface with `ifconfig`, look for `192.168.123.x`)
- UnitreeCppEnv installed (`python submodule_install.py unitree_cpp`)

**Steps:**

1. Update `net_if` in the `g1_moves_real` config if your interface is not `enP8p1s0`:
   ```python
   env: G1RealEnvCfg = G1RealEnvCfg(
       unitree=G1UnitreeCfg(net_if="enP8p1s0"),
   )
   ```

2. Launch the pipeline:
   ```bash
   python scripts/run_pipeline.py -c g1_moves_real
   ```

3. **Prepare phase** (~20 seconds at 50Hz, 1000 steps): The robot smoothly blends from its current joint positions to the locomotion policy's init pose. Keep the robot on the ground during this phase.

4. After prepare completes, the robot enters **locomotion mode** (AMO policy). It should be actively balancing.

5. Use controller to operate:
   - **Walk**: Left joystick
   - **Enter mimic mode**: Press `Start`
   - **Play motion**: Press `X`
   - **Pause motion**: Press `B`
   - **Reset motion**: Press `Y`
   - **Back to walking**: Press `Select`
   - **Emergency stop**: Press `A`

## Modifying Policies

To change which mimic policies are loaded, edit the `mimic_policies` list in the config:

```python
mimic_policies: list[G1BeyondMimicPolicyCfg] = [
    G1BeyondMimicPolicyCfg(policy_name="B_Fence1", without_state_estimator=False),
    G1BeyondMimicPolicyCfg(policy_name="Violin", without_state_estimator=False, max_timestep=500),
]
```

Key parameters:
- `policy_name`: Must match an ONNX file in `assets/models/g1/beyondmimic/`
- `without_state_estimator`: Set `False` if the policy uses state estimation, `True` if not
- `max_timestep`: Maximum playback steps before `[MOTION_DONE]` fires (omit for unlimited)

## Code Changes from Upstream

The following modifications were made to the upstream RoboJuDo codebase:

### AMOPolicy keyboard support (`robojudo/policy/amo_policy.py`)

Added WASD+QE keyboard control to `_get_commands()`, matching the existing joystick control pattern. This allows walking the robot in sim2sim without a joystick:

- `w/s` — forward/backward velocity
- `a/d` — lateral velocity
- `q/e` — yaw rate

### Keyboard debug logging (`robojudo/controller/keyboard_ctrl.py`)

Added warning-level logging to `get_data()` to emit received keyboard events for debugging controller issues.

## Robot Setup Notes

### Network Interface

The G1's onboard network interface varies by hardware revision. Common values:
- `eth0` — default in most configs
- `enP8p1s0` — found on some G1 units

Run `ifconfig` on the robot to find the correct interface (look for `192.168.123.x`).

### Python Environment on Robot

The G1 onboard computer runs Ubuntu with Python 3.10. If cyclonedds is installed system-wide, create the venv with `--system-site-packages` to inherit it:

```bash
virtualenv --python=python3.10 --system-site-packages .venv
source .venv/bin/activate
pip install -e .
python submodule_install.py unitree_cpp
```

### Safety

- Always have the robot on a tether/harness for first deployment of new motions
- Start with stationary upper-body motions (Violin, B_Fence1) before full-body motions
- Test in sim2sim first before deploying to real hardware
- `do_safety_check: bool = True` is enabled for `g1_moves_real`
- Emergency stop: press **A** on the controller at any time
