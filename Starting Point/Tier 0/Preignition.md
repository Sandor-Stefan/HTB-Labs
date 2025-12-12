
- **Task 1** - Directory Brute-Forcing is a technique used to check a lot of paths on a web server to find hidden pages. Which is another name for this?
	1. Local File Inclusion
	2. Dir Busting
	3. Hash Cracking
	- Answer: `Dir busting`

- **Task 2** - What switch do we use for nmap's scan to specify that we want to perform version detection?
	- `-sV` flag for version detection


For the next tasks we use  `nmap -sC -sV <IP_address>` to scan the open ports

- **Task 3** - What does nmap report is the service identified as running on port 80/tcp?
	- `HTTP`

- **Task 4** - What server name and version of service is running on port 80/tcp?
	- `nginx 1.14.2`

For the next steps we gonna use `gobuster`

- **Task 5** - What switch do we use to specify to `gobuster` we want to perform Dir Busting specifically?
	- `dir` 

- **Task 6** - When using `gobuster` to dir bust, what switch do we add to make sure it finds PHP pages?
	- `-x php`

For the next tasks we use `gobuster dir -u <IP_address> -w <wordlist> -x php`

- **Task 7** - What page is found during our dir busting activities>
	- `admin.php`

- **Task 8** - What is the HTTP status code reported by `gobuster` for the discovered page
	- `200` (stands for OK, meaning we have access to the web page)

- **Flag** - To find the flag we must go on our browser to `<IP_addrress>/admin.php`  and login with the following credentials
	- user: admin
	- password: admin

