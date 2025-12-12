
- **Task 1** - What is the default port for `rsync`?
	- `port 873`

Next we use `nmap -sC -sV <IP_address>` to enumerate the open ports

- **Task 2** - How many TCP ports are open on the remote host?
	- There is 1 open port

- **Task 3** - What is the protocol version used by `rsync` on the remote machine?
	- `protocol version 31`

- **Task 4** - What is the most common command name on Linux to interact with `rsync`?
	- `rsync`

- **Task 5** - What credentials do you have to pass to `rsync` in order to use anonymous authentication?
	- `None`

- **Task 6** - What is the option to only list shares and files on `rsync`
	- `--list-only`

- To list shares we use the command `rsync <IP_address>::public --list-only`
- Next to download a certain share from the target we use `rsync <IP_address>::public/<target_file> <file_location_on_our_machine>`
