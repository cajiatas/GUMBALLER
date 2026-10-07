# GUMBALLER

Our files for the gumballer.

I put these guides together so the group can understand how I got keyboard driving working and use the Pi without having to repeat the whole setup.

| Folder | What I explain |
| --- | --- |
| [01-keyboard-motor-control](01-keyboard-motor-control/README.md) | The process I followed, how ROS talks to the Pico, and the commands I use to drive the wheels with the keyboard. |
| [02-ssh-into-the-pi](02-ssh-into-the-pi/README.md) | How I connect to the Pi from a terminal. This assumes remote network access is already set up; it does not cover setting up Tailscale. |
| [03-teammate-workspace](03-teammate-workspace/README.md) | How to enter the separate workspace and Docker container for MoveIt experiments. |

If you are just trying to get started, read the SSH guide first. Then use the keyboard guide to drive the robot or the teammate guide to work on MoveIt.

## The names I use

| Name | What it is |
| --- | --- |
| `BubbleGum` | The Raspberry Pi's hostname. |
| `cajiatas` | The shared SSH login on the Pi. |
| `~/avc_ws` | My original working project on the Pi. |
| `avc-ros:jazzy` | The Docker image for my ROS 2 keyboard setup. |
| `avc_manual` | The container I start for keyboard driving. |
| `~/avc_teammate/workspace` | The separate project copy for teammate development. |
| `avc_moveit_teammate` | The separate container used by the teammate launcher. |

These notes describe the setup checked on October 6, 2026. The project folders and scripts live on the Pi; these three folders explain how to use them. Our shared login can access both project copies, so use the teammate workspace for experiments.
