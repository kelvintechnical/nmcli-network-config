# Lab: Command-Line Network Config with `nmcli`

**Series:** linux-ops-mastery — RHCSA Networking
**Subjects covered:** NetworkManager mental model, `nmcli` general/device/connection views, reading and modifying connection profiles, IPv4 method (`auto` vs `manual`), DNS fields on a profile, `nmcli con reload`, bringing connections up/down, `nmcli dev reapply`, verifying with `ip` and `ping`
**Career arcs covered:** RHCSA (NetworkManager is the default stack on EX200-style RHEL 9), RHCE (Ansible `community.general.nmcli` mirrors the same object model), SRE (safe interface changes without editing files by hand), DevOps (immutable images still need runtime NIC tuning), AI/MLOps (GPU nodes often need static management IPs and DNS before pulling containers)
**Prerequisite:** Comfort with `ip addr`, basic TCP/IP (address, prefix, gateway), and root or `sudo` on a RHEL 9 VM
**Time Estimate:** 30 to 45 minutes
**Difficulty arc:** Task 1 read-only inventory · 2–3 inspect and modify a connection profile · 4 apply changes safely · 5 scripting and non-interactive edits · 6 RHCSA-realistic capstone plus teardown

---

## Objective

Build reflexes for **NetworkManager on the command line** so you can answer RHCSA-style prompts without opening a GUI or guessing which file to edit. By the end of this lab you can list devices and connections, identify which profile owns an interface, change IPv4 addressing or DNS on that profile, and **activate** the change in a way the running kernel actually uses.

The capstone is the RHCSA-realistic framing: *"Using only `nmcli`, attach a static IPv4 address with the correct prefix length and gateway to the active Ethernet connection, set two upstream DNS servers, bring the profile fully up, and prove reachability with `ping` — then restore the original settings."*

> **Lab safety note:** Run this on a **lab VM** with console or secondary access. Mis-typing a gateway on your only NIC can drop SSH. If you are on a cloud instance, use the provider’s serial console before you change the primary interface.

---

## Concept: NetworkManager Sits Between You and the Kernel

On RHEL 9, **NetworkManager** owns the policy for interfaces: DHCP vs static, DNS, routes, and activation order. `nmcli` is the stable, scriptable front end to that daemon. You are not “configuring Linux networking” in the abstract — you are editing **connection profiles** (objects) that NetworkManager applies to **devices** (kernel interfaces).

```
   ┌─────────────────────────────────────────────────────────────┐
   │  You: nmcli con mod … / nmcli con up …                      │
   ├─────────────────────────────────────────────────────────────┤
   │  NetworkManager (nm-daemon)  ← reads profiles, talks dbus   │
   ├─────────────────────────────────────────────────────────────┤
   │  Connection profile (UUID, name, ipv4.*, ipv6.*, dns…)     │
   ├─────────────────────────────────────────────────────────────┤
   │  Device (eth0, ens192, wlan0)  ← kernel netdev              │
   ├─────────────────────────────────────────────────────────────┤
   │  Kernel: addresses, routes, neigh tables                    │
   └─────────────────────────────────────────────────────────────┘
```

`ip addr` shows what the kernel has **right now**. `nmcli con show NAME` shows what NetworkManager **remembers** for the next boot and the next `con up`.

> **Why this matters:** RHCSA rewards operators who can fix a broken NIC from a rescue shell. Editors and GUIs may be missing; `nmcli` is always there. The same object model also maps cleanly to Ansible, Terraform cloud-init, and Kickstart `%post` scripts — one mental model everywhere.

---

## 📜 Why `nmcli` Exists — The Story

Early Linux networking was a collage: `ifconfig` from net-tools, hand-edited `/etc/sysconfig/network-scripts/ifcfg-*` files, `route` commands, and distribution-specific init scripts that raced each other at boot. Predictable behavior across laptops, servers, and containers was hard.

**NetworkManager** appeared to unify policy: one daemon that listens for link changes, user sessions, VPNs, and radio killswitches, then applies the right profile at the right time. At first, admins pushed back — the early GUI-centric workflow felt opaque. The project answered with **`nmcli`**, a command-line interface that exposes the same connection/device model without X11.

