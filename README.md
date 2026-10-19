# Xiaomi AX6000 — Unlock SSH, Flash OpenWrt Firmware, Set Up Clash for Internet Access (Proxy/VPN)

This tutorial shows how to unlock SSH on the Xiaomi AX6000 **without needing a second software router**, and then flash OpenWrt and install Clash/ShellClash for proxy access. The guide is fairly long — please watch the video tutorial carefully and follow each step in order.

- **AX6000 SSH unlock video tutorial:** https://youtu.be/cLSEmZ6ufE4
- **ShellClash / internet-access video tutorial:** https://youtu.be/vn5QdyDlTtI

> ⚠️ **Disclaimer:** Flashing firmware and unlocking SSH can permanently brick your router and will likely void your warranty. Proceed at your own risk, and make sure to follow every step exactly as shown.

---

## Prerequisites

Before you start, prepare the following:

- A Windows PC (the steps below use Windows-only tools such as VMware, WinSCP and PuTTY)
- A phone with a mobile hotspot (used to trick the router into connecting to your OpenWrt VM)
- **Tool package (required):** <a href="https://github.com/aews/lyq/releases/download/AX6000/AX6000-SSH.zip" target="_blank">Download the AX6000-SSH tools >></a>
- **PuTTY (SSH client):** <a href="https://github.com/aews/lyq/releases/download/AX6000/putty.zip" target="_blank">Download PuTTY >></a>
- VMware Workstation (installed in Step 3)
- WinSCP (installed in Step 4)

---

## Step 1 — Set Up the Mobile Hotspot

Create a hotspot on your phone with exactly these settings:

| Setting | Value |
|---|---|
| Network name (SSID) | `OpenWrt` |
| Password | `12345678` |
| Band | **2.4 GHz** (must be 2.4 GHz, not 5 GHz) |

Then check the **Wi-Fi channel** the hotspot is using (it is usually channel `1`). You can verify this by connecting another computer or phone to the hotspot and inspecting the channel in the Wi-Fi app/system settings. You will need this channel number in Step 8.

## Step 2 — Flash (Downgrade) the Router Firmware

Flash the Xiaomi AX6000 with the firmware from the tool package to **downgrade** it to a version that is vulnerable to the SSH-unlock method. The exact firmware file and flashing procedure are demonstrated in the video tutorial above — use the MiFi web UI at `http://192.168.31.1` (System → Upgrade) or the Mi Wi-Fi app to perform the flash, and **do not power off the router during the process**.

## Step 3 — Install VMware and OpenWrt

1. Install **VMware Workstation** on your PC.
2. Create a new virtual machine and install **OpenWrt** in it (the OpenWrt image is included in the tool package).
3. Once booted, log in to the OpenWrt web interface:

   | Item | Value |
   |---|---|
   | Address | `192.168.5.1` |
   | Username | `root` |
   | Password | `password` |

## Step 4 — Upload `xqsystem.lua` to OpenWrt

1. Open **WinSCP** and connect to the OpenWrt VM (protocol `SCP` or `SFTP`, host `192.168.5.1`, user `root`, password `password`).
2. Upload the file **`xqsystem.lua`** (from the tool package) into this directory on OpenWrt:

   ```
   /usr/lib/lua/luci/controller/admin/
   ```

   This script adds the endpoint the stock Xiaomi firmware will call during Step 8, which is what actually unlocks SSH.

## Step 5 — Change the OpenWrt LAN IPv4 Address

In the OpenWrt web UI, go to **Network → Interfaces → LAN → Edit**, and change the **IPv4 address** to:

```
169.254.31.1
```

Save & Apply. This address segment prevents conflicts with the Xiaomi router's own `192.168.31.0/24` network while still letting the two devices talk to each other during the unlock.

## Step 6 — Enable the Mobile Hotspot for the Connection

1. Turn **on** the mobile hotspot you created in Step 1.
2. On the PC, open the hotspot's network connection properties and **uncheck "Internet Protocol Version 4 (TCP/IPv4)"** — the hotspot link is used only to carry traffic between the PC/VM and the router; it must not provide its own IPv4 configuration.

## Step 7 — Configure the VM's Bridged Networking

In VMware:

```
Edit → Virtual Network Editor → Change Settings
```

Set the relevant adapter to **Bridged (Bridged mode)** so the OpenWrt VM gets an address on the same network as the Xiaomi router. Run the Virtual Network Editor **as Administrator**. Click **OK** to save.

## Step 8 — Log In to the Xiaomi AX6000 (Trigger the Unlock)

Open your browser and visit the URL below **while logged in** to the Xiaomi router admin. Replace `XXXXXX` with your current **stok** (session token) and `xxx` with the correct **admin nonce** (both are obtained after logging in to `http://192.168.31.1` — the stok is visible in the address bar of the admin page):

