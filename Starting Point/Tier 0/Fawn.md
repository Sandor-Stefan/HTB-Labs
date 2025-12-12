
1. The acronym FTP stands for `File Transfer Protcol`
2. Usually the FTP service listens on `port 21`
3. the acronym of the protocol designed to provide similar functionality to FTP but securely, as an extension of the SSH protocol is `SFTP`
4. The command used to send an ICMP echo request to a target is `ping`

5. The version of FTP running on the target is `vsFTPd 3.0.3`
	- this was done by performing an nmap scan `nmap -sC <target_IP_address>`

6. The type of the OS running on the target is `Unix`
	- this was done by performing an nmap scan `nmap -A <target_IP_address>`

7. In order to display the `ftp` client help menu we run the command `ftp -h`
	- this was done by searching the manual

8. The username that is used over FTP when you want to log in without having an account is `anonymous`
	- When we run the `nmap -sC` scan we find out that "Anonymous FTP login is allowed" so from there we realize that this is the username

9. The response code we get for the FTP message "Login successful" is `230`

10. The command we can use to list the files available on the FTP server is `ls`

11. The command used to download files on FTP servers is `get`

12. To find the flag we download the file `get flag.txt` then we check its content `cat flag.txt` and we find the flag