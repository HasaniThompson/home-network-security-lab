# Home Network Security Lab

*Raspberry Pi 4 — Pi-hole DNS Filtering Setup & Troubleshooting Log*

## Overview

This is a step-by-step record of building a home network security lab on a Raspberry Pi 4 (2GB), starting from hardware selection through OS setup, Pi-hole DNS filtering deployment, and troubleshooting real hardware/network issues encountered along the way. WireGuard VPN setup is planned as a follow-on phase and is not yet complete.

## Hardware

### Selection

- **Raspberry Pi 4 Model B, 2GB RAM** — chosen over the Raspberry Pi Zero 2 W for built-in Gigabit Ethernet (more reliable for an always-on service) and greater RAM/CPU headroom for future expansion into GRC lab projects (vulnerability scanning, log aggregation).
- **Kit:** Vilros Raspberry Pi 4 Complete Starter Kit — board, aluminum alloy case with built-in fan, USB-C 5V/3A power supply with on/off switch, 64GB microSD card, SD-to-USB adapter, micro HDMI to HDMI cable, heatsinks.

### Assembly

- Identified the GPIO header pin layout on the board (pin 1 nearest the top corner, alternating odd/even columns).
- Connected the case fan's 2-pin power connector (red/black) to GPIO pins 4 (5V) and 6 (GND), adjacent pins on the header.
- Left the fan's separate blue PWM control wire unconnected initially (fan ran at constant full speed); later connected it to GPIO pin 8 to enable software-based temperature-triggered fan control.
- Seated the board into the aluminum case with heatsinks applied, then closed the case.

## Operating System Setup

- Flashed **Raspberry Pi OS Lite (64-bit)** to the microSD card using Raspberry Pi Imager, chosen over the full desktop image to keep the system lightweight for a headless service.
- Initial setup was done directly (keyboard, mouse, monitor connected) rather than pre-configuring Wi-Fi/SSH in Imager, due to the Pi's physical location being out of reach of an Ethernet run.
- After first boot, enabled SSH via `raspi-config` (Interface Options → SSH → Yes) to allow remote administration and remove the need for the monitor/keyboard going forward.

## Networking Configuration

### Static IP (DHCP Reservation)

A Pi-hole server needs a static IP so dependent devices don't lose their DNS server after a lease renewal. Configured via router-side DHCP reservation rather than a static IP on the Pi itself:

- Retrieved the Pi's MAC address and current IP:
  ```
  ip a
  ```
- Logged into the router admin page and located the DHCP client list (Network Settings → IPv4 Address Distribution).
- Reserved a fixed IP address (`<pi-static-ip>`, an address within the local 192.168.1.x range) to the Pi's MAC address, changing its lease type from Dynamic to Static.

### SSH Remote Access

```
ssh pi@<pi-static-ip>
```

Verified working, then physically disconnected the keyboard, mouse, and HDMI cable, transitioning the Pi to fully headless operation.

## Pi-hole Deployment

- Updated the OS before installing anything new:
  ```
  sudo apt update && sudo apt full-upgrade -y
  ```
- Installed Pi-hole using the official one-line installer:
  ```
  curl -sSL https://install.pi-hole.net | bash
  ```
- During the interactive install: selected the `wlan0` interface, chose Cloudflare as the upstream DNS provider, accepted default blocklists, enabled both IPv4/IPv6, and installed the web admin interface and lighttpd web server.
- Confirmed the dashboard was reachable at `http://<pi-static-ip>/admin` and ran a Gravity update to refresh blocklists.
- Set the laptop's network adapter to use `<pi-static-ip>` as its preferred DNS server (Windows: Settings → Network & Internet → Advanced network settings → Wi-Fi adapter → DNS server assignment → Manual) to begin filtering DNS traffic for that device.
- Confirmed functionality via the Pi-hole dashboard, which showed live query counts and a blocked-query percentage once the laptop's DNS was pointed at the Pi.

**Current scope:** DNS filtering is active for the laptop used during setup. Extending coverage to the full network (router-wide DNS) is a planned next step.

## Fan Control Configuration

To enable temperature-triggered fan speed control using the PWM wire connected to GPIO pin 8, added a device tree overlay:

```
sudo nano /boot/firmware/config.txt
```

```
dtoverlay=gpio-fan,gpiopin=14,temp=55000
```

