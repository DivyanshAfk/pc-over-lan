# pc-over-lan

Two small bash scripts that let an always-on machine (a homelab laptop, a Pi, anything on the LAN) wake a sleeping Windows PC with Wake-on-LAN, run a job on it over SSH, and switch it off again.

- `pcctl on|off|status` wakes, shuts down and checks the PC.
- `pcpy script.py [args]` wakes the PC if needed, copies the project over, runs the script in the project's virtual environment on the PC, and streams the output back.

It was written for ML and data jobs that need the PC's GPU, but it doesn't care what the script does.

## Requirements

On the always-on machine: `bash`, `wakeonlan`, `nc` (netcat), `ssh` and `scp`.

On the PC:

- Wake-on-LAN enabled in the BIOS/UEFI (often called Wake on LAN or Power On By PCI-E).
- In Windows, the wired adapter allowed to wake the computer, ideally "only a magic packet".
- OpenSSH Server installed and running (an optional Windows feature).
- A virtual environment at `<project>\.venv`.

Wake-on-LAN works over a wired connection. The PC and the sender must be on the same LAN segment, because the magic packet is a UDP broadcast.

## Setup

Set the variables the scripts ask for, for example in a file you source from your shell profile:

```bash
export PC_IP=...            # address of the PC
export PC_MAC=...           # MAC of the wired adapter
export PC_BCAST=...         # LAN broadcast address, e.g. x.y.z.255
export PC_USER=...          # user on the PC
export PC_KEY_OFF=~/.ssh/id_pc_off
export PC_KEY_WORK=~/.ssh/id_pc_work
export PCPY_LOCAL_SRC=~/myproject/src
export PCPY_REMOTE_DIR=C:/myproject
export PCPY_DATA_DIR='C:\myproject-data'
```

Put `pcctl` and `pcpy` somewhere on your `PATH` (`pcpy` calls `pcctl`), then:

```bash
pcctl status        # up or down
pcctl on            # send the magic packet
pcpy train.py       # wake if needed, sync, run, stream output
pcctl off           # shut the PC down
```

## The off key

There are two SSH keys on purpose. The work key runs jobs. The off key exists only to shut the PC down, so the everyday key can't do the dangerous thing by accident.

`pcctl off` connects without passing a command, so the PC's side has to decide what that key does. One way is to restrict the key in `authorized_keys` with a forced command:

```
command="shutdown /s /t 0",no-pty,no-port-forwarding ssh-ed25519 AAAA... pc-off
```

On Windows OpenSSH, keys for administrator accounts go in `C:\ProgramData\ssh\administrators_authorized_keys`. Check that file's permissions after editing it.

## Things to know

- **Port 22 open is not "ready".** `pcpy` polls the port for up to three minutes and then waits five more seconds.
- **`set VAR=value&& next` has no space before `&&`.** In `cmd`, a space there ends up in the value.
- **The output is piped through `tr -d '\0'`** to strip NUL bytes that Windows tools can emit.
- **The job runs in the foreground of the SSH session.** If the connection drops, the job probably stops. Run it detached if that matters.
- **Wake-on-LAN has no authentication.** Anyone on the LAN can send the packet. Keep it on your LAN or behind a VPN, never exposed to the internet.

## Licence

None chosen yet. Add one before reusing this elsewhere.
