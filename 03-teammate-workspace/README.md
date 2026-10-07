# Using the teammate workspace

I made a separate copy of the project so someone else can work on MoveIt without changing the files I used for the keyboard motor test. We still SSH into the same Pi account, `cajiatas`, but the project folders and Docker environments are separate.

| What it is | Location or name |
| --- | --- |
| My original project on the Pi | `~/avc_ws` |
| Teammate project on the Pi | `~/avc_teammate/workspace` |
| Teammate project inside Docker | `/ws` |
| Teammate Docker image | `avc-moveit-teammate:jazzy` |
| Teammate Docker container | `avc_moveit_teammate` |

I checked the project copy and backups on October 6, 2026. Both the project archive and Docker image archive are on the Pi. The package check below is how I confirm that MoveIt has finished installing before I start working.

## Getting into the folder

First, follow [the SSH guide](../02-ssh-into-the-pi/README.md) to log in as `cajiatas`. The commands in this section run in the Pi's normal SSH terminal.

To look at or edit the teammate files directly on the Pi:

```bash
cd ~/avc_teammate/workspace
pwd
ls
```

`pwd` should show `/home/cajiatas/avc_teammate/workspace`. Changing into this folder does **not** open the ROS environment. It only moves the terminal into the teammate's project directory.

## Finishing setup if needed

If the one-time setup has not finished, run this from the normal SSH terminal:

```bash
bash ~/avc_teammate/setup.sh
```

Run it as `cajiatas`, without putting `sudo` before `bash`. The script requests sudo for Docker itself. Enter the Pi account's sudo password only in the terminal when prompted; password characters will not appear as you type.

The script saves the original ROS Docker image before building a separate image with `ros-jazzy-moveit`. It also saves `avc-moveit:jazzy` if that image exists. It requires at least 10 GiB free before the backup/build stage. The working `avc-ros:jazzy` image tag is kept, and the teammate image gets its own name.

If the teammate container already exists with the expected label, the script starts it and checks MoveIt instead of replacing it. Wait for the success message; if it reports an error, the setup still needs attention.

## Opening the ROS and MoveIt environment

For normal work, use this from the Pi's SSH terminal:

```bash
bash ~/avc_teammate/start.sh
```

The launcher starts the teammate container and opens a shell at `/ws`. It also loads ROS 2 Jazzy automatically. Inside that shell, check:

```bash
pwd
echo "$ROS_DISTRO"
echo "$ROS_DOMAIN_ID"
ros2 pkg prefix moveit_ros_move_group
```

The expected results are `/ws`, `jazzy`, `42`, and `/opt/ros/jazzy`. If the package command fails, do not assume MoveIt is installed successfully. If the launcher says to run `setup.sh` first, return to the setup step.

Files under `/ws` are the same files as `~/avc_teammate/workspace` on the Pi. For example, a file saved as `/ws/notes.txt` appears at `~/avc_teammate/workspace/notes.txt` over SSH.

Use `start.sh` for editing and building. It runs the shell with the SSH account's user and group IDs so new files stay editable as `cajiatas`. Its home directory is `/ws/.home`. For a second ROS terminal, open another SSH connection and run the same launcher.

Type `exit` to leave the container shell and return to SSH. The container and its changes remain, and the launcher can start it again after a Pi reboot. Project files remain in the mounted folder even if the container is removed. Package changes made inside the container would need to be recreated if that container were deleted.

If I need to install an extra Ubuntu or ROS package, I open a root shell in **this teammate container** from the Pi terminal:

```bash
sudo docker exec -it avc_moveit_teammate bash
source /opt/ros/jazzy/setup.bash
apt-get update
```

Then I use `apt-get install` with the package name I need. I type `exit` afterward and go back to `start.sh` for normal editing. This keeps package installation inside the teammate container.

## What this environment can access

I set the teammate environment to ROS domain `42`, localhost discovery, and Docker's private networking to separate these experiments from the wheel setup. Pico device access and graphical display forwarding are not configured. RViz or the MoveIt Setup Assistant still needs display setup before its graphical interface can be used.

Since we share one Linux account, this is a separate working copy, not a permission boundary. Stick to the teammate folder and launcher; the original `~/avc_ws` is still accessible from SSH.

## Where the backups are

On the Pi, this prints the dated backup folder:

```bash
cat ~/avc_teammate/backup-path.txt
```

The saved project archive is `avc_ws.tar.gz`; the Docker image archive is `docker-images.tar`. The image archive covers saved images, not unsaved changes inside other containers. Existing Pico backup files are included in the project archive; this did not read the live Pico firmware.

I also saved the verified project archive on my Windows computer under `AVC COMPETITION/backups/before-teammate-20261006T214619439765Z/`. That Windows copy is the project-file backup; the Docker image backup is on the Pi.
