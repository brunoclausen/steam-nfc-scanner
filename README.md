Overview (one-line)

Run an Arch (or other) distrobox that has nfcpy and libnfc/pcsc installed; the container reads NFC tags and uses distrobox-host-exec (or a fallback to distrobox-enter) to open steam://... on the host so Flatpak/native Steam launches the game.

1) Create the distrobox container (example: Arch)

Run on the host:

# Create an Arch container named "nfc-arch" (first time will download image)
distrobox-create --name nfc-arch --image archlinux:latest

# Optional: if you prefer Debian/Ubuntu:
# distrobox-create --name nfc-ubuntu --image ubuntu:24.04

2) Install dependencies inside the container

Enter the container and install required packages for NFC:

# Enter the container (interactive)
distrobox-enter --name nfc-arch

# Inside the container now:
# Update & install packages (Arch example)
sudo pacman -Syu --noconfirm
sudo pacman -S --noconfirm python python-pip libnfc pcsc-lite pcsc-tools

# If using Debian/Ubuntu container instead:
# sudo apt update && sudo apt install -y python3 python3-pip libnfc-bin pcscd pcsc-tools

# Install python libs for your user
python -m pip install --user nfcpy pyscard


Notes:

If your reader needs pcscd, enable it inside the container only if container supports systemd or run pcscd manually inside container.

If sudo pacman isn’t available in your chosen image, use the appropriate package manager.

3) Scripts (put these inside the container in ~/.local/bin/)

Create ~/.local/bin/nfc_steam.py (make executable chmod +x ~/.local/bin/nfc_steam.py):

#!/usr/bin/env python3
"""
nfc_steam.py — run inside distrobox.
Reads NFC UID -> looks up appid in ~/.config/nfc_steam/tags.json
Tells host to open steam://rungameid/<appid> via distrobox-host-exec if available,
otherwise falls back to invoking distrobox-enter to call xdg-open on host.
"""
import os
import json
import subprocess
import time
import sys

try:
    import nfc
except Exception:
    print("Missing nfcpy. Run: python -m pip install --user nfcpy pyscard")
    sys.exit(1)

CONFIG_DIR = os.path.expanduser("~/.config/nfc_steam")
TAGS_FILE = os.path.join(CONFIG_DIR, "tags.json")
CONTAINER_NAME = os.environ.get("DISTROBOX_NAME", "nfc-arch")  # helpful fallback

def ensure_config():
    os.makedirs(CONFIG_DIR, exist_ok=True)
    if not os.path.exists(TAGS_FILE):
        with open(TAGS_FILE, "w") as f:
            json.dump({}, f, indent=2)

def load_tags():
    try:
        with open(TAGS_FILE, "r") as f:
            return json.load(f)
    except Exception:
        return {}

def launch_on_host(url):
    """
    Try distrobox-host-exec first (recommended). If not available fallback to
    calling distrobox-enter --name <container> -- xdg-open "<url>"
    """
    # preferred: distrobox-host-exec <cmd...>
    try:
        subprocess.run(["distrobox-host-exec", "xdg-open", url], check=False,
                       stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        return
    except FileNotFoundError:
        pass

    # fallback: use distrobox-enter to execute host xdg-open via the container runtime.
    # This invokes the host from the *host* by calling distrobox-enter which runs a command
    # outside the container. Works on many distrobox installations.
    try:
        subprocess.run(["distrobox-enter", "--name", CONTAINER_NAME, "--", "xdg-open", url],
                       check=False, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        return
    except FileNotFoundError:
        pass

    # last resort: print instruction — user can run the host command manually
    print("Could not call host xdg-open automatically. Please run on host:")
    print(f'  xdg-open "{url}"')

def launch_steam(appid):
    url = f"steam://rungameid/{appid}"
    launch_on_host(url)

def on_connect(tag):
    uid = tag.identifier.hex().upper()
    ts = time.strftime("%Y-%m-%d %H:%M:%S")
    print(f"[{ts}] NFC UID: {uid}")
    tags = load_tags()
    if uid in tags:
        entry = tags[uid]
        appid = entry.get("appid")
        name = entry.get("name", "")
        print(f"Matched: {uid} -> {appid} ({name}) — launching on host...")
        try:
            launch_steam(appid)
        except Exception as e:
            print("Launch failed:", e)
    else:
        print("No mapping for this tag. Use tag_writer.py to register it.")
    # return True to finish current connection
    return True

def main():
    ensure_config()
    print("NFC Steam listener (in distrobox). Ctrl+C to stop.")
    try:
        clf = nfc.ContactlessFrontend('usb')
    except Exception as e:
        print("Cannot open NFC reader via 'usb':", e)
        sys.exit(1)

    try:
        while True:
            print("Waiting for NFC tag...")
            clf.connect(rdwr={'on-connect': on_connect})
            time.sleep(0.3)
    except KeyboardInterrupt:
        print("Stopped by user.")
    finally:
        try:
            clf.close()
        except Exception:
            pass

if __name__ == "__main__":
    main()


Create ~/.local/bin/tag_writer.py (also chmod +x):

#!/usr/bin/env python3
"""
tag_writer.py — run inside container.
Scan a tag and store UID -> appid,name in ~/.config/nfc_steam/tags.json
"""
import os, json, sys, time

try:
    import nfc
except Exception:
    print("Missing nfcpy. Run: python -m pip install --user nfcpy pyscard")
    sys.exit(1)

CONFIG_DIR = os.path.expanduser("~/.config/nfc_steam")
TAGS_FILE = os.path.join(CONFIG_DIR, "tags.json")

def ensure_config():
    os.makedirs(CONFIG_DIR, exist_ok=True)
    if not os.path.exists(TAGS_FILE):
        with open(TAGS_FILE, "w") as f:
            json.dump({}, f, indent=2)

def load_tags():
    try:
        with open(TAGS_FILE, "r") as f:
            return json.load(f)
    except Exception:
        return {}

def save_tags(tags):
    with open(TAGS_FILE, "w") as f:
        json.dump(tags, f, indent=2)

def on_connect(tag):
    uid = tag.identifier.hex().upper()
    print("Found UID:", uid)
    appid = input("Enter Steam AppID (e.g. 730): ").strip()
    name = input("Optional name/description: ").strip()
    tags = load_tags()
    tags[uid] = {"appid": appid, "name": name}
    save_tags(tags)
    print("Saved:", uid, "->", appid, name)
    return True

def main():
    ensure_config()
    print("Place tag to program. Ctrl+C to abort.")
    try:
        clf = nfc.ContactlessFrontend('usb')
    except Exception as e:
        print("Cannot open NFC reader:", e)
        sys.exit(1)
    try:
        clf.connect(rdwr={'on-connect': on_connect})
    except KeyboardInterrupt:
        pass
    finally:
        try:
            clf.close()
        except Exception:
            pass

if __name__ == "__main__":
    main()

4) Give the container access to the USB device

When running the container interactively you can pass the USB device(s):

# Example interactive test run (host terminal)
# allow access to all USB buses (may require root); good for quick testing
distrobox-enter --name nfc-arch -- --device /dev/bus/usb

# Or with privileged (less secure)
distrobox-enter --name nfc-arch -- --privileged


For persistent auto-start we will configure systemd user unit on the host (next).

5) systemd --user unit (on the host) to run the listener at login

