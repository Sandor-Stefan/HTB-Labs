
**Challenge Scenario**
```
A Junior Developer just switched to a new source control platform
Can you fin the secret token?
```

The challenge gives me a folder `Illumination.JS` which contains 2 files `bot.js` and `config.json` and another directory `.git`

First I decide to check the `bot.js` directory
- Apparently it holds the source code for a Discord Bot
- What may help us solving the challenge lies at the end of the file, exactly the last line of code:
```JS
client.login(Buffer.from(config.token, 'base64').toString('ascii')) //Login with secret token
```
- The line holds the function for the login, which takes a variable token from the config structure and decodes it from base64 to ascii 
- Also the comment suggest that we have to find this secret token 

Looking at the beginning to see from where does this config comes, I learn that the app parses the `config.json` file and gets the values from there

As it happens we also got that `config.json` and we shall check its content
```json
{
	"token": "Replace me with token when in use! Security Risk!",
	"prefix": "~",
	"lightNum": "1337",
	"username": "UmVkIEhlcnJpbmcsIHJlYWQgdGhlIEpTIGNhcmVmdWxseQ==",
	"host": "127.0.0.1"
}
```
- The token parameter holds a message stating that we should use the token only when we run the program

The only piece remaining is the `.git` directory which is a git repo 

I `cd` into that directory and then I check its content
![[Pasted image 20260916133757.png]]
- a lot of files and also I can see the `bot.js` and `config.json` file so maybe there I will find the token

The token from the newly found `config.json` file is the same as before so no luck

A good thing when checking a git repository is to check the previous commits to see if any of them holds something different
- To achieve that I run the command `git log` to see the log of the repo
![[Pasted image 20260916133954.png]]
- The commits are shown from the newest at the top of the file to the oldest at the bottom of the file

The commit `47241a47f...` holds an interesting message
- we can learn that in this commit, the developer removed the token for security reasons
- Knowing that the following steps should involve to change to the version previous this one (meaning we should switch to commit `ddc606f8f..`) and check the content of the `config.json` file

To go to a previous command I use the following command
```bash
git checkout ddc606f8fa05c363ea4de20f31834e97dd527381
```

Unfortunately when running this command we get an error:
```bash
fatal: this operation must be run in a work tree
```

To solve the error we should specify with the argument `--work-tree` alongside the current folder location and then run the command we want
```bash
git --work-tree . checkout ddc606f8fa05c363ea4de20f31834e97dd527381
```
![[Pasted image 20260916134734.png]]
- The process was successful so now let's check the `config.json` file
![[Pasted image 20260916134943.png]]
- We got a new value for the `token` parameter so we only need to decode it from base64 to ascii
```bash
echo '<token>' | base64 -d

HTB{...}
```
