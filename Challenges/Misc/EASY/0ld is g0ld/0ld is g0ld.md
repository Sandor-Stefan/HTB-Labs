
**Challenge Scenario**
```
Old algorithms are not a waste, but they are really precious
```

We are given a pdf file `0ld is g0ld.pdf` but when trying to open it we encounter a prompt saying we have to input a password
![Pasted image 20260916185731](Images/Pasted%20image%2020260916185731.png)

The password can be cracked using the `pdfcrack` tool
```bash
pdfcrack -f 0ld is gold.pdf -w /usr/share/wordlists/rockyou.txt
```
- After 1-2 minutes it gives us the user password, being `jumanji69`

The pdf consists of one page only, having the title `Remember Me?` and a picture of some person at the top, but also something appears to be at the bottom of the page
![Pasted image 20260916190018](Images/Pasted%20image%2020260916190018.png)


Zooming in we can distinguish a morse code
![Pasted image 20260916190054](Images/Pasted%20image%2020260916190054.png)

Next step is to translate the morse code using a any tool on the internet that can do that
![Pasted image 20260916190149](Images/Pasted%20image%2020260916190149.png)
- It gives us a string and the only thing left for us to do is to put it inside `HTB{<string>}` to make a HTB flag

**Fun Fact**
- The person inside the photo is the inventor of the morse code: **Samuel Morse**