On **RHEL 7 and later**, NetworkManager became the default for server and desktop. **RHEL 9** continues that default: `nmcli` is how you add a static IP to a server whose primary disk was cloned, how you attach DNS to a connection before `podman` pulls an image, and how you recover when a bad `con mod` dropped your SSH session (from console, `nmcli con down` / `up` or reload a known-good profile).

The historical arc is not about one vendor — it is the industry-wide move from “edit a hundred ifcfg files” to “describe intent; let a daemon reconcile state.” `nmcli` is the portable expression of that intent on Red Hat systems.

> **The point of the story:** Exams and production both reward **idempotent, repeatable** network changes. `nmcli con mod` + `nmcli con up` is the idiom because it survives typos in fewer places than scattering `ip` commands that vanish on reboot.

---

## 👪 The `nmcli` Family — Who Lives There

### By object type

| Object | `nmcli` noun | What it represents |
|---|---|---|
| **General** | `nmcli general` | Overall NM state, hostname, connectivity |
| **Device** | `nmcli device` | Kernel interfaces NM knows about (ethernet, bridge, tun) |
| **Connection** | `nmcli connection` | A saved profile: IPv4 method, addresses, DNS, routes |
| **Radio** | `nmcli radio` | Wi-Fi and WWAN hardware kill state |
| **Agent** | `nmcli agent` | Secrets handling (less common in headless labs) |

### By read vs write workflow

| Goal | Typical commands |
|---|---|
| Inventory | `nmcli -t -f NAME,UUID,TYPE,DEVICE con show` |
| Inspect one profile | `nmcli con show "Wired connection 1"` |
| Change IPv4 | `nmcli con mod NAME ipv4.method manual ipv4.addresses … ipv4.gateway …` |
| Change DNS on profile | `nmcli con mod NAME ipv4.dns "8.8.8.8 1.1.1.1"` |
| Apply now | `nmcli con up NAME` or `nmcli dev reapply IFACE` |
| Reread files | `nmcli con reload` (after hand-edited keyfiles, rare in lab) |

### Companion tools on RHEL 9

| Tool | Role |
|---|---|
| `ip` | Kernel truth: addresses, routes, links |
| `ss` | Sockets listening or connected |
| `resolvectl` | Stub resolver + per-link DNS view |
| `ping`, `tracepath` | Quick L3 reachability checks |

> **The point of the family tree:** `nmcli` edits **intent**; `ip` shows **effect**. When they disagree, NetworkManager has not applied the profile yet — or another tool bypassed NM.

---

## 🔬 The Anatomy of `nmcli con show` — In One Diagram

```
$ nmcli con show "Wired connection 1" | head -n 25
connection.id:                          Wired connection 1
connection.uuid:                      6b3b9b2c-…
connection.type:                      802-3-ethernet
connection.interface-name:            ens192
ipv4.method:                          auto
ipv4.addresses:                       --
ipv4.gateway:                         --
ipv4.dns:                             --
  │        │                              │
  │        │                              └─ empty when DHCP supplies DNS
  │        └─ method: auto = DHCP for IPv4 on many server profiles
  └─ stable name you pass to `nmcli con mod` / `con up`
```

Key fields you will touch in this lab:

```
ipv4.method          auto | manual | link-local | shared | disabled
ipv4.addresses       192.0.2.10/24        (static CIDR form)
ipv4.gateway         192.0.2.1
ipv4.dns             "8.8.8.8 1.1.1.1"    (space-separated list string)
ipv4.ignore-auto-dns yes | no            (whether to ignore DHCP DNS)
```

> **Reading rule:** Always capture `nmcli con show NAME` **before** edits. Restoration is just `con mod` back to those values.

---

## 📚 `nmcli` Reference Table

| Task | Command | Notes |
|---|---|---|
| Show NM status | `nmcli general status` | Quick health snapshot |
| List devices | `nmcli dev status` | DEVICE column shows what is connected |
| List profiles | `nmcli con show` | Human table; add `-t` for scripts |
| Show one profile | `nmcli con show "PROFILE"` | Machine-readable `key: value` |
| Set static IPv4 | `nmcli con mod PROFILE ipv4.method manual ipv4.addresses 192.0.2.10/24 ipv4.gateway 192.0.2.1` | Adjust to your lab subnet |
| Append a second address | `nmcli con mod PROFILE +ipv4.addresses 192.0.2.11/24` | Leading `+` appends |
| Set DNS servers | `nmcli con mod PROFILE ipv4.dns "8.8.8.8 1.1.1.1"` | Space inside quotes |
| Ignore DHCP DNS | `nmcli con mod PROFILE ipv4.ignore-auto-dns yes` | Common with static DNS |
| Apply profile | `nmcli con up PROFILE` | Brings profile up on its interface |
| Reapply to device | `nmcli dev reapply ens192` | Useful after file tweak |
| Reload profiles | `nmcli con reload` | After keyfile edits |

