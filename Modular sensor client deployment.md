# Modular Sensor: Client Deployment Cheat Sheet

Quick reference for deploying a Stellar Cyber Modular Sensor on a client's PC or mini PC.
Ubuntu Desktop on the hardware, VirtualBox on top, the sensor VM inside. **No nested layer.**

```text
Client PC / mini PC
  └── Ubuntu 22.04 Desktop
        └── VirtualBox
              └── Modular Sensor VM
```

> [!WARNING]
> Keep this file as a blank template. Don't commit real client IPs, tenant IDs, CM addresses, tokens or passwords to GitHub.

---

## 0. Before going on site

Fill in (on paper or in the company's own system, not here):

| Item | Value |
|---|---|
| Client / tenant name | |
| Tenant ID | |
| CM address | |
| Ubuntu host IP / mask | |
| Sensor VM IP / mask | |
| Gateway | |
| DNS | |
| Switch port for mirrored traffic (SPAN) | |
| Sensor sizing (CPU / RAM / disk) | |

Bring / have ready:

- [ ] Ubuntu 22.04 Desktop installer USB
- [ ] Sensor OVA (VMware image from the platform) and its SHA-1 hash file
- [ ] Two network cables (management + mirror), one NIC each
- [ ] Firewall ports requested from the client and confirmed open (see the table below)
- [ ] Two free static IPs from the client (host and sensor), outside their DHCP range
- [ ] Sizing agreed with team lead (my lab sensor was 2 CPU / 4 GB, a real one is bigger, check `show module`)

### Firewall ports to request from the client

Request these ports to be open on the client's firewall:

| Port(s) | Requested | Confirmed open |
|---|---|---|
| 8888 / 8889 | [ ] | [ ] |
| 33443 | [ ] | [ ] |
| 8443 | [ ] | [ ] |
| 6640-6648 | [ ] | [ ] |

Port 6644 (the one `show cm` reports) is inside the 6640-6648 range.

---

## 1. BIOS

1. Enable **Intel VT-x / Virtualization** (AMD: SVM).
2. Set **Restore on AC Power Loss** to **Power On** (so it comes back after a power cut).
3. Secure Boot: turn it off, or be ready to enroll a key later (see step 5).
4. Boot from the USB.

---

## 2. Install Ubuntu Desktop 22.04

1. **Install Ubuntu > Normal installation > Erase disk and install Ubuntu**.
2. Leave the third-party software box **unticked**.
3. Create the admin user, set the hostname (for example `<client>-sensorhost`).
4. Reboot and remove the USB.

---

## 3. Set the static IP on the Ubuntu host

**GUI:** Settings > Network > gear on the **management** NIC > IPv4 > **Manual**

| Field | Value |
|---|---|
| Address | `<HOST_IP>` |
| Netmask | `<NETMASK>` |
| Gateway | `<GATEWAY>` |
| DNS | `<DNS>` |

Then toggle the connection off and on.

**Second (monitor) NIC:** gear > IPv4 > **Disable**. It must have no IP, it only receives mirrored traffic.

**Terminal alternative:**

```bash
ip -br link
nmcli con show
sudo nmcli con mod "<connection name>" ipv4.method manual \
  ipv4.addresses <HOST_IP>/<PREFIX> ipv4.gateway <GATEWAY> ipv4.dns <DNS>
sudo nmcli con up "<connection name>"
```

---

## 4. Commands after the OS is installed

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y net-tools
ip -br addr
ip route
ping -c 3 <GATEWAY>
ping -c 3 8.8.8.8
ping -c 3 google.com
egrep -c '(vmx|svm)' /proc/cpuinfo
```

- Host IP is on the management NIC, default route goes via the gateway, all pings work.
- `egrep` result above 0 means virtualization is on. If it's 0, go back to the BIOS.
- Note the NIC names from `ip -br link` (for example `eno1` = management, `eno2` = monitor). They are needed for the VM adapters.

---

## 5. Install VirtualBox

```bash
sudo apt remove --purge 'virtualbox*'
sudo apt install -y wget gnupg
wget -O- https://www.virtualbox.org/download/oracle_vbox_2016.asc | \
  sudo gpg --dearmor --yes --output /usr/share/keyrings/oracle-virtualbox-2016.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/oracle-virtualbox-2016.gpg] \
https://download.virtualbox.org/virtualbox/debian jammy contrib" | \
  sudo tee /etc/apt/sources.list.d/virtualbox.list
