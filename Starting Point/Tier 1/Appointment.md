
- **Task 1** - What does the acronym SQL stand for?
	- `Structured Query Language`

- **Task 2** - What is one of the most common type of SQL vulnerabilities?
	- `SQL Injection`

- **Task 3** - What is the 2021 OWASP Top 10 classification for this vulnerability?
	- `A03:2021-Injection`

- **Task 4** - What does nmap report as the service and version that are running on port 80 of the target?
	- `Apache httpd 2.4.38 ((Debian))`
![[Pasted image 20251117111146.png]]

- **Task 5** - What is the standard port used for the HTTPS protocol?
	- `port 443`

- **Task 6** - What is a folder called in web-application terminology?
	- `directory`

- **Task 7** - What is the HTTP response code which is given for `Not Found` errors?
	- response code `404`

- **Task 8** - What switch do we use with Gobuster to specify we're looking to discover directories and not subdomains?
	- switch - `dir`

- **Task 9** - What single character can be used to comment out the rest of a line in MySQL?
	- character - `#`

Checking the website we are met with a login form with a user and password field
- Because we are requested to try to exploit a SQL Injection using a comment I will try to put in the username field `admin' #` and a character in the password field
![[Pasted image 20251117111930.png]]
- After the login we are being redirected to the flag
![[Pasted image 20251117112220.png]]

- **Task 10** - If user input is not handled carefully, it could be interpreted as a comment. Use a comment to login as admin without knowing the password. What is the first word on the webpage returned?
	- `Congratulations`