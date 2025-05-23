
1. The acronym SMB stands for `Server Message Block`
2. SMB operates on `port 445`
3. The service name running on the port 445 is `microsoft-ds`
4. The option used with smbclient to list the available shares on the target is `-`
5. There are `4 shares` on the target
	- To find the number we use `smbclient -L <target_IP_address>` and wee count all the listed shares
6. The name of the share we are able to access in the end with no password is `WorkShare`
- To connect to the share we use: `smbclient \\\\<target_IP_address>\\WorkShare`
7. To download a file through SMB shell we use `get`
	- `get <filename>`