> **Rule one of nmcli:** Match the **profile name** exactly — quotes matter when the name contains spaces.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | EX200 expects you to configure networking without a GUI. `nmcli` is the default answer path on RHEL 9. |
| **RHCE candidate** | Ansible’s `nmcli` module parameters map 1:1 to `ipv4.*` properties you set here. |
| **SRE / Platform** | Incidents start at “can this host route?” — `nmcli` + `ip rule`/`ip route` is the recovery toolkit. |
| **DevOps** | Cloud-init and image builders often inject NM keyfiles; debugging them requires reading `nmcli con show`. |
| **AI / MLOps** | Training nodes need deterministic DNS and NCCL-friendly interfaces before frameworks start. |

---

## 🔧 The 6 Tasks

> Six phases that walk **inventory → inspect → modify → activate → script → capstone**.

---

### Task 1 — Baseline: prove NetworkManager and list devices

**Purpose:** Confirm `nmcli` is available, capture general state, and map **device names** to **connection profiles** before you change anything.

```bash
sudo -i
nmcli general status
nmcli dev status
nmcli con show
```

**Human-Readable Breakdown:** Become root (or stay with `sudo` consistently). `general status` reports whether NetworkManager is running and the overall connectivity state. `dev status` lists each interface and whether a profile is attached. `con show` lists saved profiles and which DEVICE is using each.

**Reading it left to right:** `nmcli` is the CLI front end. `general`, `device`, and `connection` are subcommands. The tables are wide — in a narrow terminal, pipe to `less -S`.

**The story:** Every safe network change starts with a screenshot in your head: which profile name owns `ens192`? If you do not know, you are one `con mod` away from editing Wi-Fi instead of Ethernet.

**Expected output:**

```text
STATE      CONNECTIVITY  WIFI-HW  WIFI     WWAN-HW  WWAN
connected  full          enabled  enabled  enabled  enabled

DEVICE  TYPE      STATE      CONNECTION
ens192  ethernet  connected  Wired connection 1
lo      loopback  connected  (externally managed)

NAME                UUID                                  TYPE      DEVICE
Wired connection 1  6b3b9b2c-…                            ethernet  ens192
lo                  …                                     loopback  lo
```

**Switches**

| Token | Meaning |
|---|---|
| `nmcli general status` | Overall NM + connectivity |
| `nmcli dev status` | Per-interface NM view |
| `nmcli con show` | Profiles and active bindings |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `NetworkManager is not running` | `systemctl enable --now NetworkManager` |
| `nmcli: command not found` | Install `NetworkManager` package group / `dnf install NetworkManager` |
| Device shows `unmanaged` | Another tool owns it — check `NM_UNMANAGED` or dispatcher rules (out of scope; pick a managed NIC) |

---

### Task 2 — Inspect the active Ethernet profile in detail

**Purpose:** Dump the full `key: value` profile for the connection tied to your lab NIC — this is your rollback reference.

```bash
NIC=$(nmcli -t -f DEVICE,TYPE dev | awk -F: '$2=="ethernet"{print $1; exit}')
echo "Using NIC: $NIC"

PROFILE=$(nmcli -t -f NAME,DEVICE con show --active | awk -F: -v d="$NIC" '$2==d{print $1; exit}')
echo "Using PROFILE: $PROFILE"

nmcli con show "$PROFILE" | egrep 'connection\.|ipv4\.|ipv6\.|dns'
```

**Human-Readable Breakdown:** Script-friendly `nmcli -t` prints colon-separated records. The first ethernet device becomes `$NIC`. The active connection whose DEVICE matches that NIC becomes `$PROFILE`. Then grep the high-signal lines from `con show`.

**Reading it left to right:** `-t` selects tabular “parse me” mode. `-f` limits columns. `awk` picks the first ethernet row. The second `nmcli` query maps active NAME to DEVICE.

