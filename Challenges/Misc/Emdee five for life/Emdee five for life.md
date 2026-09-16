
**Challenge Scenario:**
- `Can you encrypt fast enough?`

We are provided with a link that hosts a website that gives us a string and we have to encrypt it using MD5 and then pass the encrypted string to get the flag
![[Pasted image 20260831153334.png]]
- The catch is that we have to use a script, otherwise if we encrypting it by copying the string and then encrypting it and then passing it we get the message `Too slow!`

Before creating the script we need to check how the encrypted string is passed in the POST request
- To do that I open the Web Developer Tools and go to the `Network` Tab to check the POST request, especially the data being sent
![[Pasted image 20260831154451.png]]
- The string is passed alongside the parameter `hash`

Now to solve the challenge the following script is used:
```python
import requests
import hashlib
from bs4 import BeautifulSoup

url = "http://154.57.164.78:31883"
request = requests.session()

page = request.get(url)
text = BeautifulSoup(page.content,"html.parser")

string = text.select('h3')[0].text
encrypted=hashlib.md5(string.encode('utf-8')).hexdigest()

data = {'hash':encrypted}

resp = request.post(url,data)
print(resp.text)
```

After running the script with `python ./script.py` I got the flag:
![[Pasted image 20260831155025.png]]
