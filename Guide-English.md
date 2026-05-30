# Replacing the Orange Spain Livebox 6 with a UFiber Loco and Unifi USG-3P

## Why Replace the Livebox?

The Orange Spain Livebox 6 is a locked-down, ISP-controlled router. You cannot configure VLANs, set custom firewall rules, run network-wide ad blocking, or manage your network as a unified system. Replacing it gives you full control over your network with proper firewall rules, VLAN segmentation, custom DNS, and a single management plane for your network.

At the time of writing, the USG-3P is End-of-Life. However, it remains affordable and easily accessible on Wallapop and from other used hardware resellers. Particularly when combined with a self hosted controller on a Raspberry Pi or other mini PC, it is a low-cost, low barrier to entry method of replacing existing 

**Important:** This guide was written while using a Unifi Controller running 6.0.28, due to use of EoL hardware on the author's network. If you are running a more up-to-date version, you will need to (a) ensure that your controller supports your intended Gateway, and (b) check the equivalent settings locations in newer controller versions, as this guide will make reference to locations in 6.0.28. 

### Why Not Use ONT Mode?

The Orange Livebox has a 'Modem Only' mode, which in theory, allows you to treat it as a simple ONT, and use a commercial or opnsense router for actual routing (including the USG-3P), and skip the Nano Fiber Loco / Nano G completely. 

In my experience, this did not work under any circusmtances or hardware combinations. Your mileage may vary, and if you already have an existing router, you may wish to try that first, as this guide is rather involved.

## Requirements

As a minimum, you will need the following (or their equivalent) to follow this guide:

