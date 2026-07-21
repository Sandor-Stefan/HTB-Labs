- First I start with the `nmap` scan for enumeration
- The scan give me the following information![[Pasted image 20250813080241.png]]

- We have:
	- a web application on port `80`
	- a `ssh` service open on port `22` 
	- another web application on port `8080`

- Visiting the web app on port `80` give rather a simple web page
![[Pasted image 20250813080501.png]]

- If we click on the `Docs` button from the upper right side I am being forwarded to the web application `XWiki` which from the `nmap` scan I learned that is hosted on the port `8080`![[Pasted image 20250813080656.png]]

- At the bottom off the page I find the version of the `XWiki` app which is `XWiki Debian 15.10.8` ![[Pasted image 20250813081212.png]]

- Next thing, I search on the internet for an vulnerability/exploit for this version and I found it, this being the `CVE-2025-24893`

- I also found a GitHub repo with an exploit for this vulnerability https://github.com/gunzf0x/CVE-2025-24893
- I clone it on my machine and I run the command to get a reverse shell exactly as in the GitHub repo, only modifying the IP and PORT so I can catch it with the `nc` listener![[Pasted image 20250813082334.png]]

- On another terminal I opened started the listener on port `4444` and I got access as the `xwiki` user ![[Pasted image 20250813090816.png]]

- Now that I have access, I searched the `/` folder to find the location of  config file
```shell
find / -name *.cfg 2>/dev/null
```
![[Pasted image 20250813091817.png]]

- Going through the config file I find that there is a file, named `hibernate` that stores documents related to `XWiki`
![[Pasted image 20250813091542.png]]

- This file is `hibernate.cfg.xml` and is in the same folder `/usr/lib/xwiki/WEB-INF` as the `xwiki.cfg` 
- Parsing through the contents of the file I find a password used by the `xwiki` user for connection![[Pasted image 20250813092404.png]]

- Now that I have a password I `cd` into the `/home` folder and find the only user: `oliver` 

- Then I try to `ssh` into the `oliver` account using the password found the `hibernate.cfg.xml` 
```shell
ssh oliver@10.10.11.80
Password: theEd1t0rTeam99
```
![[Pasted image 20250813093233.png]]

## Root flag

- When I used the command `id` we can see that `oliver` is a part of the `netdata` group, which is very unusual 

- I search for vulnerabilities related to `netdata` and I found the following article, about the `ndsudo` local privilege escalation https://github.com/netdata/netdata/security/advisories/GHSA-pmhq-4cxq-wj93 
- On the target machine I found the `ndsudo` plugin in the `/opt/netdata/usr/libexec/netdata/plugins.d` 

- Related to the `ndsudo` vulnerability I found the `CVE-2024-32019` and with this I also found an GitHub repo for an exploit https://github.com/AliElKhatteb/CVE-2024-32019-POC

- Inside the repository I found all the instructions on what to do
- I modify the `exploit.c` file found in the repo so I can get a shell as root user
```C
#include <unistd.h>  // for setuid, setgid, execl
#include <stddef.h>  // for NULL

int main() {
    setuid(0);
    setgid(0);
    execl("/bin/bash", "/bin/bash", NULL);
    return 0;
}
```

- Then I compile the exploit on my machine
```shell
x86_64-linux-gnu-gcc -o nvme exploit.c -static
```

- I open a `python http server` to transfer `nvme` compiled binary file
- I make the `nvme` file an executable
- Finally I run the following command to exploit the vulnerable binary via `PATH` manipulation and I got access as the `root` user![[Pasted image 20250813095524.png]]

