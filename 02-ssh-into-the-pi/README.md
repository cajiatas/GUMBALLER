# How I SSH into the Pi

I use SSH to open a terminal on the Raspberry Pi from my computer. Once I connect, the commands I type run on the Pi. This lets me use ROS, start the keyboard controller, and work on files without plugging a monitor and keyboard into the Pi.

This guide starts with remote network access already working. I am only covering the SSH connection here, not the process of setting up Tailscale.

## 1. Open a terminal on my computer

On Windows, I open **PowerShell** or **Windows Terminal**. On a Mac or Linux computer, the regular Terminal works too.

The Pi needs to be powered on and reachable. These were its addresses when I checked on October 6, 2026:

| Item | Value |
| --- | --- |
| Pi name | `BubbleGum` |
| Login | `cajiatas` |
| Address over the existing Tailscale connection | `100.89.80.46` |
| Address on the local network | `192.168.0.219` |

## 2. Connect

When the existing remote connection is available, I run this **on my computer**:

```bash
ssh cajiatas@100.89.80.46
```

If I am on the same local network as the Pi, I can use:

```bash
ssh cajiatas@192.168.0.219
```

I use one of those commands, depending on how I am reaching the Pi. The local address can change if the network assigns it a different address.

The command has two parts: `cajiatas` is the account I am signing into, and the address after `@` tells SSH which computer to connect to. This is the normal [SSH command format](https://learn.microsoft.com/en-us/windows/terminal/tutorials/ssh).

If SSH asks for a password, I enter the Pi account password in the terminal. Nothing appears while I type it, including asterisks. That is normal. If the computer already has an accepted SSH key configured, it may connect without a password prompt.

On a first connection, SSH may ask whether I trust the host key. I check the fingerprint with the person maintaining the Pi before accepting it. If SSH later reports that the host key has changed, I check what changed before reconnecting.

My private SSH key stays on my computer. Each teammate uses their own approved key or the account's password if password login is enabled; no private keys or passwords belong in this repo.

## 3. Make sure I am on the Pi

After signing in, the prompt normally looks like:

```text
cajiatas@BubbleGum:~ $
```

I can confirm the connection with:

```bash
hostname
whoami
pwd
```

The expected results are `BubbleGum`, `cajiatas`, and `/home/cajiatas` for a new login. At this point I am using the Pi's Linux terminal, even if I opened it from Windows.

## 4. Open the project I need

To look at the original project:

```bash
cd ~/avc_ws
ls
```

For teammate development, I use the separate copy instead:

```bash
cd ~/avc_teammate/workspace
ls
```

Here, `~` means `/home/cajiatas`. Changing into a folder does not start ROS or enter Docker. For that next step I use the [keyboard guide](../01-keyboard-motor-control/README.md) or the [teammate guide](../03-teammate-workspace/README.md).

Keyboard driving needs two Pi terminals. I open a second terminal tab on my computer and run the same SSH command again, then put the bridge in one tab and keyboard control in the other.

## 5. Leave the Pi

After stopping anything I am running, I type:

```bash
exit
```

That closes the SSH session and returns me to my computer's terminal. If I am inside a Docker shell, the first `exit` returns me to the Pi; a second `exit` disconnects SSH. Before leaving a driving session, I follow the shutdown steps in the keyboard guide.

## If the connection does not work

| What I see | What I check |
| --- | --- |
| `Connection timed out` or `No route to host` | The Pi is on, the address is correct, and my computer can reach that network. |
| `Connection refused` | I reached the address, but SSH may not be running or listening there. |
| `Permission denied` | The username and password/key are correct. If it says `publickey`, I need an accepted SSH key. |
| `ssh` is not recognized on Windows | The Windows OpenSSH Client needs to be available before this command can run. |
| A password seems like it is not typing | Password input is hidden; I type it once and press Enter. |

If I need to check the current local address, I run `hostname -I` from a terminal already on the Pi, or ask someone who has access to it.
