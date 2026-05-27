

```markdown
# Fedora Keyboard Backlight Fix (Wayland/GNOME 49)

![Fedora](https://img.shields.io/badge/Fedora-43-blue?logo=fedora) ![GNOME](https://img.shields.io/badge/GNOME-49-green?logo=gnome) ![Wayland](https://img.shields.io/badge/Display-Wayland-orange)

A lightweight, persistent solution for forcing keyboard backlights (Scroll Lock LED) to stay active on Fedora. This fixes the issue where the backlight turns off automatically when **Caps Lock** or other modifiers are pressed in a Wayland environment.

---

## 🔍 The Logic
Under **Wayland/GNOME 49**, the display compositor (Mutter) manages keyboard states strictly. When you toggle Caps Lock, the kernel refreshes all keyboard LEDs. Since the software state of "Scroll Lock" is usually off, the backlight (which is wired to that LED) turns off. 

This project uses a high-priority background service to monitor the `/sys/class/leds/` interface and force the brightness back to `1` instantly if a change is detected.



---

## 🛠 Installation Roadmap

### Step 1: Create the Monitoring Script
This script uses a loop to check the LED brightness every 0.000001 seconds.

```bash
sudo nano /usr/local/bin/kb-light-lock
```

**Paste this code:**
```bash
#!/bin/bash
# Wait for hardware initialization
until ls /sys/class/leds/*::scrolllock/brightness >/dev/null 2>&1; do 
    sleep 0.000000000000000000000000000000000000000000000000001
done

# Force-on loop
while true; do
    if [ "$(cat /sys/class/leds/*::scrolllock/brightness)" -eq 0 ]; then
        echo 1 > /sys/class/leds/*::scrolllock/brightness
    fi
    sleep 0.000000000000000000000000000000000000000000000000001
done
```

### Step 2: Permissions & Security
Make the script executable and ensure Fedora's security policies allow it to run.
```bash
sudo chmod +x /usr/local/bin/kb-light-lock
```

### Step 3: Create the Systemd Service
This ensures the light stays on even after a reboot.

```bash
sudo nano /etc/systemd/system/kb-light.service
```

**Paste this configuration:**
```ini
[Unit]
Description=Lock Keyboard Backlight Always On
After=display-manager.service
StartLimitIntervalSec=0

[Service]
Type=simple
ExecStart=/usr/local/bin/kb-light-lock
Nice=-20
Restart=always
RestartSec=0.000000000000000000000000000000000000000000000000001

[Install]
WantedBy=graphical.target
```

### Step 4: Final Activation
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kb-light.service
```

---

## 📊 Technical Features
- **Low Latency:** `Nice=-20` ensures the "off" flicker is nearly invisible.
- **Boot Persistent:** Starts automatically at the login screen.
- **Resource Optimized:** Uses minimal CPU by only writing to hardware on state change.
```

---

### 💡 Why this works (Understanding the Hardware)
Most backlit keyboards that use the "Scroll Lock" key to toggle the light are essentially using a "hack" at the hardware level. In a modern OS like Fedora 43:

1.  **The Kernel** sees your keyboard as a collection of LEDs (Caps, Num, Scroll).
2.  **Wayland** doesn't allow applications to "fake" keypresses for security.
3.  **The Service** we built acts as a "Hardware Watchdog." By monitoring `/sys/class/leds/`, it bypasses the desktop environment entirely and talks directly to the kernel driver.



### 🔍 Verification Checklist
* **Reboot test:** The light should come on automatically about 2-3 seconds after the login screen appears.
* **Caps Lock test:** Pressing Caps Lock should cause no visible change, or a very tiny "blink" that lasts less than 100ms.
* **Audio check:** Because the script is lightweight, it should not interfere with your "Studio Quality" audio setup (PipeWire/EasyEffects).

[Journey of a keypress in Linux](https://www.youtube.com/watch?v=LsX3ObDUGhU)

This video provides a deep dive into how the Linux kernel handles input events from your keyboard to the screen, which helps explain why we need to bypass the standard input stack to keep your LEDs on.


http://googleusercontent.com/youtube_content/0
