# luna

Wrapper for [xtool](https://github.com/xtool-org/xtool) to automate develop iOS app and install it using [TrollStore](https://github.com/opa334/TrollStore)


## Why

Sideloading an app with a free Apple Developer ID will stops the app from launching after **7 days** and has to be re-signed and re-installed. TrollStore installs apps permanently by exploiting an AMFI/CoreTrust signature-verification bug, so once installed, an app just... stays installed.

`luna` automates that workflow: build the `.ipa` with xtool, serve it over the local network, and tell TrollStore (over SSH) to fetch and install it then launching the app afterward.

## TrollStore settings
 
For `luna install`/`luna dev` to work without you tapping through prompts on the device, open **TrollStore → Settings** and set:
 
- **URL Scheme Enabled** → On (this is what lets `luna` trigger installs via the `apple-magnifier://install?url=...` scheme over SSH)
- **Show Install Confirm Alert** → **Never**
> ⚠️ **Warning:** Setting the install confirm alert to "Never" means *any* app or script that can trigger TrollStore's URL scheme can silently install an `.ipa` on your device with no prompt. This is convenient for automation but removes a safety check, only enable it if you understand the tradeoff, and keep your device off untrusted networks while it's set this way.

## Requirements

- **Host machine**: [xtool](https://github.com/xtool-org/xtool) installed and in `PATH`
- **iOS device**:
  - [TrollStore](https://github.com/opa334/TrollStore) installed
  - An SSH server installed and running, reachable from the host over the network. If your device isn't jailbroken yet, jailbreak it first (e.g. with [Dopamine](https://ellekit.space/dopamine/)), then install OpenSSH through your package manager of choice (Sileo, Zebra, etc.) and make sure the service is enabled/started
- Host and device on the same network (luna's HTTP server serves the `.ipa` to the device)
- `python3` (used for the temporary HTTP server on the host)
- Optional: [`sshpass`](https://linux.die.net/man/1/sshpass) if you want to use password-based SSH auth instead of keys/agent

## Installation

```bash
git clone https://github.com/wenyhtt/luna.git
cd luna
chmod +x luna
```

Run it from inside your xtool project directory (or move/symlink `luna` somewhere on your `PATH`).

## Setup

Initialize a config file in your project directory:

```bash
./luna init
```

This creates a `.lunabuild` file. Fill in the required values:

```bash
./luna config ipa <path>       # path to the built .ipa (e.g. ./yourapp.ipa)
./luna config device <ip>      # your iOS device's IP address
./luna config url <scheme>     # your app's own URL scheme, so luna can open it after install
./luna config port <port>      # (optional) local HTTP server port, default 8000
```

View your current config anytime:

```bash
./luna config show
```

### SSH authentication

By default `luna` connects as `mobile@<device-ip>` using standard SSH (key or agent-based auth). If you'd rather use a password, create a `.env` file in the project directory:

```
LUNA_SSH_PASS=yourpassword
```

When `LUNA_SSH_PASS` is set, `luna` uses `sshpass` instead of plain `ssh`.

## Usage

| Command | Description |
|---|---|
| `luna init` | Create a `.lunabuild` config in the current directory |
| `luna config show` | Print the current config |
| `luna config ipa <path>` | Set the `.ipa` path |
| `luna config device <ip>` | Set the target device's IP |
| `luna config url <scheme>` | Set your app's own URL scheme (used to auto-launch it after install) |
| `luna config port <port>` | Set the local HTTP server port (default `8000`) |
| `luna build` | Run `xtool dev build --ipa` |
| `luna install` | Serve the `.ipa` locally and trigger a TrollStore install over SSH |
| `luna dev` | `build` + `install` in one shot |
| `luna version` | Print the luna version |

### Typical workflow

```bash
./luna init
./luna config ipa ./MyApp.ipa
./luna config device 192.168.1.50
./luna config url myapp

./luna dev
```

`luna dev` will:
1. Build the `.ipa` with `xtool dev build --ipa`.
2. Spin up a temporary `python3 -m http.server` serving the `.ipa`'s directory.
3. SSH into the device and trigger TrollStore's install URL scheme (`apple-magnifier://install?url=...`) pointed at the local server.
4. Wait for the install to finish, then (if a URL scheme is configured) SSH into the device again and open your app via its own URL scheme.
5. Tear down the HTTP server.

> Set `URL_SCHEME` to whatever custom URL scheme your app registers for itself (e.g. `myapp://`) — luna uses it to trigger the app to open once TrollStore has installed it. If you skip this, `luna install`/`luna dev` will still install the app, it just won't auto-launch it afterward.

## Configuration files

**`.lunabuild`** plain `key=value` file, written/read by `luna config`:

```ini
IPA_PATH=./yourapp.ipa
DEVICE_IP=192.168.1.50
URL_SCHEME=myapp
PORT=8000
```

**`.env`** for SSH password auth:

```ini
LUNA_SSH_PASS=yourpassword
```

Both files are project-local, keep them out of version control (they can contain device IPs and credentials).

## License

GPL-3.0. See [LICENSE](LICENSE) for the full text.