Create file ~/.config/systemd/user/nfc-steam-distrobox.service on the host:

[Unit]
Description=NFC Steam Launcher (distrobox)
After=default.target

[Service]
Type=simple
# ExecStart runs the listener inside the container and passes /dev/bus/usb devices into the container.
ExecStart=/usr/bin/distrobox-enter --name nfc-arch -- ~/.local/bin/nfc_steam.py
# If you need to expose USB devices explicitly, use:
# ExecStart=/usr/bin/distrobox-enter --name nfc-arch -- --device /dev/bus/usb ~/.local/bin/nfc_steam.py

Restart=on-failure
RestartSec=5
Environment=DISTROBOX_NAME=nfc-arch

[Install]
WantedBy=default.target


Then enable & start (on the host):

systemctl --user daemon-reload
systemctl --user enable --now nfc-steam-distrobox.service


Notes:

If your distrobox supports --device or --privileged when used non-interactively, uncomment and adapt the ExecStart commented example.

If distrobox-enter on your host is in a different path, adjust ExecStart to the full path (find with which distrobox-enter).

6) Testing procedure

On the host, ensure Steam handles steam:// links:

xdg-open "steam://rungameid/730"


Steam should open. If not, adjust to flatpak run com.valvesoftware.Steam -applaunch 730 on host.

Start the container listener manually (host):

distrobox-enter --name nfc-arch -- ~/.local/bin/nfc_steam.py


or, if testing with device exposure:

distrobox-enter --name nfc-arch -- --device /dev/bus/usb ~/.local/bin/nfc_steam.py


Run tag_writer.py inside the container to register a tag:

distrobox-enter --name nfc-arch -- ~/.local/bin/tag_writer.py


Place tag on reader — the container should detect the UID, look up ~/.config/nfc_steam/tags.json (inside the container), then call host xdg-open steam://rungameid/<appid> and Steam should start.

7) Fallback & Troubleshooting

distrobox-host-exec not found: the script falls back to distrobox-enter --name <container> -- xdg-open <url>. If both are missing the script will print the host command to run manually.

Reader not visible in container:

Try --device /dev/bus/usb or --privileged.

Check lsusb on host; get vendor/product with lsusb and use that to craft a precise ContactlessFrontend('usb:072f:2200') if needed.

Permission errors as non-root: ensure host udev rule grants access to your user (on host /etc/udev/rules.d/99-acr122u.rules as earlier described). The container will then see the device via host.

If pcscd is required for your reader and starting it inside container is hard, you can run pcscd on host and forward device access to container.

If Steam flatpak is sandboxed and doesn't open with xdg-open from host, try flatpak run com.valvesoftware.Steam -applaunch <appid> on host.

8) Security / ergonomics tips (recommended)

Keep ~/.config/nfc_steam/tags.json backed up (it lives in the container home). If you prefer host-side mapping, place tags.json on a host path and bind-mount it into container.

To bind-mount host config into container automatically, set up a distrobox with --home or --shared-home options or use container volumes.

Add a double-scan safety: require same UID scanned twice within 5s before launching to avoid accidental triggers (I can add that to the script if you want).

If you want, I’ll:

add the double-scan confirmation to nfc_steam.py now, or

produce a version that reads AppID from an NDEF record on the tag (so you can program tags with AppID and avoid tags.json), or

produce a one-file systemd + install script that sets up the whole pipeline automatically (host udev rule, container creation, package installs, copying scripts, enabling service).

Pick one and I’ll add it immediately. Geeky grin engaged — this is fun.
