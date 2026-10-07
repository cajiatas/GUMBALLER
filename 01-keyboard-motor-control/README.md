# How I got keyboard control working

I set this up so I could drive the robot's wheels with a keyboard while working on the rest of GUMBALLER. The Raspberry Pi runs ROS 2 and handles the commands. The Pico handles the motor outputs and reads the wheel encoders. This setup controls the drive wheels; the arm and MoveIt are separate work.

## How the pieces connect

```mermaid
flowchart LR
    K[Keyboard in a Pi or SSH terminal] --> T[ROS keyboard node]
    T -->|/cmd_vel: TwistStamped| B[pico_bridge.py]
    B -->|USB serial| P[Pico 2 wheel program]
    P --> D[Motor drivers]
    D --> W[Left and right wheels]
    W -->|Encoder feedback| P
```

The keyboard node and bridge run together inside the `avc_manual` Docker container on the Pi. When I use SSH, my computer sends the keypresses to that terminal; ROS still runs on the Pi.

## What I did to get here

### 1. Got ROS running on the Pi

My Pi runs Debian 13. I installed Docker so I could use the Ubuntu environment for ROS 2 Jazzy without replacing the Pi's operating system. An image is the saved software setup, and a container is a running instance of it.

I created `~/avc_ws` to keep the project files outside the container and checked that a file saved there survived after a container was removed. I also tested a ROS publisher and subscriber before connecting any motors. A publisher sends messages, a topic carries them, and a subscriber receives them.

