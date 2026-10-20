# PocketMac

Control your MacBook Air (M2) from your iPhone 12 — see the screen live and drive it by touch, even with the lid closed. No cloud service in the middle, end-to-end encrypted, locked to your one iPhone.

```
 iPhone 12 ──(direct, encrypted)── MacBook Air M2
   PocketMac app                     PocketMac Host (menu bar)
```

## What's in here

| Folder | What |
|---|---|
| `Host/` | **PocketMac Host**: the Mac menu-bar app (screen capture, hardware encoder, input, lid-closed mode) |
| `iOS/` | **PocketMac**: the iPhone app (SwiftUI + UIKit) |
| `Shared/` | Protocol + crypto + networking used by both |
| `tools/` | Build scripts and tests (`run-tests.sh`, `pmtest`, `wiretest`, `check-ios.sh`) |
| `.github/workflows/build-ios.yml` | Cloud build that makes the `.ipa` for sideloading |

## Features

- **Live screen** with hardware HEVC encode on the M2 and hardware decode on the iPhone. 30 or 60 fps, three quality levels.
- **Touch control** in two modes:
  - **Trackpad**: slide to move the pointer, tap to click, tap-then-slide to drag, two fingers to scroll (with momentum), two-finger tap for right-click.
  - **Touch**: tap exactly where you want to click, long-press for right-click.
- **Pinch to zoom**. The picture follows the pointer while you're zoomed in.
- **Three-finger gestures**: swipe ← → to switch Spaces, ↑ for Mission Control, ↓ for app windows. Three-finger tap shows the keyboard.
- **Keyboard** with any language (Hindi works), plus a shortcut bar: Esc, Tab, sticky ⌘⌥⌃⇧, arrows, ⌘C/V/X/Z/A/F/S/W/T, ⌘Tab, ⌘Space, Home/End/PgUp/PgDn, F1–F12.
- **Lid closed**: keeps the Mac awake and adds a virtual screen shaped exactly like your iPhone (1266×584 pt @2x = 2532×1170 px). Text is readable without zooming.
- **Clipboard both ways**: Mac copies land on the iPhone automatically. Send or paste the iPhone clipboard to the Mac with one tap.
- **Quick actions**: Spotlight, Mission Control, ⌘Tab, show desktop, screenshot, Force Quit, play/pause/next, volume, wake or turn off the screen, lock, sleep, lid mode on/off.
- **Mac status on the phone**: battery %, charging, round-trip latency, fps.
- **Battery guard**: turns lid-closed mode off when the Mac battery reaches 15% (adjustable), so it can sleep safely.
- **Auto-reconnect** when the network changes (Wi-Fi ↔ mobile data).

## Security model

- **Mac ID = key fingerprint.** The 12-character ID (e.g. `1N81-1EPB-JZXH`) is derived from the Mac's long-term X25519 public key. Typing the ID pins that key, so a fake "Mac" can't complete the login and can't collect password guesses.
- **Password** is mixed into the key exchange with PBKDF2-SHA256 (210,000 rounds). The Mac stores only the derived key, never the password.
- **Locked to one iPhone.** The first iPhone to log in gets paired. After that, every other device is refused before it can even try a password. To add a new phone, click "Pair new iPhone" on the Mac, which opens a 5-minute window.
- **Brute force**: after 5 wrong passwords, logins lock for 30 s, then the lockout doubles with each further wrong try (up to 1 h).
- **Encryption**: X25519 ephemeral + static keys (forward secrecy) → HKDF → AES-256-GCM with counter nonces. Replayed, reordered or modified packets are rejected.
- On the iPhone, the saved login sits in the Keychain behind Face ID / passcode, and the iPhone's own key never leaves it.
- The Mac shows a different menu-bar icon while an iPhone is connected, and macOS shows its screen-recording indicator.

All of this is covered by `tools/run-tests.sh`: wrong password, a second phone, a fake Mac ID, and lockout.

## About "no server" and different networks