**The story:** On exam day, profile names vary (`Wired connection 1` vs `System eth0`). Computing `$PROFILE` once prevents fat-finger renames.

**Expected output:**

```text
Using NIC: ens192
Using PROFILE: Wired connection 1
connection.id:                        Wired connection 1
connection.uuid:                      6b3b9b2c-…
connection.type:                      802-3-ethernet
connection.interface-name:            ens192
ipv4.method:                          auto
ipv4.dns:                             --
ipv4.ignore-auto-dns:                 no
ipv6.method:                          auto
```

**Switches**

| Token | Meaning |
|---|---|
| `-t` | Tabular colon output for scripts |
| `-f FIELD,...` | Column selection |
| `con show --active` | Only profiles currently up |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `$PROFILE` empty | No active ethernet — bring one up with `nmcli con up` or attach a cable / virt NIC |
| Multiple ethernets | Tighten `awk` or set `NIC` manually to the exam interface |

---

### Task 3 — Core change: add static DNS servers to the profile

**Purpose:** Practice a **non-destructive** `con mod` that does not yet change your address — only upstream resolvers on the same DHCP lease.

```bash
nmcli con mod "$PROFILE" ipv4.dns "8.8.8.8 1.1.1.1"
nmcli con mod "$PROFILE" ipv4.ignore-auto-dns yes

nmcli con show "$PROFILE" | egrep 'ipv4\.dns|ignore-auto-dns|method'
nmcli con up "$PROFILE"
```

**Human-Readable Breakdown:** Set `ipv4.dns` to two public resolvers (lab only — corporate networks may block them). Flip `ipv4.ignore-auto-dns` to `yes` so DHCP-provided DNS does not override your list. Show the resulting fields, then `con up` to apply.

**Reading it left to right:** `con mod` mutates the persistent profile. `con up` reapplies it to the kernel and resolver stack. Until `con up`, some properties may not fully reflect in traffic.

**The story:** This mirrors the real ticket: “hardcode DNS for this DB subnet.” You are learning the **property names** the exam cares about.

**Expected output:**

```text
ipv4.method:                          auto
ipv4.dns:                             8.8.8.8;1.1.1.1;
ipv4.ignore-auto-dns:                 yes
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/3)
```

**Switches**

| Token | Meaning |
|---|---|
| `ipv4.dns` | Space-separated in CLI; semicolons in `con show` output |
| `ipv4.ignore-auto-dns yes` | Prefer static list over DHCP option 6 |
| `nmcli con up PROFILE` | Apply profile now |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| DNS still old | Run `resolvectl status` and confirm the link points at NM; then `nmcli dev reapply $NIC` |
| `Error: Failed to modify connection` | Typo in profile name — reprint `nmcli con show` |

---

### Task 4 — Verification: resolver view and reachability

**Purpose:** Prove the running system picked up resolver configuration and basic L3 still works.

```bash
resolvectl status | sed -n '1,40p'
ping -c 3 google.com
ip -4 addr show dev "$NIC"
```

**Human-Readable Breakdown:** `resolvectl` shows what **systemd-resolved** (default stub path on RHEL 9) thinks per link. `ping` validates DNS + routing. `ip addr` shows addresses still present on `$NIC`.

**Reading it left to right:** DNS is not magic — it is UDP/TCP to a resolver IP learned from NM, then A/AAAA queries, then ICMP echo for the ping target.

**The story:** Graders and SREs trust **three independent witnesses**: NM profile (`nmcli con show`), resolver (`resolvectl`), and kernel (`ip route get 1.1.1.1`).

**Expected output:**

```text
Global
       Protocols: +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Link 2 (ens192)
    Current Scopes: DNS
         DNS Servers: 8.8.8.8 1.1.1.1
...
PING google.com (142.250.191.14) 56(84) bytes of data.
64 bytes from … icmp_seq=1 ttl=117 time=12.3 ms
...
3 packets transmitted, 3 received, 0% packet loss
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.0.2.50/24 brd 192.0.2.255 scope global dynamic noprefixroute ens192
```

**Switches**

| Token | Meaning |
|---|---|
| `resolvectl status` | Per-link DNS + domain routing |
| `ping -c 3` | Stop after three echoes |
| `ip -4 addr show dev IF` | IPv4 addresses only, one interface |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `ping: google.com: Name or service not known` | DNS broken — verify `ipv4.dns` and `con up`; try `ping 8.8.8.8` to isolate |
| No route | Gateway wrong if you already went static — fix `ipv4.gateway` |