```
http://192.168.31.1/cgi-bin/luci/;stok=XXXXXX/api/xqsystem/extendwifi_connect_inited_router?ssid=OpenWrt&password=12345678&encryption=WPA2PSKenctype=CCMP&channel=1&band=2g&admin_username=root&admin_password=admin&admin_nonce=xxx
```

Notes:

- `ssid=OpenWrt`, `password=12345678` and `channel=1` must match the hotspot you created in Step 1.
- This request makes the router connect to your hotspot and run the `xqsystem.lua` script you uploaded in Step 4, which unlocks SSH.

## Step 9 — Retrieve the Default root Password from the Serial Number (SN)

The SN is printed on the bottom label of the router (it can also be seen in the admin UI at `192.168.31.1`). Use it to calculate the default `root` account password:

```
https://miwifi.dev/ssh
```

* If the website above is unreachable, download and use this offline tool instead: <a href="https://github.com/aews/lyq/releases/download/AX6000/SN.zip" target="_blank">Download SN tool >></a> <br>

SN format example:

```
30212/F1ZD22032
```

After that, connect with PuTTY to the router's IP on **port 22** using `root` and the calculated password. If the unlock succeeded, the login banner will show:

```
ARE U OK
```

---

## FAQ / Troubleshooting

**Important notes:**

1. The **stok** changes every time you log in to the Xiaomi router admin. Always copy the latest stok before retrying.
2. After a successful SSH unlock there may be **no visible confirmation** in the browser — verify by logging in with PuTTY. Seeing `ARE U OK` means the unlock worked.
3. If the first attempt fails, log in to the Xiaomi admin again, copy a fresh **stok**, and repeat Step 8.

**If you cannot log in to OpenWrt at `192.168.5.1`:**

1. Restart OpenWrt — either reboot it manually or run the command `reboot` in its console.
2. Confirm the OpenWrt LAN IP was actually changed to `169.254.31.1` (Step 5). If it was, you must use **that** address to log in.
3. Change your PC's local network adapter IP into the **same subnet** as the OpenWrt IP (e.g. `169.254.31.x`).
4. In VMware: shut down the running OpenWrt VM, then go to **Edit → Virtual Network Editor → Change Settings → Restore Default Settings**, and set Bridged mode again (Step 7).
5. Enter the OpenWrt IP in the browser to log in.

**If you get the error `{"msg":"一键换机过程中请求对端接口失败","code":1643}`
(translated: *"One-click device migration: request to the peer interface failed"*):**

In the OpenWrt VM console, run:

```
vim /etc/config/dhcp
```

Then press and hold **`d`** to delete all content, press **ESC**, type **`:wq`**, and press **Enter** to save and quit. Finally reboot the VM:

```
reboot
```

---

# Installing ShellClash for Proxy Access

Xiaomi AX6000 internet-access tutorial video: https://youtu.be/vn5QdyDlTtI

SSH into OpenWrt (or the router, depending on where you are installing) and run the following one-liner to install **ShellClash**:

```
export url='https://testingcf.jsdelivr.net/gh/juewuy/ShellCrash@master' && sh -c "$(curl -kfsSl $url/install.sh)" && . /etc/profile &> /dev/null
```

If the installation fails (e.g. the mirror is blocked or unreachable), use one of the backup install sources below:

```
#1
export url='https://raw.githubusercontent.com/juewuy/ShellCrash/master' && sh -c "$(curl -kfsSl $url/install.sh)" && . /etc/profile &> /dev/null
```

```
#2
export url='https://fastly.jsdelivr.net/gh/juewuy/ShellCrash@master' && sh -c "$(curl -kfsSl $url/install.sh)" && source /etc/profile &> /dev/null
```

```
#3
export url='https://gh.jwsc.eu.org/master' && sh -c "$(curl -kfsSl $url/install.sh)" && . /etc/profile &> /dev/null
```

```
#4
export url='https://testingcf.jsdelivr.net/gh/juewuy/ShellCrash@master' && wget -q --no-check-certificate -O /tmp/install.sh $url/install.sh  && sh /tmp/install.sh && . /etc/profile &> /dev/null
```

```
#5
export url='http://t.jwsc.eu.org' && wget -q -O /tmp/install.sh $url/install.sh  && sh /tmp/install.sh && . /etc/profile &> /dev/null
```

```
#6
export url='https://testingcf.jsdelivr.net/gh/juewuy/ShellCrash@master' \
&& sh -c "$(curl -kfsSl $url/install.sh)" \
&& . /etc/profile &> /dev/null
```

After installation, follow the on-screen menu to import your subscription/rules, then manage Clash from the router or from the web UI at:

- **Clash Web UI:** http://192.168.31.1:9999/ui

---

## Useful Links

- [PuTTY download >>](https://github.com/aews/lyq/releases/download/AX6000/putty.zip)
- [Clash Web UI — http://192.168.31.1:9999/ui](http://192.168.31.1:9999/ui)
- Proxy provider ("airport") used by the author: http://www.txyun.xyz/
- Telegram channel: https://t.me/kjfxlyq