| Item                                                         | Purpose                                                   | Notes                                                        |
| ------------------------------------------------------------ | --------------------------------------------------------- | ------------------------------------------------------------ |
| **Ubiquiti UFiber Loco** (UF-LOCO) or **UFiber Nano G** (UF-NANO) | GPON ONT — replaces the Livebox's fibre modem function    | The Loco is cheaper and sufficient. The Nano G has built-in WiFi. Both should work identically for this purpose. |
| **Ubiquiti USG-3P** (UniFi Security Gateway)                 | Router/firewall — replaces the Livebox's routing function | EOL as of November 2024, but still works with controller versions up to 8.x. See [EOL Notes](#eol-notes). |
| **UniFi Controller**                                         | Software to manage the USG (and optionally APs/switches)  | Self-hosted via Docker recommended. See [Controller Setup](#controller-setup). |
| **A computer with Python 3 and SSH**                         | For serial number cloning and configuration               | Windows, macOS, or Linux.                                    |
| **An ethernet cable**                                        | Direct connection to the Loco during setup                | You'll need to disconnect from your network temporarily.     |
| **Your Orange Spain contract credentials**                   | PLOAM password and serial number from the Livebox         | Extracted in [Step 1](#step-1-extract-credentials-from-the-livebox). |

### Optional but strongly recommended

- **A cheap unmanaged gigabit switch** — useful if you need to keep local connectivity between devices during migration.
- **A phone with mobile data** — useful for creating a mobile hotspot or checking references during stages where the internet is not available.

## Architecture Overview

```
┌─────────────────┐     SC/APC fibre      ┌──────────────┐
│  Orange Spain   │◄────────────────────► │  UFiber Loco │
│  Rosetta.       │     GPON              │  (ONT/modem) │
│                 │                       └──────┬───────┘
└─────────────────┘                          Ethernet
                                                │
                                          ┌─────┴──────┐
                                          │  USG-3P    │
                                          │  WAN: eth0 │◄── VLAN 832 (DHCP)
                                          │  LAN: eth1 │
                                          └─────┬──────┘
                                                │
                                          ┌─────┴──────┐
                                          │  Switch    │
                                          │  + APs     │
                                          │  + Devices │
                                          └────────────┘
```

The Loco handles GPON authentication with Orange's OLT and passes raw ethernet to the USG. The USG tags traffic on VLAN 832, gets a public IP via DHCP, and routes your LAN. The UniFi controller manages the USG configuration.

## Critical: Orange Spain ≠ Orange France

Many guides online (especially in French) describe replacing Orange routers using **DHCP Option 90** authentication, **vendor-class-identifier "sagem"**, **user-class strings**, and **PPPoE**. **None of this applies to Orange Spain.**

Orange Spain uses (at the time of writing of this guide, May 2026):

- **VLAN 832** with standard **DHCP** (no PPPoE)
- **No Option 90**
- **No vendor-class-identifier**
- **No user-class**
- **No RFC3118 authentication strings**

If you see any guide telling you to generate a hex string from `fti/xxxxxxxxx` credentials or set vendor-class to "sagem", that guide is for **Orange France** and will **actively prevent** your connection from working in Spain.

The only authentication that matters is at the **GPON layer**: the PLOAM password and the ONT serial number.

------

## Step 1: Extract Credentials from the Livebox

Before disconnecting anything, log into your Livebox at `http://192.168.1.1` and note down these values:

### 1.1 PLOAM Password (ONT Password)

Navigate to the information/GPON section of the Livebox admin panel. Look for the **PLOAM password** or **ONT password**. It will be a 10-character alphanumeric string (e.g., `A5XK92WR7D`).

### 1.2 Serial Number

Find the **GPON serial number**. It will look like `SMBS4A8F7C12` — a 4-character vendor ID followed by 8 hex characters. Write down:

- **Full serial**: e.g., `SMBS4A8F7C12`
- **Vendor ID**: the first 4 characters (e.g., `SMBS`)
- **Device serial**: the remaining 8 characters (e.g., `4A8F7C12`)

### 1.3 MAC Address

Find the **WAN MAC address** of the Livebox (not the WiFi MAC or LAN MAC). Format: `XX:XX:XX:XX:XX:XX`.

### 1.4 Summary Table

Save these somewhere accessible offline — you'll need them when you're disconnected from the internet:

| Parameter                | Your Value      | Example             |
| ------------------------ | --------------- | ------------------- |
| PLOAM Password           |                 | `A5XK92WR7D`        |
| Full Serial              |                 | `SMBS4A8F7C12`      |
| Vendor ID                |                 | `SMBS`              |
| Device Serial            |                 | `4A8F7C12`          |
| WAN MAC Address          |                 | `DC:15:C8:1A:E3:4B` |
| Loco Default IP          | `192.168.1.1`   | —                   |
| Loco Default Credentials | `ubnt` / `ubnt` | —                   |

------

## Step 2: Prepare the UFiber Loco

### 2.1 Initial Access

1. Disconnect your PC from your network.
2. Connect your PC's ethernet port directly to the Loco's **LAN port**.
3. Power the Loco via its 24V PoE adapter or a micro-USB power source rated at **5V/1A or higher**. Do not rely on a PC USB 2.0 port — it may not deliver enough current (the Loco draws up to 3.5W = 700mA at 5V).
4. Set your PC to a static IP: `192.168.1.100`, subnet `255.255.255.0`, gateway `192.168.1.1`.
5. Browse to `https://192.168.1.1` (note: **HTTPS**, not HTTP — the Loco redirects to HTTPS and some browsers will refuse the redirect silently). Accept the certificate warning.
6. Login: `ubnt` / `ubnt`.

> **Common pitfall:** If the browser shows "unable to connect", try both `http://` and `https://`. If neither works, verify your static IP is actually `192.168.1.100` and not, e.g., `192.168.100.1` (an easy transposition error in Windows). Run `ipconfig` to confirm.

> **Hyper-V / WSL users:** If you have Hyper-V or WSL enabled on Windows, a virtual network adapter may intercept traffic from your physical ethernet port. Disable the vEthernet adapter temporarily or use a different machine.

### 2.2 Update Firmware

Go to **System** → **Upload Firmware** and update to the latest version. At time of writing, firmware **4.4.6** works. The device will reboot (wait 3-5 minutes). Firmware copies from Ui.com are available in the \Firmware folder of this repository. All firmware is unmodified, and is copyright Ubiquiti.

### 2.3 Configure GPON Settings

1. Go to the **GPON** tab.
2. Set **OLT Profile**: try **Profile 3** (FiberHome) first. See [Profile Selection](#profile-selection) for details.
3. Set **PLOAM Password**: your password from Step 1 (e.g., `A5XK92WR7D`).
4. **Hex Format**: **OFF** (unchecked).
5. Leave LOID Authentication fields at defaults (`ubnt` / `ubntubnt`) — Orange Spain doesn't use LOID.
6. Click **Save**.

### 2.4 Change Management IP

**This is critical.** The USG will also use `192.168.1.1` as its LAN IP. The Loco must be on a different address.

1. Go to the **Network** tab.
2. Change the device IP to something outside your DHCP range, e.g., `192.168.1.15` or `192.168.100.1`.
3. Save. You'll need to reconnect at the new IP.

### 2.5 Enable SSH

Go to **System** and ensure **SSH** is enabled on port 22. You'll need this for serial cloning.

------

## Step 3: Clone the Serial Number (Critical)

This is the step most guides miss, and it's the single most common reason for GPON authentication failure. The Loco ships with Ubiquiti's own vendor ID (`UBNT`) and a random serial number. Orange's OLT expects to see your Livebox's serial number and vendor ID.

### 3.1 Install Dependencies

On your PC (while you still have internet, or downloaded in advance):

```bash
pip install paramiko scp
```

### 3.2 Download the Serial Hack Tool

Download from: https://github.com/v-a-c-u-u-m/ufiber_nano_serial_hack

Clone the repo or download the ZIP and extract it.

### 3.3 Convert Your Serial to Hex Format

The GPON serial is 8 bytes: 4 bytes vendor ID (ASCII) + 4 bytes device serial (hex).

For example, `SMBS4A8F7C12`:

- `SMBS` → ASCII hex: `53:4D:42:53`
- `4A8F7C12` → hex bytes: `4A:8F:7C:12`
- **Full serial parameter**: `53:4D:42:53:4A:8F:7C:12`

To convert your vendor ID to hex, use an ASCII table or an online tool:

| Char | Hex  | Char | Hex  | Char | Hex  | Char | Hex  |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| A    | 41   | H    | 48   | O    | 4F   | V    | 56   |
| B    | 42   | I    | 49   | P    | 50   | W    | 57   |
| C    | 43   | J    | 4A   | Q    | 51   | X    | 58   |
| D    | 44   | K    | 4B   | R    | 52   | Y    | 59   |
| E    | 45   | L    | 4C   | S    | 53   | Z    | 5A   |
| F    | 46   | M    | 4D   | T    | 54   |      |      |
| G    | 47   | N    | 4E   | U    | 55   |      |      |

The device serial part (the 8 characters after the vendor ID) should be treated as **hex bytes** — each pair of characters is one byte.

### 3.4 Verify Before Writing (Read-Only)

Connect your PC to the Loco via ethernet (static IP on same subnet), then (using whatever IP you set in Step 2.4):

```bash
python ubi_serial_hack.py -r 192.168.1.15 -p 22 --readonly --insecure
```

Password: `ubnt` (or whatever you set).

You should see output like:

```
[~] vendor_id:        "UBNT"
[~] serial_id:        "a9e0dd3c"
[~] base_mac_addr:    ac:8b:a9:e0:dd:3c
[~] gpon_password:    "A5XK92WR7D"
```

Confirm the GPON password matches. The vendor ID and serial are Ubiquiti defaults — we're about to change them.

### 3.5 Write the Cloned Serial

```bash
python ubi_serial_hack.py -r 192.168.1.15 -p 22 \
  --serial 53:4D:42:53:4A:8F:7C:12 \
  --mac DC:15:C8:1A:E3:4B \
  --insecure
```

Replace the serial and MAC with **your** values from Step 1.

> **The `--insecure` flag is required for the UFiber Loco.** The script will complain about firmware checksum differences on Loco devices without it. This is normal and safe.

### 3.6 Verify the Write

Run the read-only command again and confirm all values match your Livebox:

```bash
python ubi_serial_hack.py -r 192.168.1.15 -p 22 --readonly --insecure
```

Expected output:

```
[~] vendor_id:        "SMBS"
[~] serial_id:        "4A8F7C12"
[~] base_mac_addr:    dc:15:c8:1a:e3:4b
[~] gpon_password:    "A5XK92WR7D"
```

------

## Step 4: Test GPON Authentication

1. Disconnect the ethernet cable from the Loco.
2. Unplug the SC/APC fibre cable from the Livebox. Handle it gently — don't touch the ferrule.
3. Plug the fibre into the Loco's **GPON port**.
4. Watch the LEDs. Wait up to 60 seconds.

### LED Indicators

- **Green bars** = GPON authenticated successfully 
- **Orange/amber flashing** = Attempting registration
- **White/blue solid bars** = Powered on, no GPON authentication

> **Important:** The LEDs can be misleading on some firmware versions. The Loco may show only white bars even when GPON is authenticated. **The dashboard is authoritative.** To check properly: reconnect ethernet to the Loco, browse to `https://192.168.1.15`, and look at the dashboard. If it shows **STATE: CONNECTED** with RX power around **-15 to -20 dBm**, authentication has succeeded regardless of what the LEDs show. My own Fiber Loco, for example, never shows green bars, despite working perfectly.

### Profile Selection

If Profile 3 doesn't authenticate, cycle through profiles:

| Profile   | OLT Manufacturer | Notes                                                        |
| --------- | ---------------- | ------------------------------------------------------------ |
| Profile 1 | Ubiquiti OLT     | **Skip this** — it's for Ubiquiti's own OLT equipment. It doesn't expose PLOAM or third-party authentication settings. |
| Profile 2 | Huawei           | Common for Orange Spain                                      |
| Profile 3 | FiberHome        | Common for Orange Spain — try this first                     |
| Profile 4 | ZTE              | Less common but works in some areas                          |

To change profiles: go to `https://192.168.1.15` → GPON tab → change the OLT Profile dropdown → Save. The device reboots. Reconnect the fibre and check again.

In our experience, **Profiles 2, 3, and 4 all worked** — the OLT was flexible. Your mileage may vary depending on your central office equipment. If none of Profiles 2/3/4 produce a CONNECTED state, your OLT may be Alcatel/Nokia, which the UFiber devices cannot authenticate against. In that case, you would need a different ONT (e.g., a Huawei HG8010H with serial cloning).

### Rollback

If nothing works: unplug the fibre from the Loco, reconnect it to the Livebox, and power the Livebox on. You're back to normal in 2 minutes. Nothing has been changed on the Orange side.

------

## Step 5: Configure the USG-3P

### 5.1 Physical Wiring

```
Fibre → Loco GPON port → Loco LAN port → USG WAN (eth0)
                                           USG LAN (eth1) → Switch → Devices
```

### 5.2 The config.gateway.json File

The USG needs to create a VLAN 832 sub-interface on its WAN port and request a DHCP lease over it. It also needs a NAT masquerade rule on that VLAN interface (in my experience, not on plain `eth0`), and — critically — firewall and routing fixes that the UniFi controller does not apply correctly by default.

Create this file on your UniFi controller:

```json
{
    "firewall": {
        "source-validation": "loose"
    },
    "interfaces": {
        "ethernet": {
            "eth0": {
                "vif": {
                    "832": {
                        "address": ["dhcp"],
                        "description": "Orange Spain Internet",
                        "dhcp-options": {
                            "default-route": "update",
                            "default-route-distance": "1",
                            "name-server": "update"
                        },
                        "firewall": {
                            "in": {
                                "name": "WAN_IN"
                            },
                            "local": {
                                "name": "WAN_LOCAL"
                            }
                        }
                    }
                }
            }
        }
    },
    "service": {
        "nat": {
            "rule": {
                "5010": {
                    "outbound-interface": "eth0.832",
                    "type": "masquerade"
                }
            }
        }
    }
}
```

This file does four things:

1. **VLAN 832 with DHCP** — creates the `eth0.832` sub-interface and requests a public IP over it.
2. **NAT masquerade on `eth0.832`** — without this, the USG gets a public IP but your LAN traffic won't be NATted through it. The USG's default masquerade rules point at `eth0`, not `eth0.832`.
3. **Firewall rules on `eth0.832`** — this is the critical one that most guides miss. See [The VLAN 832 Firewall Gap](#the-vlan-832-firewall-gap) below.
4. **Source validation set to "loose"** — fixes reverse-path filtering that causes legitimate return traffic to be dropped as "martian". See [Reverse-Path Filtering](#reverse-path-filtering) below.

### The VLAN 832 Firewall Gap

When the UniFi controller creates WAN_LOCAL and WAN_IN firewall rules (either its defaults or ones you add in the UI), it applies them to `eth0` — the physical WAN interface. But your actual internet-facing interface is `eth0.832`, the VLAN sub-interface. **The controller does not automatically apply firewall rules to VLAN sub-interfaces.** This means that without the `firewall` block in the config above, your USG's WAN port is effectively unfiltered: every port scan, SSH brute-force attempt, and probe from the internet hits the USG's services directly.

On a device with a Cavium Octeon MIPS64 CPU and 512MB of RAM, this constant bombardment (which is normal — any public IP gets scanned continuously by automated bots worldwide) can cause the USG to become unresponsive within hours. The symptoms are intermittent internet loss that requires a power cycle to fix, with no clear error in the logs.

The `firewall.in` and `firewall.local` entries in the config above tell EdgeOS to apply your WAN_IN and WAN_LOCAL chains to `eth0.832`, so all traffic arriving on the Orange VLAN is subject to the same firewall rules as traffic on the physical interface. This is essential for stability.

### Reverse-Path Filtering

The `"source-validation": "loose"` setting changes the kernel's reverse-path filter (`rp_filter`) from strict mode (1) to loose mode (2). In strict mode, the kernel drops any packet whose source address would not be routed back out the same interface it arrived on. With VLAN 832, legitimate return traffic from the internet can arrive on `eth0.832` but the kernel's routing table may say the source should be reachable via `eth0` — the parent interface. Strict mode drops this as a "martian" packet and logs it.

The symptom is intermittent connectivity: some traffic works, some is silently dropped, and the logs fill with lines like:

```
IPv4: martian source 192.168.1.x from <external-ip>, on dev eth0.832
```

Loose mode still validates that the source address is reachable via *some* interface (it's not disabled), but doesn't require it to be the same one the packet arrived on. This is the correct setting for any VLAN-based WAN configuration.

### 5.3 Where to Place the File

**If your controller is Docker-based (self-hosted):**

Find your UniFi data volume mount. Common paths:

```bash
# Check your docker-compose.yml for the volume mapping
# If: volumes: - ./config:/config
# Then the file goes at:
/opt/docker/unifi/config/data/sites/default/config.gateway.json
```

Create the `sites/default/` directory if it doesn't exist:

```bash
mkdir -p /path/to/unifi/config/data/sites/default/
```

> **The site name "default"** corresponds to the internal site ID, not the display name. Check your controller URL — it shows `manage/site/XXXXX/` where `XXXXX` is the site ID. Use that value instead of `default` if it's different.

**If your controller is running natively (not Docker):**

```bash
/usr/lib/unifi/data/sites/default/config.gateway.json
```

### 5.4 Quick Test via SSH (Optional, Before Controller Adoption)

If you want to test internet before dealing with controller adoption, you can configure the USG directly via SSH:

```bash
ssh ubnt@192.168.1.1
# Password: ubnt (factory default)

configure
set interfaces ethernet eth0 vif 832 address dhcp
set interfaces ethernet eth0 vif 832 description "Orange Spain Internet"
set interfaces ethernet eth0 vif 832 dhcp-options default-route update
set interfaces ethernet eth0 vif 832 dhcp-options name-server update
set interfaces ethernet eth0 vif 832 firewall in name WAN_IN
set interfaces ethernet eth0 vif 832 firewall local name WAN_LOCAL
set firewall source-validation loose
set service nat rule 5010 outbound-interface eth0.832
set service nat rule 5010 type masquerade
commit
save
```

Check for a public IP:

```bash
show interfaces ethernet eth0 vif 832
```

If you see a public IP and `ping 8.8.8.8` works from your desktop — the Livebox is successfully replaced.

> **SSH configuration is overwritten when the controller provisions the USG.** This is fine for testing, but you need the `config.gateway.json` in place before controller adoption to make the config persistent.

------

## Step 6: Controller Adoption

### 6.1 Docker Networking — Host Mode Required

**This is the most common adoption failure when running the UniFi controller in Docker.**

The UniFi controller needs to SSH *into* the USG to provision it. If the controller container is on a Docker bridge network (the default), it cannot route to `192.168.1.1`. The inform process (USG → controller) works because Docker maps the port, but provisioning (controller → USG via SSH) fails.

**Symptom:** The USG appears in the controller, you click Adopt, it enters an "Adopting" loop and never completes, or shows "Disconnected" repeatedly.

**Fix:** Switch the controller container to host networking.

Before (bridge mode — **broken for USG adoption**):

```yaml
services:
  unifi-controller:
    image: linuxserver/unifi-controller:version-6.0.28
    container_name: unifi-controller
    restart: unless-stopped
    ports:
      - "3478:3478/udp"
      - "10001:10001/udp"
      - "8080:8080"
      - "8443:8443"
      - "8880:8880"
      - "6789:6789"
      - "5514:5514/udp"
    volumes:
      - ./config:/config
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
```

After (host mode — **works**):

```yaml
services:
  unifi-controller:
    image: linuxserver/unifi-controller:version-6.0.28
    container_name: unifi-controller
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./config:/config
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
```

Remove the entire `ports` block — with host networking, the container binds directly to the host's ports. Verify no other services on the host conflict with UniFi's ports (8080, 8443, 3478/udp, 10001/udp, 8880, 6789, 5514/udp).

**Important:** The rest of the yaml settings are examples from my own deployment to help provide context. You should use the correct values for your own docker-compose.yml. If you don't know how to set those, I strongly suggest reading up on Docker before attempting this migration (https://docs.docker.com/get-started/). You may also wish to consider using the Unifi Controller on bare metal rather than Docker, but remember to set the version to whatever you need to support your own hardware configuration.

Then:

```bash
docker compose down
docker compose up -d
```

### 6.2 Adoption Process

1. Ensure the `config.gateway.json` is in place at `sites/default/`.

2. Open and login to the controller UI at `https://<controller-ip>:8443`.

3. The USG should appear as "Pending Adoption".

4. Click **Adopt**.

5. If it doesn't appear, SSH into the USG and force the inform:

   ```bash
   ssh ubnt@192.168.1.1
   set-inform http://<controller-ip>:8080/inform
   ```

   You may need to run `set-inform` twice — once to introduce the device, and again after clicking Adopt.

### 6.3 Adoption Troubleshooting

**USG stuck in adoption loop:**

If the USG repeatedly cycles between "Adopting" and "Disconnected":

1. Check the Docker networking (see 6.1 above).

2. Force-remove the device from the controller database:

   ```bash
   docker exec -it unifi-controller mongo --port 27117
   ```

   ```javascript
   use ace
   db.device.remove({"model": "UGW3"})
   ```

   ```bash
   docker restart unifi-controller
   ```

3. Factory reset the USG (hold reset 10 seconds), then re-adopt.

**Config not applied after adoption:**

If the USG adopts but doesn't have VLAN 832:

1. Verify the file is visible inside the container:

   ```bash
   docker exec -it unifi-controller cat /usr/lib/unifi/data/sites/default/config.gateway.json
   ```

2. Force re-provision by resetting the config version:

   ```bash
   docker exec -it unifi-controller mongo --port 27117 --eval \
     'db.getSiblingDB("ace").device.update({"model":"UGW3"},{$set:{"cfgversion":"0"}})'
   ```

3. Restart the controller: `docker restart unifi-controller`

------

## Step 7: Verification

Once everything is working:

```bash
# From the USG SSH:
show interfaces ethernet eth0 vif 832
# Should show a public IP address

# From any LAN device:
ping 8.8.8.8
ping google.com
```

### Verify the Firewall and rp_filter Fixes

These are just as important as the basic connectivity check. After the controller provisions the USG, SSH in and confirm:

```bash
# Check rp_filter is set to loose (should return: 2)
cat /proc/sys/net/ipv4/conf/eth0.832/rp_filter

# Check firewall is applied to eth0.832 (should show eth0.832 → WAN_LOCAL)
sudo iptables -L VYATTA_FW_LOCAL_HOOK -nv --line-numbers
```

In the `VYATTA_FW_LOCAL_HOOK` output, you should see a line mapping `eth0.832` to `WAN_LOCAL`. If it's missing, the firewall section of your `config.gateway.json` isn't being applied — check the file path and force re-provision.

### Suggested Post-Migration Checklist

- [ ] Loco GPON dashboard shows **CONNECTED** with good RX power (-15 to -20 dBm)
- [ ] USG has a public IP on `eth0.832`
- [ ] LAN devices can reach the internet
- [ ] DNS resolution works (`ping google.com`)
- [ ] Controller shows USG as "Connected" and managed
- [ ] `config.gateway.json` is in place (config survives re-provisioning)
- [ ] `rp_filter` on `eth0.832` is set to `2` (loose)
- [ ] `VYATTA_FW_LOCAL_HOOK` shows `eth0.832 → WAN_LOCAL`
- [ ] Access points re-adopted and broadcasting (if applicable)

If all of these work, congratulations! You can now put your Orange Livebox in the back of a cupboard and forget about it until you switch ISP and they ask for it back. 

Enjoy your new ability to properly route VLANs and set your own DNS.

------

## EOL Notes

The **USG-3P** was declared End of Life by Ubiquiti in **November 2024**:

- UniFi Network controller versions **9.x and later cannot adopt USG devices at all**.
- The USG-3P works with controller versions up to **8.x**.
- If you self-host your controller (Docker), you can pin the version and control your own update cycle.

If you've already got newer Ubiquiti / Unifi hardware, consider whether the USG-3P is still worth it for your use case. If you already have one, it will continue working indefinitely as long as you control your controller version.

------

## Controller Setup

If you don't have a UniFi controller yet, the simplest approach for self-hosting is Docker. A minimal `docker-compose.yml`:

```yaml
services:
  unifi-controller:
    image: linuxserver/unifi-controller:latest
    container_name: unifi-controller
    restart: unless-stopped
    network_mode: host  # Required for USG adoption
    volumes:
      - ./config:/config
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Madrid
```

Run with `docker compose up -d`. Access the UI at `https://<host-ip>:8443`.

> **Pin the controller version** if you have a USG-3P. Use a specific tag like `version-8.6.9` instead of `latest` to prevent accidental upgrades that drop USG support.

------

## Troubleshooting Reference

### GPON won't authenticate

| Symptom                               | Likely Cause              | Fix                                                          |
| ------------------------------------- | ------------------------- | ------------------------------------------------------------ |
| White bars, no CONNECTED on dashboard | Serial not cloned         | Run the serial hack tool (Step 3)                            |
| White bars after serial cloning       | Wrong OLT profile         | Try Profiles 2, 3, and 4                                     |
| No RX power on dashboard              | Fibre not seated properly | Reseat the SC/APC connector, check for dust                  |
| All profiles fail                     | Alcatel/Nokia OLT         | UFiber cannot authenticate against Alcatel. Need a different ONT. |

### USG won't get internet

| Symptom                                   | Likely Cause                                       | Fix                                                |
| ----------------------------------------- | -------------------------------------------------- | -------------------------------------------------- |
| No IP on eth0.832                         | VLAN 832 not configured                            | Check `config.gateway.json` or SSH config          |
| Public IP on eth0.832 but no LAN internet | Missing NAT masquerade on `eth0.832`               | Add rule 5010 to config.gateway.json               |
| USG shows default rules on `eth0` only    | Controller provisioned without config.gateway.json | Ensure file is in correct path, force re-provision |

### USG instability (crashes, internet drops requiring power cycle)

| Symptom                                   | Likely Cause                                       | Fix                                                          |
| ----------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| Internet drops after hours, power cycle fixes it | Firewall not applied to `eth0.832` | Add the `firewall` block to the VLAN 832 config. See [The VLAN 832 Firewall Gap](#the-vlan-832-firewall-gap). |
| Logs show SSH brute-force from external IPs | WAN_LOCAL not filtering `eth0.832` | Same fix — the firewall block applies WAN_LOCAL to the VLAN interface. |
| Logs show "martian source" on `eth0.832` | Strict reverse-path filtering | Add `"source-validation": "loose"` to the firewall section of config.gateway.json. See [Reverse-Path Filtering](#reverse-path-filtering). |
| Random lockups in warm weather, no log evidence | Thermal shutdown | See [Thermal Considerations](#thermal-considerations) below. |

### Thermal Considerations

The USG-3P is passively cooled with a small aluminium heatsink. It is prone to thermal lockup, particularly in warm climates or enclosed spaces. There does not appear to be any graceful throttling that I could determine from my testing. When the SoC overheats, the device silently locks up and requires a power cycle. There is no warning in the logs because the failure is instantaneous.

Contributing factors include DPI (Deep Packet Inspection), IDS/IPS, and high connection-table churn — all of which significantly increase CPU load and heat output on the MIPS64 SoC. DPI and IDS/IPS should not be enabled on the USG-3P in any case due to its limited processing power, but they also worsen the thermal situation. If you need this sort of capability, you should consider if the USG-3P is right for you - an opnSense router on a device such as a Lenovo m920q might be a better fit.

If you experience unexplained lockups that correlate with ambient temperature, ensure the USG is in a well-ventilated area with adequate airflow, not inside an enclosed cabinet or stacked with other warm equipment. An active 80mm or 120mm fan directed at the device can make a significant difference. Expect the USG to struggle at sustained ambient temperatures above ~30°C unless actively cooled.

### Controller adoption fails

| Symptom                                 | Likely Cause                   | Fix                                                          |
| --------------------------------------- | ------------------------------ | ------------------------------------------------------------ |
| Adoption loop (Adopting → Disconnected) | Docker bridge networking       | Switch to `network_mode: host`                               |
| USG stuck as "Disconnected"             | Stale device entry             | Remove via MongoDB, factory reset USG                        |
| SSH to USG rejected after adoption      | Controller changed credentials | Use credentials from Settings → Site → Device Authentication |

------

## Credits and References

- [mordor.world — Sustituir Livebox 6 de Orange por UFiber Loco y un router neutro](https://www.mordor.world/2023/04/26/sustituir-livebox-6-de-orange-por-ufiber-loco-y-un-router-neutro/) — Spanish-language guide that was the starting point for this migration.
- [v-a-c-u-u-m/ufiber_nano_serial_hack](https://github.com/v-a-c-u-u-m/ufiber_nano_serial_hack) — Python tool for cloning serial numbers onto UFiber Loco/Nano G devices.
- [Ubiquiti Community Forums](https://community.ui.com/) — Various threads on GPON authentication and UFiber configuration.

------

## License

This guide is provided as-is under the MIT License. Use at your own risk. Disconnecting ISP equipment may violate your service agreement — check your contract.
