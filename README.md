# TeraBox → Google Drive Transfer Guide
### Using AList on Oracle Cloud Infrastructure (OCI) ARM

> **Author:** Adarsh Reddy  
> **Method:** AList self-hosted on OCI Mumbai ARM Flex → Google Drive  
> **Transfer Size:** ~140 GB  
> **Cost:** Free (OCI Always Free) + TeraBox Premium (~₹300/month, optional but recommended)

---

## Table of Contents

1. [Overview](#overview)
2. [Why This Method](#why-this-method)
3. [Prerequisites](#prerequisites)
4. [Architecture](#architecture)
5. [Phase 1 — OCI VM Setup](#phase-1--oci-vm-setup)
6. [Phase 2 — Fix apt Mirror (OCI Mumbai ARM)](#phase-2--fix-apt-mirror-oci-mumbai-arm)
7. [Phase 3 — Install AList](#phase-3--install-alist)
8. [Phase 4 — Get TeraBox Session Cookie](#phase-4--get-terabox-session-cookie)
9. [Phase 5 — Configure AList via SSH Tunnel](#phase-5--configure-alist-via-ssh-tunnel)
10. [Phase 6 — Add TeraBox Storage](#phase-6--add-terabox-storage)
11. [Phase 7 — Add Google Drive Storage](#phase-7--add-google-drive-storage)
12. [Phase 8 — Configure Google Drive API (Own Credentials)](#phase-8--configure-google-drive-api-own-credentials)
13. [Phase 9 — Start the Transfer](#phase-9--start-the-transfer)
14. [Phase 10 — Monitor and Handle Errors](#phase-10--monitor-and-handle-errors)
15. [Troubleshooting](#troubleshooting)
16. [What Doesn't Work (Save Your Time)](#what-doesnt-work-save-your-time)
17. [FAQ](#faq)

---

## Overview

This guide walks you through transferring all your data from TeraBox (1024terabox.com) to Google Drive using AList — a self-hosted file manager that supports both platforms — running on an Oracle Cloud Infrastructure (OCI) ARM instance. The transfer runs entirely in the cloud, your laptop can be off, and it handles retries automatically.

---

## Why This Method

TeraBox's free accounts block programmatic downloads via their API (`errno: 2`, `errno: 113`). Every other method was tested and failed:

| Method | Result |
|---|---|
| Direct TeraBox API | `errno: 2` — blocked for free accounts |
| yt-dlp | Unsupported URL |
| MultCloud | Dropped TeraBox support |
| rclone (no native TeraBox backend) | Not supported |
| Python scripts + filemetas endpoint | `info: []` — empty response |
| AirExplorer (free) | Works but throttled to ~50 KB/s |

**AList with TeraBox Premium** is the only fully automated, server-side solution that works reliably.

---

## Prerequisites

- **OCI ARM Flex instance** (Always Free — 4 OCPU, 24 GB RAM available, use at least 1 OCPU / 6 GB RAM)
- **Ubuntu 22.04** on the OCI instance
- **TeraBox Premium** (~₹300/month) — free accounts hit `errno: 113` speed limit errors
- **Google account** with sufficient Drive storage
- **SSH access** to your OCI VM
- **Windows/Mac laptop** for browser-based OAuth steps

---

## Architecture

```
TeraBox Account (1024terabox.com)
        │
        │  [AList Terabox driver — "Crack" mode]
        ▼
OCI Mumbai ARM Instance
(AList running on port 5244)
        │
        │  [AList GoogleDrive driver — your own OAuth credentials]
        ▼
Google Drive
```

Everything runs on OCI. Once started, your laptop can be turned off.

---

## Phase 1 — OCI VM Setup

### 1.1 Create an OCI Always Free ARM Instance

1. Go to [cloud.oracle.com](https://cloud.oracle.com)
2. Compute → Instances → Create Instance
3. Select **Ampere A1 Flex** (ARM) — this is Always Free eligible
4. Set shape: **1 OCPU, 6 GB RAM** minimum (you can go up to 4 OCPU / 24 GB on Always Free)
5. Boot volume: **100 GB** (Always Free gives you 200 GB total)
6. Image: **Ubuntu 22.04**
7. Add your SSH public key
8. Create

### 1.2 Open Required Ports

Go to OCI Console → Networking → Virtual Cloud Networks → your VCN → Security → Default Security List → Add Ingress Rule:

- **Source CIDR:** `0.0.0.0/0`
- **IP Protocol:** TCP
- **Destination Port Range:** `5244`

> **Note:** OCI has 3 layers of firewall: Security Lists, Network Security Groups, and iptables. We'll handle iptables in Phase 3.

### 1.3 SSH Into Your Instance

```bash
ssh -i /path/to/your/private.key ubuntu@YOUR_OCI_PUBLIC_IP
```

On Windows with a config file (`~/.ssh/config`):
```
Host oci-arm
    HostName YOUR_OCI_PUBLIC_IP
    User ubuntu
    IdentityFile C:\path\to\key.pem
    IdentitiesOnly yes
```

Then just: `ssh oci-arm`

---

## Phase 2 — Fix apt Mirror (OCI Mumbai ARM)

> ⚠️ **Critical for OCI Mumbai ARM instances.** The default Ubuntu mirrors are blocked by OCI Mumbai's egress filtering. Skip this and apt will hang forever.

```bash
# Comment out all default sources
sudo sed -i 's/^deb/#deb/g' /etc/apt/sources.list

# Add a working mirror
sudo tee /etc/apt/sources.list.d/fast-mirror.list <<EOF
deb http://mirror.us.leaseweb.net/ubuntu-ports jammy main restricted universe multiverse
deb http://mirror.us.leaseweb.net/ubuntu-ports jammy-updates main restricted universe multiverse
deb http://mirror.us.leaseweb.net/ubuntu-ports jammy-security main restricted universe multiverse
EOF

sudo apt update
```

This should complete in under 30 seconds. If it hangs, the mirror is wrong — try `http://ports.ubuntu.com/ubuntu-ports` as an alternative.

---

## Phase 3 — Install AList

### 3.1 Install AList

```bash
curl -fsSL "https://alist.nn.ci/v3.sh" | sudo bash -s install
```

You'll see output in Chinese (normal — AList is a Chinese open source project). Key info printed:
- **Public URL:** `http://YOUR_IP:5244`
- **Username:** `admin`
- **Initial Password:** (shown in output, save it)

### 3.2 Open Port in iptables

OCI instances use iptables as a third firewall layer:

```bash
sudo iptables -I INPUT 4 -p tcp --dport 5244 -j ACCEPT
```

### 3.3 Configure AList for Reliability

Edit the config to persist tasks and tune concurrency:

```bash
sudo systemctl stop alist

sudo python3 -c "
import json
with open('/opt/alist/data/config.json', 'r') as f:
    config = json.load(f)

# Reduce workers to avoid Google Drive quota issues
config['tasks']['copy']['workers'] = 2
config['tasks']['copy']['max_retry'] = 10
config['tasks']['copy']['task_persistant'] = True
config['tasks']['transfer']['workers'] = 2
config['tasks']['transfer']['max_retry'] = 10
config['tasks']['transfer']['task_persistant'] = True

with open('/opt/alist/data/config.json', 'w') as f:
    json.dump(config, f, indent=2)
print('Done')
"

sudo systemctl start alist
```

### 3.4 Verify AList is Running

```bash
sudo systemctl status alist
curl http://localhost:5244  # should return HTML
```

> **IPv6 Note:** AList on OCI ARM listens on `:::5244` (IPv6 only). Always use `http://[::1]:5244` for local API calls, not `localhost` or `127.0.0.1`.

---

## Phase 4 — Get TeraBox Session Cookie

You need your TeraBox session cookie to authenticate AList with your account.

### 4.1 Get Cookie from Browser

1. Open **1024terabox.com** in Chrome/Brave
2. Log in to your account
3. Press **F12** → go to **Application** tab
4. Left sidebar → Storage → Cookies → `https://www.1024terabox.com`
5. Copy these specific cookie values:

| Cookie Name | Where to find it |
|---|---|
| `ndus` | Required — your session token |
| `csrfToken` | Required — CSRF protection token |
| `browserid` | Required — browser fingerprint |
| `lang` | Optional — set to `en` |
| `ndut_fmt` | Optional but helps |

### 4.2 Construct the Cookie String

Format it as:
```
ndus=VALUE; csrfToken=VALUE; browserid=VALUE; lang=en; ndut_fmt=VALUE
```

### 4.3 Save Cookie on VM

```bash
nano ~/terabox_cookie.txt
# paste the cookie string
# Ctrl+O → Enter → Ctrl+X to save
```

### 4.4 Verify Cookie Works

```bash
python3 -c "
import requests, os
COOKIE = open(os.path.expanduser('~/terabox_cookie.txt')).read().strip()
HEADERS = {'Cookie': COOKIE, 'User-Agent': 'Mozilla/5.0'}
r = requests.get('https://www.1024terabox.com/api/list',
    params={'app_id':'250528','dir':'/','web':'1','num':'10','clienttype':'0'},
    headers=HEADERS)
for f in r.json().get('list', []):
    print(f['isdir'], f['server_filename'])
"
```

You should see your TeraBox file/folder list printed out.

> **Cookie Refresh:** Cookies expire. If you get authentication errors later, repeat this step and update the cookie in AList storage settings.

---

## Phase 5 — Configure AList via SSH Tunnel

AList's web UI runs on port 5244. Since we can't easily expose it (OCI Mumbai blocks many ports), use SSH tunneling to access it from your browser.

### 5.1 Open SSH Tunnel

**On your laptop** — open a new terminal/PowerShell:

```bash
# Linux/Mac
ssh -L 5244:[::1]:5244 ubuntu@YOUR_OCI_IP -N -i /path/to/key.pem

# Windows (using SSH config)
ssh -L 5244:[::1]:5244 oci-arm -N
```

Keep this window open. Do not close it.

### 5.2 Access AList UI

Open your browser and go to:
```
http://localhost:5244
```

Login with:
- **Username:** `admin`
- **Password:** (from Phase 3 install output, e.g., `zH99ynQg`)

---

## Phase 6 — Add TeraBox Storage

1. Go to **Manage** (bottom of the page) → **Storages** → **Add**
2. Select driver: **Terabox**
3. Fill in:

| Field | Value |
|---|---|
| Mount Path | `/terabox` |
| Root folder path | `/` |
| Cookie | (paste your full cookie string from Phase 4) |
| Download api | **Crack** ← Critical, select this not "Official" |

4. Click **Add**

> **Why "Crack" mode?** The "Official" mode hits `www.terabox.com` which is blocked from OCI Mumbai. "Crack" mode uses `www.1024terabox.com` which is accessible.

### 6.1 Verify TeraBox is Connected

Go to **Home** → click **terabox** folder. You should see your files listed. If it's empty, check:
- Cookie is fresh and valid
- Download api is set to "Crack"
- Restart AList: `sudo systemctl restart alist`

---

## Phase 7 — Add Google Drive Storage

### 7.1 Get a Refresh Token

On your **laptop**, use rclone to get a Google Drive refresh token:

**Download rclone** from [rclone.org/downloads](https://rclone.org/downloads/) if not installed.

```bash
rclone config
```

Follow the prompts:
```
n → New remote
Name: gdrive
Type: drive
client_id: [Enter — leave blank for now, use default]
client_secret: [Enter]
scope: 1 (full access)
root_folder_id: [Enter]
service_account_file: [Enter]
Edit advanced config: n
Use web browser: y (if on local machine) OR n (if on remote)
```

If `n` (remote machine without browser):
```bash
# Run this command on your laptop
rclone authorize "drive" "eyJzY29wZSI6ImRyaXZlIn0"
```

Browser opens → sign in with Google → allow access → copy the token from terminal.

Paste the token back into the VM when prompted.

### 7.2 Get the Refresh Token Value

```bash
cat ~/.config/rclone/rclone.conf | grep refresh_token
```

Copy the `refresh_token` value from the JSON.

### 7.3 Add Google Drive in AList

1. **Manage** → **Storages** → **Add**
2. Driver: **GoogleDrive**
3. Fill in:

| Field | Value |
|---|---|
| Mount Path | `/gdrive` |
| Root folder id | `root` |
| Refresh token | (paste from rclone config) |
| Client id | (leave as default for now, replace later) |
| Client secret | (leave as default for now, replace later) |

4. Click **Add**

Verify: Go to **Home** → click **gdrive** → you should see your Google Drive folders.

---

## Phase 8 — Configure Google Drive API (Own Credentials)

The default AList Google credentials have a shared quota used by thousands of users worldwide — you'll hit rate limits quickly. Use your own credentials for unlimited quota.

### 8.1 Create a Google Cloud Project

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a new project (e.g., `drive-api`)
3. Go to **APIs & Services** → search **Google Drive API** → **Enable**

### 8.2 Create OAuth Credentials

1. **APIs & Services** → **Credentials** → **Create Credentials** → **OAuth 2.0 Client ID**
2. If prompted for consent screen:
   - User type: External
   - App name: `alist-drive`
   - Support email: your Gmail
   - Developer contact: your Gmail
   - Save and continue (skip scopes)
3. Application type: **Desktop app**
4. Name: `alist`
5. Click **Create** — download or copy your **Client ID** and **Client Secret**

### 8.3 Add Yourself as Test User

1. **OAuth consent screen** → **Audience** → **Add users**
2. Add your Gmail address
3. Save

### 8.4 Generate New Refresh Token with Your Credentials

On your laptop:

```powershell
.\rclone.exe authorize "drive" --client-id "YOUR_CLIENT_ID" --client-secret "YOUR_CLIENT_SECRET"
```

Browser opens → sign in → allow → copy the new token.

### 8.5 Update AList Google Drive Storage

1. **Manage** → **Storages** → Edit **gdrive**
2. Replace:
   - **Client id** → your Client ID
   - **Client secret** → your Client Secret
   - **Refresh token** → new token from above
3. **Save**

---

## Phase 9 — Start the Transfer

### 9.1 Select All TeraBox Files

1. Go to **Home** → click **terabox**
2. Check the top checkbox to select all files/folders
3. Click the **Copy** button (appears in toolbar when files selected)
4. In the destination picker → select **gdrive** → optionally create a subfolder like `TERABOX`
5. Make sure **Overwrite existing files** is unchecked (to skip already transferred files on retry)
6. Click **OK**

### 9.2 Monitor Progress

Go to **Manage** → **Tasks** → **Copy**

You'll see all transfer tasks with status:
- **Running** — actively transferring
- **Succeeded** — done ✓
- **WaitingRetry** — failed, will retry automatically

### 9.3 Detach and Let It Run

Once tasks are queued, you can close the SSH tunnel — **the transfer continues on OCI**.

The task queue persists across AList restarts (we set `task_persistant: true`).

To reconnect and check progress later:
```bash
# On laptop
ssh -L 5244:[::1]:5244 oci-arm -N
# Then open http://localhost:5244/@manage/tasks/copy
```

---

## Phase 10 — Monitor and Handle Errors

### Common Errors and Fixes

**`illegal seekableStream`**
- Cause: TeraBox free account speed limit
- Fix: Upgrade to TeraBox Premium

**`errno: 113` / `no dlink found`**
- Cause: TeraBox free account download restriction
- Fix: Upgrade to TeraBox Premium

**`Quota exceeded for quota metric 'Queries per minute'`**
- Cause: Using default AList shared Google credentials
- Fix: Set up your own Google Cloud OAuth credentials (Phase 8)

**`connection reset by peer` on terabox.com**
- Cause: OCI Mumbai blocks terabox.com directly
- Fix: Make sure Download api is set to "Crack" (uses 1024terabox.com)

**`unauthorized_client`**
- Cause: Refresh token generated with different client ID
- Fix: Regenerate refresh token using your own client ID/secret

### Retry Failed Tasks

In **Manage → Tasks → Copy** → click **Retry Failed** to restart all failed tasks.

### Check AList Logs

```bash
sudo journalctl -u alist -f
```

---

## Troubleshooting

### AList won't start

```bash
sudo systemctl status alist
sudo journalctl -u alist -n 50
```

### TeraBox folder shows empty

1. Check cookie is valid — run the verification command from Phase 4.4
2. Verify Download api is set to **Crack** in storage settings
3. Restart AList: `sudo systemctl restart alist`
4. Refresh the page and click the terabox folder again

### Can't reach AList web UI

Check the SSH tunnel is running:
```bash
# On VM — verify AList is listening
sudo netstat -tlnp | grep 5244
# Should show: tcp6  :::5244  LISTEN

# Restart tunnel on laptop
ssh -L 5244:[::1]:5244 oci-arm -N
```

### apt hangs on OCI Mumbai

You forgot to apply the mirror fix from Phase 2. Apply it and try again.

### All tasks wiped after AList restart

You didn't set `task_persistant: true`. Edit the config (Phase 3.3) and re-queue the transfer.

---

## What Doesn't Work (Save Your Time)

These approaches were all tested and failed for TeraBox free accounts:

```
❌ TeraBox API /api/filemetas → errno: 0, info: [] (empty)
❌ TeraBox API /api/download → errno: 2 (not authorized)
❌ TeraBox API /api/downloadlocate → errno: 2
❌ xpan endpoint → 404 (removed)
❌ yt-dlp → Unsupported URL
❌ MultCloud → No TeraBox support
❌ terasnap.netlify.app → Cookie validation error
❌ AirExplorer free → 2 GB/hour limit (~50 KB/s effective)
❌ Python scripts with any endpoint → Same API blocks
```

**The root cause:** TeraBox (1024terabox.com) specifically blocks server-side/programmatic downloads for free accounts at the API level. Premium removes these restrictions.

---

## FAQ

**Q: Do I need to keep my laptop on during the transfer?**  
A: No. Once tasks are queued in AList, everything runs on your OCI VM. Close the SSH tunnel, shut down your laptop — the transfer continues.

**Q: What if my OCI VM restarts?**  
A: AList is configured as a systemd service (`enabled`), so it starts automatically. With `task_persistant: true`, tasks survive restarts.

**Q: How long will the transfer take?**  
A: Depends on TeraBox's download speed (even with premium, OCI Mumbai → TeraBox servers can vary). Expect 1-3 days for 100-150 GB.

**Q: Will duplicate files be transferred?**  
A: AList doesn't deduplicate by default. If you need deduplication, download to OCI first and run `fdupes -r --delete --noprompt /path/to/data/` before uploading to Google Drive.

**Q: What if my TeraBox cookie expires mid-transfer?**  
A: Tasks will fail with auth errors. Go to AList → Storages → Edit terabox → update the cookie → Save → Retry Failed tasks.

**Q: Can I use this for other cloud storage instead of Google Drive?**  
A: Yes. AList supports OneDrive, Dropbox, S3, and dozens more. The TeraBox side stays the same — just add a different destination storage.

**Q: Is AList safe?**  
A: AList is open source (GitHub: alist-org/alist, 40k+ stars). Your credentials are stored locally on your OCI VM in a SQLite database. Nobody else has access.

**Q: My TeraBox is on terabox.com not 1024terabox.com — does this work?**  
A: In India, terabox.com is blocked and 1024terabox.com is the accessible domain. They're the same service. The "Crack" mode in AList's Terabox driver specifically uses 1024terabox.com.

---

## Quick Reference Commands

```bash
# Fix apt mirror (OCI Mumbai ARM — run once)
sudo sed -i 's/^deb/#deb/g' /etc/apt/sources.list
sudo tee /etc/apt/sources.list.d/fast-mirror.list <<EOF
deb http://mirror.us.leaseweb.net/ubuntu-ports jammy main restricted universe multiverse
deb http://mirror.us.leaseweb.net/ubuntu-ports jammy-updates main restricted universe multiverse
deb http://mirror.us.leaseweb.net/ubuntu-ports jammy-security main restricted universe multiverse
EOF
sudo apt update

# Install AList
curl -fsSL "https://alist.nn.ci/v3.sh" | sudo bash -s install

# Open iptables for port 5244
sudo iptables -I INPUT 4 -p tcp --dport 5244 -j ACCEPT

# AList service management
sudo systemctl start alist
sudo systemctl stop alist
sudo systemctl restart alist
sudo systemctl status alist

# View AList logs live
sudo journalctl -u alist -f

# SSH tunnel (run on laptop)
ssh -L 5244:[::1]:5244 oci-arm -N

# Test TeraBox cookie
python3 -c "
import requests, os
COOKIE = open(os.path.expanduser('~/terabox_cookie.txt')).read().strip()
HEADERS = {'Cookie': COOKIE, 'User-Agent': 'Mozilla/5.0'}
r = requests.get('https://www.1024terabox.com/api/list', params={'app_id':'250528','dir':'/','web':'1','num':'10','clienttype':'0'}, headers=HEADERS)
[print(f['isdir'], f['server_filename']) for f in r.json().get('list', [])]
"

# AList admin API token
TOKEN=$(curl -s -X POST 'http://[::1]:5244/api/auth/login' \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"YOUR_PASSWORD"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['token'])")
```

---

## Resources

- [AList Documentation](https://alist.nn.ci/guide/)
- [AList GitHub](https://github.com/alist-org/alist)
- [OCI Always Free Resources](https://www.oracle.com/cloud/free/)
- [rclone Downloads](https://rclone.org/downloads/)
- [Google Cloud Console](https://console.cloud.google.com)

---

*Guide written based on real transfer experience. Every error listed in the troubleshooting section was actually encountered and resolved during the process of transferring ~140 GB from TeraBox to Google Drive.*
