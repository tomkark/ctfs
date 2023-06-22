```c
#include <stdio.h>

int main(){
	unsigned int random;
	random = rand();	// random value!

	unsigned int key=0;
	scanf("%d", &key);

	if( (key ^ random) == 0xdeadbeef ){
		printf("Good!\n");
		system("/bin/cat flag");
		return 0;
	}

	printf("Wrong, maybe you should try 2^32 cases.\n");
	return 0;
}
```


Welp, if random is a random number each run, how can we predict it to know what to xor it with?
Let's run a test program to see rand() actually working.

```c
#include <stdio.h>


int main (int argc, char *argv[])
{
    unsigned int random = rand();
    printf("%d", random);

    return 0;
}
```
I compiled it and ran it multiple times and to no surprise it printed out the same number over and over again. This happens since we did not set a seed, so it uses the same seed (1) each run. This means we literally know the key to our solution, we just need to see the random number inside the virtual machine and then enter that number ^ `0xdeadbeef`.

```bash
(gdb) disas main
Dump of assembler code for function main:
   0x00000000004005f4 <+0>:	push   %rbp
   0x00000000004005f5 <+1>:	mov    %rsp,%rbp
   0x00000000004005f8 <+4>:	sub    $0x10,%rsp
   0x00000000004005fc <+8>:	mov    $0x0,%eax
   0x0000000000400601 <+13>:	callq  0x400500 <rand@plt>
   0x0000000000400606 <+18>:	mov    %eax,-0x4(%rbp)
   0x0000000000400609 <+21>:	movl   $0x0,-0x8(%rbp)
   0x0000000000400610 <+28>:	mov    $0x400760,%eax
   0x0000000000400615 <+33>:	lea    -0x8(%rbp),%rdx
   0x0000000000400619 <+37>:	mov    %rdx,%rsi
   0x000000000040061c <+40>:	mov    %rax,%rdi
   0x000000000040061f <+43>:	mov    $0x0,%eax
   0x0000000000400624 <+48>:	callq  0x4004f0 <__isoc99_scanf@plt>
   0x0000000000400629 <+53>:	mov    -0x8(%rbp),%eax
   0x000000000040062c <+56>:	xor    -0x4(%rbp),%eax
   0x000000000040062f <+59>:	cmp    $0xdeadbeef,%eax
   0x0000000000400634 <+64>:	jne    0x400656 <main+98>
   0x0000000000400636 <+66>:	mov    $0x400763,%edi
   0x000000000040063b <+71>:	callq  0x4004c0 <puts@plt>
   0x0000000000400640 <+76>:	mov    $0x400769,%edi
   0x0000000000400645 <+81>:	mov    $0x0,%eax
   0x000000000040064a <+86>:	callq  0x4004d0 <system@plt>
   0x000000000040064f <+91>:	mov    $0x0,%eax
   0x0000000000400654 <+96>:	jmp    0x400665 <main+113>
   0x0000000000400656 <+98>:	mov    $0x400778,%edi
   0x000000000040065b <+103>:	callq  0x4004c0 <puts@plt>
   0x0000000000400660 <+108>:	mov    $0x0,%eax
   0x0000000000400665 <+113>:	leaveq 
   0x0000000000400666 <+114>:	retq   
End of assembler dump.
(gdb) b *0x0000000000400609
Breakpoint 1 at 0x400609
(gdb) r
Starting program: /home/random/random 
/bin/bash: warning: setlocale: LC_ALL: cannot change locale (en_IN.UTF-8)

Breakpoint 1, 0x0000000000400609 in main ()
(gdb) p $rbp-0x4
$1 = (void *) 0x7ffcce12485c
(gdb) p *0x7ffcce12485c
$2 = 1804289383
```

Then our key is `1804289383 ^ 0xdeadbeef = 3039230856`.