---

### Task 5 — Edge case: switch to **manual** IPv4 on a disposable alias (safer pattern)

**Purpose:** Practice the full static stanza **without** guessing the live gateway on your only SSH path. Add a **second** address to the same NIC via `+ipv4.addresses`, keeping the primary DHCP address intact.

```bash
nmcli con mod "$PROFILE" +ipv4.addresses 198.51.100.10/24
nmcli con up "$PROFILE"

ip -4 addr show dev "$NIC"
ping -c 2 -I 198.51.100.10 198.51.100.1 || echo "expected fail if .1 is not a router"
```

**Human-Readable Breakdown:** The leading `+` on `+ipv4.addresses` appends an additional address block. Your original DHCP address should remain. The ping to `198.51.100.1` may fail unless that router exists — the goal is to see the **secondary** source IP in `ping` output when it succeeds; otherwise you still proved the address attached.

**Reading it left to right:** NetworkManager programs secondary local addresses as **secondary** labels on the interface. Policy routing is not required for simple receive tests.

**The story:** In production you often add a service VIP this way. On the exam, they may require replacing `auto` entirely — Task 6 does the disciplined version with rollback.

**Expected output:**

```text
Connection successfully activated …
2: ens192: …
    inet 192.0.2.50/24 … dynamic … ens192
    inet 198.51.100.10/24 brd 198.51.100.255 scope global secondary ens192
PING 198.51.100.1 (198.51.100.1) from 198.51.100.10 …
From 198.51.100.10 icmp_seq=1 Destination Host Unreachable
expected fail if .1 is not a router
```

**Switches**

| Token | Meaning |
|---|---|
| `+ipv4.addresses` | Append an extra static address |
| `ping -I ADDR` | Bind outgoing ping to source IP |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Secondary address missing | `nmcli con show "$PROFILE" \| grep addresses` — check for typos |
| `Error: invalid prefix '24 '` | Remove stray spaces inside CIDR |

---

### Task 6 — Capstone: temporary static renumber + proof + full rollback

**Task statement:** *"For the active lab Ethernet profile, switch `ipv4.method` to `manual`, set `ipv4.addresses` to `192.0.2.150/24`, set `ipv4.gateway` to `192.0.2.1`, set DNS to `9.9.9.9` and `149.112.112.112`, activate, show kernel routes, then restore the Task 2 baseline from your notes."*

**Purpose:** Execute a full static migration **only if** your lab subnet actually matches `192.0.2.0/24`. If it does not, substitute your real network’s address, prefix, and gateway from Task 2’s `ip route` output.

```bash
# Capture rollback fields BEFORE you change method:
nmcli -g connection.uuid con show "$PROFILE" > /root/nmcli-lab-rollback.txt
nmcli con show "$PROFILE" | egrep 'ipv4\.(method|addresses|gateway|dns)|ignore-auto-dns' | tee -a /root/nmcli-lab-rollback.txt

# CAPSTONE (edit numbers if your lab is not 192.0.2.0/24):
nmcli con mod "$PROFILE" ipv4.method manual \
  ipv4.addresses 192.0.2.150/24 \
  ipv4.gateway 192.0.2.1 \
  ipv4.dns "9.9.9.9 149.112.112.112" \
  ipv4.ignore-auto-dns yes
nmcli con up "$PROFILE"

ip route | grep default
ping -c 2 192.0.2.1

# ROLLBACK (example — replace with values from your rollback file):
nmcli con mod "$PROFILE" ipv4.method auto ipv4.addresses "" ipv4.gateway ""
nmcli con mod "$PROFILE" ipv4.dns "" ipv4.ignore-auto-dns no
nmcli con up "$PROFILE"
```

**Human-Readable Breakdown:** Snapshot the current ipv4 stanza. Apply manual IPv4 + gateway + DNS. Prove default route and gateway ping. Restore `auto` method, clear static fields, re-enable DHCP DNS behavior, reactivate.

**Layer stack you built:**

```text
nmcli con mod …          ← author intent in NetworkManager
nmcli con up …           ← apply to netdev
ip route / ping          ← witness kernel L3
/root/nmcli-lab-rollback.txt  ← human proof for graders / future you
```