sudo apt update
sudo apt install -y virtualbox-7.2 build-essential dkms linux-headers-$(uname -r)
sudo usermod -aG vboxusers $USER
sudo reboot
```

After the reboot:

```bash
vboxmanage --version
groups
```

If a blue **MOK / Secure Boot** screen appears on reboot: **Enroll MOK > Continue > enter the password set during install**.
If VirtualBox still complains about its kernel driver, turn Secure Boot off in the BIOS.

---

## 6. Get the sensor image onto the PC

1. Platform: **System > Data Source Management > Sensors > Sensor Installation > Download Images** (VMware type).
2. Copy the OVA and hash file to the PC (USB or `scp`).
3. Verify:

```bash
sha1sum aella-modular-ds-*.ova
```

It must match the hash file. If not, download again.

---

## 7. Import the OVA into VirtualBox

1. **File > Import Appliance**, choose the `.ova`.
2. Settings:
   - CPU and RAM: the agreed sizing (not the lab sizing)
   - Virtual Disk Image format: **VDI**
   - MAC Address Policy: **Generate new MAC addresses for all network adapters**
   - VM name: something clear, for example `sensor-<client>`
3. **Import**.

---

## 8. VM network and display settings

Select the VM > **Settings**:

| Setting | Value |
|---|---|
| Network > Adapter 1 | Bridged Adapter on the **management NIC** (`eno1`), Intel PRO/1000 MT Desktop, Promiscuous Mode **Allow All** |
| Network > Adapter 2 | Bridged Adapter on the **monitor NIC** (`eno2`), Intel PRO/1000 MT Desktop, Promiscuous Mode **Allow All** |
| Display > Graphics Controller | **VMSVGA** (clears the warning) |

On real hardware leave the Acceleration settings at default.

---

## 9. Configure the sensor

1. Start the VM and open its console.
2. Log in as `aella`. The first login forces a password change: set a new one and store it in the company's password manager.
3. At the `Sensor>` prompt:

```text
set hostname <client>-sensor
set interface management ip <SENSOR_IP>/<NETMASK>
set interface management gateway <GATEWAY>
set interface management dns <DNS>
set tenant_id <TENANT_ID>
set cm <CM_ADDRESS>
show interface
show cm
```

Don't skip `set tenant_id`.

- `show interface` shows the sensor IP on `management`.
- `show cm` must show **Status: ESTABLISHED**.

Other useful commands:

```text
show module
show module request
```

---

## 10. Platform side

1. **Sensors** page: wait a few minutes for the sensor to appear.
2. **Authorize** it.
3. Check the **tenant** is the client's.
4. Assign the **Modular Sensor Profile** for the client (ask team lead which one).
5. Status turns **green**. Open the sensor details and check the services are green.
6. After the SPAN port is connected: the **Input** bytes on the Sensors page should start going up.

---

## 11. Make it survive a reboot or power cut

1. Ubuntu: **Settings > Users > Automatic Login** on (so the session starts without anyone logging in).
2. **Startup Applications** > add:

```bash
VBoxManage startvm "<VM name>" --type headless
```

3. Reboot the PC and confirm the sensor comes back and `show cm` is ESTABLISHED. (Check with team lead if the company has a preferred way to do auto-start.)

---

## 12. Before leaving the site

- [ ] `show cm` is ESTABLISHED
- [ ] Sensor is green and authorized under the correct tenant
- [ ] Mirrored traffic is arriving (Input bytes rising)
- [ ] Reboot test passed
- [ ] Host IP, sensor IP, VM name and BIOS settings are recorded (in the company's system)
- [ ] No tokens, passwords or client details left on the PC, USB or in screenshots

---

## Quick troubleshooting

| Problem | What to check |
|---|---|
| Ubuntu has no internet | Cable on the right NIC, manual IP, gateway and DNS, then ping the gateway first |
| VirtualBox: "VT-x is not available" | Enable virtualization in the BIOS |
| VirtualBox kernel driver / module error | Secure Boot: enroll the MOK key or turn Secure Boot off |
| `show cm` not ESTABLISHED | CM address, tenant ID, DNS, firewall ports open (see section 0), sensor IP not clashing with another device |
| Sensor doesn't appear in the platform | Same checks as above, plus the sensor's date and time |
| No traffic (Input stays at 0) | SPAN port config on the switch, cable on the monitor NIC, VM adapter 2 on the right NIC with Promiscuous Allow All |
| Some services red | Profile resources too small: `show module`, then raise CPU/RAM |
| Wrong tenant | Run `set tenant_id <ID>` again and recheck in the platform |