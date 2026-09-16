**Challenge Scenario**
```
Are you able to cheat me and get the flag?
```

We are given an executable `impossible_password.bin` and we have to work with it to get the flag

First time running the script, I noticed that it takes an input and the repeats it inside `[]` closed brackets
![[Pasted image 20260915193625.png]]
- So most probably the goal is to find the password

First thing I can try is to disassemble and decompile the code and for that I will use ghidra

In the symbol tree one function especially holds gets my attention
![[Pasted image 20260915194024.png]]

This function holds the code of the binary and we can uncover what it does behind the curtain
```C
void FUN_0040085d(void)

{
  int iVar1;
  char *__s2;
  undefined1 local_48;
  undefined1 local_47;
  undefined1 local_46;
  undefined1 local_45;
  undefined1 local_44;
  undefined1 local_43;
  undefined1 local_42;
  undefined1 local_41;
  undefined1 local_40;
  undefined1 local_3f;
  undefined1 local_3e;
  undefined1 local_3d;
  undefined1 local_3c;
  undefined1 local_3b;
  undefined1 local_3a;
  undefined1 local_39;
  undefined1 local_38;
  undefined1 local_37;
  undefined1 local_36;
  undefined1 local_35;
  char local_28 [20];
  int local_14;
  char *local_10;
  
  local_10 = "SuperSeKretKey";
  local_48 = 0x41;
  local_47 = 0x5d;
  local_46 = 0x4b;
  local_45 = 0x72;
  local_44 = 0x3d;
  local_43 = 0x39;
  local_42 = 0x6b;
  local_41 = 0x30;
  local_40 = 0x3d;
  local_3f = 0x30;
  local_3e = 0x6f;
  local_3d = 0x30;
  local_3c = 0x3b;
  local_3b = 0x6b;
  local_3a = 0x31;
  local_39 = 0x3f;
  local_38 = 0x6b;
  local_37 = 0x38;
  local_36 = 0x31;
  local_35 = 0x74;
  printf("* ");
  __isoc99_scanf(&DAT_00400a82,local_28);
  printf("[%s]\n",local_28);
  local_14 = strcmp(local_28,local_10);
  if (local_14 != 0) {
                    /* WARNING: Subroutine does not return */
    exit(1);
  }
  printf("** ");
  __isoc99_scanf(&DAT_00400a82,local_28);
  __s2 = (char *)FUN_0040078d(0x14);
  iVar1 = strcmp(local_28,__s2);
  if (iVar1 == 0) {
    FUN_00400978(&local_48);
  }
  return;
}
```

The flow of the function is as follows:
1. First, it reads the input we provide and compares it to the `local_10` string which holds the value `SuperSeKretKey`
2. If the 2 strings are identical it gets another input from us and compares it to the string `__s2` which is the result of another function
3. If this match as well it goes to the `FUN_00400978` function which takes as a parameter the address of the `local_48` variable
	- This function most probably prints the flag 

Since we got the value of the first key, the only thing remaining is to find the 2nd key value so we have to check the `FUN_0040078d` function
![[Pasted image 20260915194909.png]]
- Unfortunately as it can be seen the construction of the string is quite complicated as it uses for every character the formula `'!' + rand() % 0x5e` and the `rand()` function is seeded using the `time()` function, meaning that is  very difficult to reconstruct the key

Since finding the 2nd key is out of discussion we go back to the function `FUN_00400978` which prints the flag
![[Pasted image 20260915195233.png]]
- The construction of the flag is easier as it takes 14 bytes from the address given as a parameter or until one of the bytes is `0x09` then for each character it does the XOR operation of the character with 9

Since the parameter is `local_48` it means that it starts from its value `0x41` and it goes until the `local_35` variable address
- All this happens because the variables are stored in memory one after another having consecutive addresses
![[Pasted image 20260915195827.png]]

Getting all the values from `local_48` to `local_35` and decoding them we get the following string `A]Kr=9k0=0o0;k1?k81t`

Now that we have the secret string and we got the algorithm behind creating the flag we can use 2 lines of python to get the actual flag
```python
var=b"A]Kr=9k0=0o0;k1?k81t"
flag=bytes(b ^ 9 for b in var)
print(flag.decode())
```
