
**Challenge Scenario**
```
Find the password (say PASS) and enter the flag in the form HTB{PASS}
```

The challenge gives us a zip archive containing an executable `EasyPass.exe`
![[Pasted image 20260908201110.png]]
- The executable is a `PE32` windows executable that has a GUI

To be able to see what the executable does I will use `wine` a tool that enables me to run windows executables in a Linux environment
```bash
wine EasyPass.exe
```
![[Pasted image 20260908201723.png]]
- The executable is very simple as it needs you to type a password click the button `Check Password` and it opens a pop up saying `Wrong Password` so from here I guess that I have to find out the correct password

For further exploiting I start to decompile the executable using `ghidra`
- I open up ghidra and start a new project and put the executable to an `Auto Analyze` process
![[Pasted image 20260908202816.png]]
- I can see that there is one function found with a name `entry` and a lot other but they are identified by the address they are found at

One thing that came to my mind is to check for strings found through the executable, especially for one of the `Enter Password`, `Check Password` or `Wrong Password` and maybe I will find the actual password

To make the search easier I will go to `Window -> Defined Strings` and search for `Password`
![[Pasted image 20260908203231.png]]
- I found the string `Wrong Password` in the data segment but this does not help much so maybe If I search for `References` I can find the string used somewhere

I right click the string go to `References -> Show References to Address`
![[Pasted image 20260908203430.png]]

I fount a reference of the string and when I checked I found out that it is used in some function called `FUN_00404628`
![[Pasted image 20260908203639.png]]

Checking the function I see that there are 2 parameters located in `EAX` and `EDX` 
![[Pasted image 20260908203833.png]]
- Since the `Wrong Password` string appears as a consequence to this function (indicated by the `JNZ LAB_00454144`) I believe that one of the 2 parameters hold the password we input and the other the correct password

Next to find the correct password we have to start the program in debug mode with the help of a debugger and set a breakpoint at `0x00454131` where the function gets called and check the value of the 2 registers, `EAX` and `EBX`

For that I will use `winedbg` that comes along `wine` in the same package

I start the `winedbg` debugger
```bash
winedbg --gdb EasyPass.exe
```

Then I set the break point and continue
```bash
wine-gdb> break *0x00454131
Breakpoint 1 at 0x454131
wine-gdb> c
Continuing.
```

Then the executable opens and I type a random password and hit `Check Password`
![[Pasted image 20260908204725.png]]

Next we check EAX and EDX
```bash
wine-gdb> x/s $eax
wine-gdb> x/s $edx
```
![[Pasted image 20260908204812.png]]
- As we can see EAX holds the random password we typed and `EDX` holds the string `fortran!` which I believe is the correct password

Upon typing the password `fortran!` another window pops up saying `Good Job. Congratulations` so the challenge is solved
![[Pasted image 20260908205006.png]]