> **Note:** This line references GPIO 14 for the fan overlay — see [Troubleshooting](#troubleshooting-log) below regarding a possible interaction between this pin assignment and system stability, identified as an open question during the incident described.

## Troubleshooting Log

This section documents two real issues encountered during the project and the diagnostic steps used to resolve them.

### Issue 1: Undervoltage causing Wi-Fi failure after reboot

**Symptom:** After a routine reboot (to apply the fan overlay change above), the Pi became completely unreachable via ping and SSH for over an hour.

**Diagnosis:**
- Confirmed the Pi was not responding on the network:
  ```
  ping <pi-static-ip>
  ```
- Reconnected the monitor/keyboard directly to view boot output. Found two relevant log lines:
  ```
  My IP address is 127.0.1.1
  hwmon hwmon2: Undervoltage detected!
  ```
- The `127.0.1.1` loopback address confirmed Wi-Fi never successfully connected on that boot. The undervoltage warning pointed to a power delivery problem.
- Confirmed the power issue directly:
  ```
  vcgencmd get_throttled
  ```
  Returned `throttled=0x50000`, confirming both an undervoltage event and throttling had occurred during that boot session.

**Resolution:**
- Moved the power supply from its original outlet/power strip to a different wall outlet and rebooted.
- Re-checked throttling status — returned `throttled=0x0`, confirming the outlet (not the power supply itself) was the root cause.

### Issue 2: Wi-Fi connection not persisting across reboots

**Symptom:** Even after resolving the power issue, `wlan0` showed no IP address and the interface was down.

**Diagnosis:**
- Confirmed the wireless driver and firmware had loaded correctly with no errors:
  ```
  sudo dmesg | grep -i brcm
  ```
- Ruled out a radio soft/hard block:
  ```
  sudo rfkill list
  ```
- Brought the interface up manually and checked connection state:
  ```
  sudo ip link set wlan0 up
  sudo wpa_cli status
  ```
  Result: `state=DISCONNECTED` — the interface and driver were healthy, but no connection attempt was being made.
- Checked for a legacy `wpa_supplicant` configuration file:
  ```
  sudo cat /etc/wpa_supplicant/wpa_supplicant.conf
  ```
  Result: file did not exist. This Raspberry Pi OS release (Debian 13 / Bookworm-based) manages Wi-Fi through NetworkManager rather than a static `wpa_supplicant` config file.
- Checked NetworkManager directly:
  ```
  nmcli device status
  nmcli connection show
  ```
  Result: no saved Wi-Fi connection profile existed at all — only loopback and ethernet connections were listed. The home network had never been saved as a proper NetworkManager profile, which explained why it worked on first boot (via a one-time Imager-provided configuration) but could not reconnect automatically afterward.

**Resolution:**
- Created a proper, persistent NetworkManager Wi-Fi connection:
  ```
  sudo nmcli device wifi connect "<SSID>" password "<password>"
  ```
- Verified the fix:
  ```
  nmcli device status
  ip a show wlan0
  ssh pi@<pi-static-ip>
  ```
  All three confirmed a working, persistent connection. Unlike the initial setup, this connection is now saved as a managed NetworkManager profile and is expected to survive future reboots.

## Regularly Used Commands (Reference)

**Connecting**
```
ssh pi@<pi-static-ip>
```

**System health**
```
vcgencmd measure_temp
vcgencmd get_throttled
ip a
```

**Network troubleshooting**
```
nmcli device status
nmcli connection show
sudo nmcli device wifi connect "<SSID>" password "<password>"
```

**Shutdown / reboot**
```
sudo shutdown -h now
sudo reboot
```

**System updates**
```
sudo apt update && sudo apt full-upgrade -y
```

**Pi-hole**
```
pihole -up
pihole -g
sudo pihole -a -p
```
Dashboard: `http://<pi-static-ip>/admin`

## Next Steps

- [ ] Extend Pi-hole DNS filtering network-wide (router-level DNS configuration or per-device rollout)
- [ ] Deploy WireGuard via PiVPN for secure remote access to the home network and DNS filtering while away from home
- [ ] Confirm the `gpio-fan` overlay is not contributing to boot/network instability; consider testing with the overlay temporarily disabled
- [ ] Longer-term: build out additional GRC-oriented lab components (asset inventory mapped to CIS Controls, vulnerability management cycle, access control documentation) on the same device