**The story:** The exam rewards **correct order**: snapshot, modify, activate, verify, rollback. Never “fix forward” blind on your only NIC without a console.

**Expected verification output:**

```text
ipv4.method:                          manual
ipv4.addresses:                       192.0.2.150/24
ipv4.gateway:                         192.0.2.1
default via 192.0.2.1 dev ens192 proto static metric …
64 bytes from 192.0.2.1: icmp_seq=1 ttl=64 time=0.3 ms
```

**Cleanup**

```bash
nmcli con mod "$PROFILE" +ipv4.addresses "" 2>/dev/null || true
# If secondary address persists, remove explicitly with:
# nmcli con mod "$PROFILE" ipv4.addresses '192.0.2.150/24'  # then edit down — simplest: restore from rollback snapshot manually

rm -f /root/nmcli-lab-rollback.txt
exit
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| SSH dropped after `con up` | Console in — wrong gateway or /24 on wrong segment |
| `Could not activate` | IP collision — pick a free host address |
| Cannot restore DHCP | `ipv4.method auto` **and** clear static `ipv4.addresses` / `gateway` fields |

---

## 🔍 `nmcli` Decision Guide

```
Need to change networking on RHEL 9?
  │
  ├── "Is NetworkManager managing this NIC?"
  │       ├── yes → continue with nmcli
  │       └── no  → fix NM control first (out of scope)
  │
  ├── "Read-only triage"
  │       └── nmcli dev status → nmcli con show --active → ip addr
  │
  ├── "Edit DNS but keep DHCP address"
  │       └── ipv4.dns … + ipv4.ignore-auto-dns yes → con up
  │
  ├── "Move to static IPv4"
  │       └── snapshot → ipv4.method manual + addresses + gateway → con up → ping gateway
  │
  ├── "Add VIP / secondary address"
  │       └── +ipv4.addresses CIDR → con up
  │
  └── "Undo everything"
          └── restore fields from snapshot → con up → verify `ip addr`
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Capture `nmcli general`, devices, and connections
- [ ] 02 Resolve `$PROFILE` for the lab ethernet NIC and dump `ipv4.*`
- [ ] 03 Set static DNS + `ignore-auto-dns` and `con up`
- [ ] 04 Verify with `resolvectl`, `ping`, and `ip addr`
- [ ] 05 Append a secondary address with `+ipv4.addresses`
- [ ] 06 Capstone static renumber + proof + rollback + cleanup

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Typo in profile name | `Error: unknown connection` | Copy/paste `nmcli con show` |
| Forgot `con up` | `ip addr` unchanged vs expectation | Always activate after mod |
| Wrong CIDR prefix | Host unreachable on local segment | `/24` vs `/26` matters |
| Static without gateway | Pings LAN work, DNS fails | Set `ipv4.gateway` |
| Two defaults | Flaky routing | Check for duplicate profiles both “automatically connect” |
| Edited wrong NIC | SSH died, wrong interface moved | Use `$NIC` discovery or serial console |
| Quoted DNS wrong | Parser error | One pair of quotes around the whole list |
| Assumed `ifcfg` path | File not where tutorials said | RHEL 9 still supports keyfiles under `/etc/NetworkManager/system-connections/` |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Memorize the spine: `nmcli con show` → `nmcli con mod … ipv4.*` → `nmcli con up`. Say aloud **method / addresses / gateway / dns** when under pressure.

**RHCE candidate**
- Translate this lab directly into `community.general.nmcli` tasks — same keys, YAML instead of CLI.

**SRE / Platform interview**
- Explain difference between **systemd unit restart** and **`nmcli con up`**: NM reconciles desired state without necessarily restarting unrelated links.

**DevOps**
- Golden images should ship DHCP; bake **activation scripts** that call `nmcli` for site-specific statics in second boot.

**AI / MLOps**
- Multi-homed GPU nodes: document which profile owns **IB** vs **Ethernet** before you `con mod` DNS for pulls from registries.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 37 — Local Host Resolution (`/etc/hosts`) | Overrides DNS for specific names |
| Lab 38 — Configuring DNS Servers (`/etc/resolv.conf`) | Resolver output after NM sets DNS |
| Lab 39 — SSH and Key-Based Auth | Network must be up before `ssh-copy-id` |
| Lab — `ip` and routing basics (if in series) | Kernel view after NM applies routes |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
