
**Challenge Scenario:**
- `After struggling to secure our secret strings for a long time, we finally figured out the solution to our problem: Make decompilation harder. It should now be impossible to figure out how our programs work!`

The challenge gave me a zip archive containing an executable so first I decided to run it to see what I have to do:
![[Pasted image 20260830174348.png]]
- from the output, my guess is that I have to find the password inside the binary and then pass it as an argument 

To check the content of the binary I simply used the command `cat ./behindthescenes`

Searching for the password proved to be easy, as I found some strings that led me finding the following strings 
![[Pasted image 20260830174734.png]]
- `./challenge <password>Itz_0nly_UD2> HTB{%s}`
- From the findings above I realized that the password can only be the `Itz_0nly_UD2` string since it is preceded by the `<password>` string and succeeded by the `HTB{%s}` which it is similar to a print statement

Trying the password as an argument it led me to the discovery of the flag:
![[Pasted image 20260830175058.png]]