- **Same Wi-Fi, or no Wi-Fi at all**: the iPhone finds the Mac by itself (Bonjour, or Apple peer-to-peer Wi-Fi). Truly no server, works even in a car with no router.
- **Different networks (mobile data, office Wi-Fi…)**: two devices behind NAT, especially on a mobile carrier, cannot find each other with zero infrastructure. Something has to tell each one where the other is. PocketMac uses **[Tailscale](https://tailscale.com)**, a free mesh VPN (WireGuard):
  - Its coordination server only swaps public keys and addresses. It never sees your screen or input.
  - Traffic goes **directly** between the phone and the Mac. Only when a carrier blocks direct paths does it fall back to an encrypted relay, which still can't read anything.
  - PocketMac's own encryption runs on top, so even Tailscale can't see inside.
  - Want zero third parties? Run your own coordination server ([Headscale](https://github.com/juanfont/headscale)), or use a public IPv6 / port-forward on port 47800 and enter that address instead.

## Setup

### 1. Mac: PocketMac Host

```bash
tools/build-host.sh
```

```bash
cp -R "build/PocketMac Host.app" /Applications/
```

```bash
open "/Applications/PocketMac Host.app"
```

In the window that opens:
1. **Set a password** (8+ characters).
2. **Screen Recording → Allow…** and turn on *PocketMac Host*.
3. **Accessibility → Allow…** and turn on *PocketMac Host*.
4. Optional: **Start at login**, **Keep running with lid closed**, **Install helper** (lets the iPhone toggle lid mode, and lets the battery guard work).

> The app is signed ad-hoc. If you rebuild it, macOS forgets the two permissions: remove and re-add it in System Settings → Privacy & Security.

### 2. iPhone: build the `.ipa` in the cloud (no Xcode needed)

1. Create a **private** repository on github.com (e.g. `pocketmac`).
2. Push this folder to it:
   ```bash
   git remote add origin https://github.com/<you>/pocketmac.git
   ```
   ```bash
   git push -u origin main
   ```
3. On GitHub: **Actions → Build iPhone app (.ipa)**. It starts on every push, or you can press **Run workflow**. It takes about 5–10 minutes.
4. Open the finished run → **Artifacts → PocketMac-ipa** → download and unzip it to get `PocketMac.ipa`.

### 3. Install it on the iPhone over USB

**Sideloadly** (free, Mac app):
1. Plug the iPhone in with a cable and tap *Trust* on the phone.
2. Drag `PocketMac.ipa` into Sideloadly, enter your Apple ID, then press Start.
3. On the iPhone: **Settings → General → VPN & Device Management** → trust your Apple ID.
4. On iOS 16+: **Settings → Privacy & Security → Developer Mode** → On (the phone restarts).

**3uTools** (you already have it): *Toolbox → IPA Signature* with your Apple ID, then *Install*.

A free Apple ID signature lasts **7 days**. Re-sign and reinstall weekly (Sideloadly can do it automatically). Your saved Macs and logins stay on the phone.

### 4. Connect

- **At home**: open PocketMac → the Mac appears under **Nearby** → tap it → enter the password → done. Face ID unlocks it next time.
- **From anywhere**: install Tailscale on the Mac and the iPhone and sign in with the same account. In the Mac's PocketMac Host window, copy the **Tailscale** address (100.x.y.z). On the iPhone: edit the Mac → **Internet address** → paste it. PocketMac tries the nearby path and the Tailscale path at once and keeps whichever connects first.
  - In the Tailscale admin page, turn on **Disable key expiry** for the Mac so it never logs out while you're away.

## Lid closed

1. Mac menu bar → **Keep running with lid closed** (asks for your Mac password). This runs `pmset -a disablesleep 1`.
2. Close the lid. Within about 2 seconds PocketMac adds a virtual screen shaped for your iPhone, and the stream switches to it.
3. Open the lid and the virtual screen disappears.

⚠️ With this on, the Mac doesn't sleep at all. Don't put it in a closed bag while running. It can overheat and drain the battery. The battery guard turns it off at 15% if the helper is installed.

Limitations:
- A Mac that's already **asleep** can't be woken over the internet. Keep lid mode on if you need it later.
- When the Mac is **locked**, macOS may block the lock screen from being captured or typed into by apps. If that happens on your Mac, set **Require password after screen saver / display off** to a longer delay in Lock Screen settings.
- macOS shows a "Screen Recording" reminder roughly once a month. Click Allow when you're at the Mac.

## Battery & performance

- **Mac**: when the screen doesn't change, nothing is encoded and nothing is sent, so idle cost is ~0. Encoding runs on the M2's media engine, not the CPU cores. With no phone connected, the host only listens on a socket.
- **iPhone**: hardware decode straight into the display layer with no CPU copies. 30 fps by default; switches to *Battery saver* automatically in Low Power Mode.
- Frames only go out when the network can carry them (at most a couple in flight). On a slow link you get a lower frame rate, not growing lag. Bitrate adapts automatically.
- After motion stops, the last frame is re-sharpened up to 3 times, so text gets crisp even on mobile data.

## Measured resource use (MacBook Air M2, PocketMac Host)

**Idle (no iPhone connected)** — 60 s average:

| Version | CPU | Wake-ups | Energy impact | Memory |
|---|---|---|---|---|
| v1 | 2.42 % | 4.4 /s | 2.43 | 33 MB |
| **now** | **0.02 %** | **0–0.1 /s** | **0.02** | **13 MB** |

For comparison on the same Mac at the same time: Claude app 0.58 % (260 MB), Wispr Flow 0.45 % (163 MB), ControlCenter 0.45 % (54 MB), Finder 0.01 % (144 MB), Tailscale's network extension 0.00 % (12 MB).

**Connected, A/B on an identical moving test picture, nobody touching** (30 s, two rounds each):

| | Before smart frame rate | After |
|---|---|---|
| CPU | 4.1–4.5 % | 2.7–2.8 % |
| Wake-ups | 5.9–10.3 /s | 3.1–3.2 /s |
| Frames sent | 651 | 227 |
| Data | 123 kbps | 29 kbps |

While you touch or type, it runs at full frame rate; when only small things animate by themselves it drops to 10 fps. A busy real screen (big changes, ~30 fps) measured 4.9 % CPU.

## Development

```bash
tools/run-tests.sh /tmp/pm-test
```
Runs a throw-away test-pattern host on localhost and checks pairing, wrong password, second phone refused, fake Mac ID, lockout, HEVC + H.264 decode.

```bash
tools/check-ios.sh
```
Type-checks the iPhone app against Mac Catalyst's UIKit, so cloud builds don't fail.

```bash
tools/test-catalyst.sh /tmp/pm-cat
```
Runs the iPhone app as a Mac app connected to a test host.

## Ideas for next versions

- Mac **audio** on the iPhone (ScreenCaptureKit audio → AAC).
- **File transfer**: send photos to the Mac's Downloads folder; download files from the Mac.
- **Multiple displays**: pick which screen to view.
- **Hardware keyboard / mouse** connected to the iPhone or iPad.
- **Home Screen widget / Shortcuts**: "Lock my Mac", "Mac battery".
- **Privacy curtain**: black out the Mac's own screen while you control it remotely.
