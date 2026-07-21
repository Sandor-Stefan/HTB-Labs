
The legend Cicada comes back and it gives me an IP address to enter the system an crack the puzzle

### Enumeration

First is the Enumeration part where `nmap` will help me to uncover the open ports:
```bash
nmap -A -T4 cicada.htb
```
 - The scan reveals basic open ports for a windows machine like `kerberos`, `netbios`, `ldap`, `SMB` and also get the `Host: CICADA-DC`

Since we have SMB open I will try to see if we can connect to it and see what it gives me

### SMB

First thought is to see if anonymous connection is allowed since it is one of the most common misconfigurations on easy machines so I did that with `smbclient`
![[Pasted image 20260720141041.png]]
- the anonymous login worked and I can see that there are some interesting shares present, but the most interesting is the `HR` share

Connecting to the HR share I can see a `.txt` file present:
![[Pasted image 20260720141547.png]]
- I download the file and there a default password `Cicada$M6Corpb*@Lp#nZp!8` is present and a text representing a change of password procedure
![[Pasted image 20260720141714.png]]

The next thing is to enumerate the current users and check which one of them still uses the default password

### Lookupsid

Next we are going to use `Impacket's lookupsid` tool with which will try to brute force the SIDs of any user on the target
```bash
impacket-lookupsid 'cicada.htb/gues'@cicada.htb -no-pass
```
- since the output contains a lot of info and I need only the usernames I write a small bash script to get only the usernames:
![[Pasted image 20260720143515.png]]

### Password Spraying

Now that I have only the usernames next thing is to test each one to see which one still uses the default password from above:
- To execute the `Password Spraying` attack I will use `crackmapexec`
![[Pasted image 20260720143826.png]]
- the output shows that the user who still uses the default password is `michael.wrightson`

No luck in finding additional shares with this user so we have to escalate laterally to another user
![[Pasted image 20260720144323.png]]
### Enumerating Domain Users

I will use again `crackmapexec` to enumerate the other users on the machine but this time, with the user `michael.wrigthson` credentials
![[Pasted image 20260720144626.png]]
- the output gives me a valuable information - the user `david.orelious` wrote his password in the AD description (bad move David), the password is `aRt$Lp#7t*VQ!3`

### Foothold

With `david.orelious` credentials we can check the shares he has access and this time I will use the `crackmapexec` tool
![[Pasted image 20260720145303.png]]
- the `DEV` share is accessible with read permissions so let's check it:

The `DEV` share uncover a PowerShell script available and I downloaded it:
![[Pasted image 20260720145501.png]]

The script creates a backup `zip` but the important information I get, represents the following user credentials:
	- user - `emily.oscars`
	- password - `Q!3@Lp#M6b*7t*Vt`

Since there is a script that runs with those credentials I think I can get a shell with the help of `Evil-WinRM`
```bash
evil-winrm -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt' -i cicada.htb
```
- The shell works and it gives us the user flag
![[Pasted image 20260720150644.png]]

## Privilege Escalation

First thing to check are the privileges that Emily has with `whoami /priv`
![[Pasted image 20260720151002.png]]
- And as easy as this machine was we also find the `SeBackupPrivilege` permission enabled
- This permission is design to facilitate system backups and also access to system-protected files
- In our scenario that means we have access to the `SYSTEM` and `SAM` Windows Registry Hives, which contain the information needed for escalating our privileges

The `SAM` (Security Account Manager) hive contains local user account and group membership info (including hashed passwords)

The `SYSTEM` hive contains system config settings (including the system boot key required to decrypt the password hashes stored in `SAM`)

Using the `reg save` command we can get the information mentioned above:
![[Pasted image 20260720151718.png]]

Next we are going to download those 2 files to our local machine using the `download` command available in `evil-winrm`
![[Pasted image 20260720152326.png]]

Now we are going to use `Impacket's secretsdump` module to dump the user `NTLM` hashes
- The `NTLM` hash represents a cryptographic version of a user's plaintext password
- Once retrieved we can use it in a `Pass-The-Hash` attack to authenticate to the system without cracking the password
![[Pasted image 20260720152625.png]]
- `-sam` options specifies the `SAM` files containing the encrypted password data
- `-system` options specifies the `SYSTEM` file containing the decryption key
- `local` specifies that the files are local
- the hash we need is the Administrator hash, the one in the red box: `2b87e7c93a3e8a0ea4a581937016f341` 

Now we can pass the hash and log in to the Administrator account where I can find the System flag and finish the legendary Cicada puzzle
![[Pasted image 20260720153101.png]]