I used `Dockerfile.teleop` to build `avc-ros:jazzy` from `ros:jazzy-ros-base-noble`. It installs `ros-jazzy-teleop-twist-keyboard` and `python3-serial`. The first package reads the keyboard; the second lets the bridge communicate over USB. I used the existing [ROS keyboard package](https://github.com/ros2/teleop_twist_keyboard) for the key mappings and speed parameters.

This is the image build command I used. I only need to rebuild when changing the image or recreating it:

```bash
sudo docker build -t avc-ros:jazzy -f ~/avc_ws/Dockerfile.teleop ~/avc_ws
```

### 2. Checked the keyboard messages first

Before connecting ROS to the Pico, I watched `/cmd_vel` in another terminal and checked that `i` produced forward speed and `k` produced zero speed. That first test used ordinary `Twist` messages. The finished bridge uses `TwistStamped`, which also includes the time the command was created.

That timestamp matters because I do not want an old command waiting in a queue to restart the robot later.

### 3. Backed up the existing Pico code

I connected the Pico to the Pi with a USB data cable and found its serial device with:

```bash
ls -l /dev/serial/by-id/
lsusb
```

I used `mpremote` to inspect and copy its files into `~/avc_ws/pico-backup.lj15vN`. The existing code came from the previous team. It already had motor drivers, encoder reading, wheel-speed control, and differential-drive calculations. Its startup program also used the old arm and IMU.

I kept that backup and made a separate `pico_manual` working copy for the wheels. The drive wiring was unchanged, so I kept its GPIO assignments:

| Wheel | PWM | Direction A/B | Encoder A/B |
|---|---:|---|---|
| Left | GP21 | GP19 / GP20 | GP6 / GP7 |
| Right | GP16 | GP18 / GP17 | GP26 / GP27 |

These are GPIO numbers, not physical header-pin numbers.

### 4. Adapted the wheel code for manual control

I used the existing controller structure and updated the startup, timeout, and stopping behavior. Motor output starts at zero, stopping clears the stored output, and encoder variables are initialized before their interrupts are enabled. Wheel speed is calculated using the actual elapsed time between measurements.

The controller compares requested wheel speed with measured speed and adjusts PWM output 50 times per second. Encoder-based speed measurement runs at 100 Hz. I kept the inherited settings of 64 encoder counts per motor revolution, a 102.083 gear ratio, 0.080 m wheel radius, and 0.51 m wheel separation. Those are the values in the code, not a new physical calibration.

The Pico receiver accepts newline-terminated commands:

| Command | Meaning |
|---|---|
| `PING` | Stop and answer `PONG,AVC_WHEELS_V1` |
| `STOP` | Stop and answer `STOPPED` |
| `V,0.05,0` | Request 0.05 m/s forward with no turn |

### 5. Checked it in stages

I checked the six wheel-program files for syntax, uploaded them to `/pico_manual` on the Pico, and ran the compatibility check. Then I checked that startup, zero-speed commands, and shutdown left motor outputs at zero.

With motor power disconnected, I turned each wheel by hand for the encoder check. With both wheels still, the counters read `0 / 0`. Turning the left wheel forward gave `6996 / 0`; turning the right gave `0 / 9008`. Each wheel affected its own counter, and forward rotation gave positive counts. I used this to check the encoder connections and direction, not to calibrate distance.

I backed up the original startup file in `pico-startup-backup.S53FAm` and uploaded `pico_boot_manual.py` as the Pico's `/main.py`. That launcher loads `/pico_manual/main.py` at startup. I then checked `PING` and `STOP` replies before connecting the ROS bridge.

Finally, I tested the full keyboard-to-Pico path with motor power off, verified stop acknowledgements, and moved on to brief powered wheel checks. Keyboard driving then worked in use.

## How I start it again

I sign in as `cajiatas` using the [SSH guide](../02-ssh-into-the-pi/README.md), connect the Pico to the Pi, and open two Pi terminals. All commands below run **on the Pi**. I keep wheel-motor and servo power off while starting the software.

In the first terminal:

```bash
sudo docker run --rm -it --init \
  --name avc_manual \
  --device /dev/serial/by-id/usb-MicroPython_Board_in_FS_mode_c8a533cc0f918c96-if00:/dev/pico \
  -v "$HOME/avc_ws:/ws:ro" \
  -e ROS_AUTOMATIC_DISCOVERY_RANGE=LOCALHOST \
  avc-ros:jazzy python3 /ws/pico_bridge.py
```

I wait for `Pico connected. Waiting for fresh keyboard commands.` and leave it running. The mount exposes my Pi workspace as `/ws` with read-only access, and the device mapping exposes this Pico as `/dev/pico` inside the container.

In the second terminal:

```bash
sudo docker exec -it avc_manual bash -c 'source /opt/ros/jazzy/setup.bash && ros2 run teleop_twist_keyboard teleop_twist_keyboard --ros-args -p stamped:=true -p speed:=0.05 -p turn:=0.15'
```

I select that terminal and press `k`, without Enter. The bridge should print:

```text
STOP: keyboard requested zero speed
Pico acknowledged STOP.
```

For the initial powered check, I securely raise both drive wheels, keep servo power off, enable wheel-motor power, and tap `i`. Both wheels should briefly turn forward and stop. I keep the motor-power cutoff within reach.

| Key | Action |
|---|---|
| `i` | Forward |
| `,` | Backward |
| `j` / `l` | Turn left / right in place |
| `u` / `o` | Forward while curving left / right |
| `k` | Request a stop |

I use lowercase keys with Caps Lock off and Shift released. Holding a movement key supplies repeated keypresses; releasing it relies on the timeout. I press `k` when I want to stop.

To finish, I press `k`, turn motor power off, then press Ctrl+C in the keyboard terminal and Ctrl+C in the bridge terminal.

## Limits and things I learned

The starting commands request 0.05 m/s and 0.15 rad/s. The bridge rejects commands above 0.10 m/s forward/backward or 0.30 rad/s turning. Each wheel has a 0.25 m/s command limit, and the Pico caps PWM duty at 30%. These are initial test limits, not measured maximum performance.

The bridge requests a stop when a command reaches about 0.6 seconds old. It sends valid USB movement commands every 0.05 seconds while that command is fresh. The Pico also stops commanding movement after 0.5 seconds without a valid USB velocity command. These timers are separate; they are not a promise that the physical robot stops instantly. `STOPPED` confirms the command was processed, not that an encoder measured zero motion.

Two problems I ran into were simple:

- **Caps Lock:** uppercase turning keys request sideways motion, which this two-wheel bridge rejects. Keeping Caps Lock off fixed it.
- **The left wheel barely moving during `u`:** this was expected. At the starting settings, the left wheel is requested at about 0.012 m/s and the right at 0.088 m/s. `j` is the key for turning left in place.

If Docker cannot find the Pico, I check the USB data cable and rerun the device-list commands. If the port is busy, I close `mpremote` or the other serial program first. If the bridge reports a fault, invalid command, or missing acknowledgement, I switch motor power off and inspect that error before restarting. I leave the keyboard speed-adjustment keys alone during these tests.

## Where I kept the files

| File or folder in `~/avc_ws` | Purpose |
|---|---|
| `Dockerfile.teleop` | Builds the ROS image |
| `pico_bridge.py` | Validates ROS commands and sends USB commands |
| `pico_manual/main.py` | Receives USB commands on the Pico |
| `pico_manual/mobile_base/motor_driver.py` | PWM output and direction pins |
| `pico_manual/mobile_base/sensored_motor_driver.py` | Encoder interrupts and counts |
| `pico_manual/mobile_base/wheel_driver.py` | Converts counts into measured wheel speed |
| `pico_manual/mobile_base/wheel_controller.py` | Adjusts motor output toward requested speed |
| `pico_manual/mobile_base/diff_drive_controller.py` | Converts forward/turn requests into left/right speeds |
| `pico_boot_manual.py` | Launcher uploaded as the Pico's `/main.py` |
| `pico_import_check.py` | Checks compilation and module paths on the Pico |
| `pico_zero_check.py` | Checks zero outputs and timer shutdown |
| `pico_encoder_check.py` | Checks encoder counts by hand |
| `pico-backup.lj15vN/` | Previous team's original Pico files |
| `pico-startup-backup.S53FAm/` | Original startup-file backup |

Editing the Pi's files does not automatically update the Pico. Its uploaded `/pico_manual` copy is separate. The Windows workspace copy is also a snapshot, so changes on the Pi do not automatically appear there.
