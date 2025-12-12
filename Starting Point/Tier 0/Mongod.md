
We start directly with an `nmap` scan using `nmap -A -T4 -p1-65535 <IP_address>` to reveal the open ports

- **Task 1** - How many TCP ports are open on the machine?
	- The scan shows 2 port opens:
		- port 22
		- port 27017

- **Task 2** - Which service is running on port 27017 of the remote host?
	- `MongoDB 3.6.8`

- **Task 3** - What type of database is MongoDB?
	- `NoSQL`

For the next tasks I have to install the [MongoDB shell utility](https://www.mongodb.com/try/download/shell), but the version needs to be <= 2.3.2

- **Task 4** - What command is used to launch the interactive MongoDB shell from the terminal

To connect to the MongoDB server we use the command `mongosh mongodb://<IP_Address>:27017`

- **Task 5** - What is the command used for listing all the databases present on the MongoDB server?
	- `show dbs`

- **Task 6** - What is the command used for listing out the collections in a database?
	- `show collections`

- **Task 7** - What command is used to dump the content of all the documents within the collection named `flag`?
	- `db.flag.find()`