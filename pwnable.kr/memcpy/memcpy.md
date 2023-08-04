```c
// compiled with : gcc -o memcpy memcpy.c -m32 -lm
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <sys/mman.h>
#include <math.h>

unsigned long long rdtsc(){
        asm("rdtsc");
}

char* slow_memcpy(char* dest, const char* src, size_t len){
	int i;
	for (i=0; i<len; i++) {
		dest[i] = src[i];
	}
	return dest;
}

char* fast_memcpy(char* dest, const char* src, size_t len){
	size_t i;
	// 64-byte block fast copy
	if(len >= 64){
		i = len / 64;
		len &= (64-1);
		while(i-- > 0){
			__asm__ __volatile__ (
			"movdqa (%0), %%xmm0\n"
			"movdqa 16(%0), %%xmm1\n"
			"movdqa 32(%0), %%xmm2\n"
			"movdqa 48(%0), %%xmm3\n"
			"movntps %%xmm0, (%1)\n"
			"movntps %%xmm1, 16(%1)\n"
			"movntps %%xmm2, 32(%1)\n"
			"movntps %%xmm3, 48(%1)\n"
			::"r"(src),"r"(dest):"memory");
			dest += 64;
			src += 64;
		}
	}

	// byte-to-byte slow copy
	if(len) slow_memcpy(dest, src, len);
	return dest;
}

int main(void){

	setvbuf(stdout, 0, _IONBF, 0);
	setvbuf(stdin, 0, _IOLBF, 0);

	printf("Hey, I have a boring assignment for CS class.. :(\n");
	printf("The assignment is simple.\n");

	printf("-----------------------------------------------------\n");
	printf("- What is the best implementation of memcpy?        -\n");
	printf("- 1. implement your own slow/fast version of memcpy -\n");
	printf("- 2. compare them with various size of data         -\n");
	printf("- 3. conclude your experiment and submit report     -\n");
	printf("-----------------------------------------------------\n");

	printf("This time, just help me out with my experiment and get flag\n");
	printf("No fancy hacking, I promise :D\n");

	unsigned long long t1, t2;
	int e;
	char* src;
	char* dest;
	unsigned int low, high;
	unsigned int size;
	// allocate memory
	char* cache1 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
	char* cache2 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
	src = mmap(0, 0x2000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

	size_t sizes[10];
	int i=0;

	// setup experiment parameters
	for(e=4; e<14; e++){	// 2^13 = 8K
		low = pow(2,e-1);
		high = pow(2,e);
		printf("specify the memcpy amount between %d ~ %d : ", low, high);
		scanf("%d", &size);
		if( size < low || size > high ){
			printf("don't mess with the experiment.\n");
			exit(0);
		}
		sizes[i++] = size;
	}

	sleep(1);
	printf("ok, lets run the experiment with your configuration\n");
	sleep(1);

	// run experiment
	for(i=0; i<10; i++){
		size = sizes[i];
		printf("experiment %d : memcpy with buffer size %d\n", i+1, size);
		dest = malloc( size );

		memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
		t1 = rdtsc();
		slow_memcpy(dest, src, size);		// byte-to-byte memcpy
		t2 = rdtsc();
		printf("ellapsed CPU cycles for slow_memcpy : %llu\n", t2-t1);

		memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
		t1 = rdtsc();
		fast_memcpy(dest, src, size);		// block-to-block memcpy
		t2 = rdtsc();
		printf("ellapsed CPU cycles for fast_memcpy : %llu\n", t2-t1);
		printf("\n");
	}

	printf("thanks for helping my experiment!\n");
	printf("flag : ----- erased in this source code -----\n");
	return 0;
}
```

We have a cool experiment here testing an implementation of a 'faster' memcpy vs a slow implementation of it.
The test asks the user to choose a size that represents bytes of data to compare the two implementations, it does it 10 times for sizes in the range of [8,16], [16,32], [32, 64] ..... [4096, 8192]

So let's try to run and see the results:
```bash
memcpy@pwnable:~$ nc 0 9022 
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : 8
specify the memcpy amount between 16 ~ 32 : 16
specify the memcpy amount between 32 ~ 64 : 32
specify the memcpy amount between 64 ~ 128 : 64 
specify the memcpy amount between 128 ~ 256 : 128
specify the memcpy amount between 256 ~ 512 : 256
specify the memcpy amount between 512 ~ 1024 : 512
specify the memcpy amount between 1024 ~ 2048 : 1024
specify the memcpy amount between 2048 ~ 4096 : 2048
specify the memcpy amount between 4096 ~ 8192 : 4096
ok, lets run the experiment with your configuration
experiment 1 : memcpy with buffer size 8
ellapsed CPU cycles for slow_memcpy : 2046
ellapsed CPU cycles for fast_memcpy : 188

experiment 2 : memcpy with buffer size 16
ellapsed CPU cycles for slow_memcpy : 182
ellapsed CPU cycles for fast_memcpy : 226

experiment 3 : memcpy with buffer size 32
ellapsed CPU cycles for slow_memcpy : 350
ellapsed CPU cycles for fast_memcpy : 280

experiment 4 : memcpy with buffer size 64
ellapsed CPU cycles for slow_memcpy : 504
ellapsed CPU cycles for fast_memcpy : 136

experiment 5 : memcpy with buffer size 128
ellapsed CPU cycles for slow_memcpy : 952
```

Looks like it crashed, let's try to compile the source ourselves using the given flags at the first line of the source.
`gcc -o memcpy memcpy.c -m32 -lm`

```bash
memcpy@pwnable:~$ mkdir /tmp/writeup
memcpy@pwnable:~$ cd /tmp/writeup
memcpy@pwnable:/tmp/writeup$ gcc -o memcpy /home/memcpy/memcpy.c -m32 -lm
memcpy@pwnable:/tmp/writeup$ cat - > input
8
16
32
64
128
256
512
1024
2048
4096
^C  
memcpy@pwnable:/tmp/writeup$ ./memcpy < input 
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : specify the memcpy amount between 16 ~ 32 : specify the memcpy amount between 32 ~ 64 : specify the memcpy amount between 64 ~ 128 : specify the memcpy amount between 128 ~ 256 : specify the memcpy amount between 256 ~ 512 : specify the memcpy amount between 512 ~ 1024 : specify the memcpy amount between 1024 ~ 2048 : specify the memcpy amount between 2048 ~ 4096 : specify the memcpy amount between 4096 ~ 8192 : ok, lets run the experiment with your configuration
experiment 1 : memcpy with buffer size 8
ellapsed CPU cycles for slow_memcpy : 2104
ellapsed CPU cycles for fast_memcpy : 240

experiment 2 : memcpy with buffer size 16
ellapsed CPU cycles for slow_memcpy : 290
ellapsed CPU cycles for fast_memcpy : 244

experiment 3 : memcpy with buffer size 32
ellapsed CPU cycles for slow_memcpy : 298
ellapsed CPU cycles for fast_memcpy : 388

experiment 4 : memcpy with buffer size 64
ellapsed CPU cycles for slow_memcpy : 550
ellapsed CPU cycles for fast_memcpy : 130

experiment 5 : memcpy with buffer size 128
ellapsed CPU cycles for slow_memcpy : 1078
Segmentation fault (core dumped)
```

So we crash because of a segfault, let's try to gdb this thing and find out where it crashes.

```bash
memcpy@pwnable:/tmp/writeup$ gdb memcpy 
(gdb) r < input
Starting program: /tmp/writeup/memcpy < input
/bin/bash: warning: setlocale: LC_ALL: cannot change locale (en_IN.UTF-8)
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : specify the memcpy amount between 16 ~ 32 : specify the memcpy amount between 32 ~ 64 : specify the memcpy amount between 64 ~ 128 : specify the memcpy amount between 128 ~ 256 : specify the memcpy amount between 256 ~ 512 : specify the memcpy amount between 512 ~ 1024 : specify the memcpy amount between 1024 ~ 2048 : specify the memcpy amount between 2048 ~ 4096 : specify the memcpy amount between 4096 ~ 8192 : ok, lets run the experiment with your configuration
experiment 1 : memcpy with buffer size 8
ellapsed CPU cycles for slow_memcpy : 2140
ellapsed CPU cycles for fast_memcpy : 240

experiment 2 : memcpy with buffer size 16
ellapsed CPU cycles for slow_memcpy : 240
ellapsed CPU cycles for fast_memcpy : 178

experiment 3 : memcpy with buffer size 32
ellapsed CPU cycles for slow_memcpy : 340
ellapsed CPU cycles for fast_memcpy : 354

experiment 4 : memcpy with buffer size 64
ellapsed CPU cycles for slow_memcpy : 654
ellapsed CPU cycles for fast_memcpy : 130

experiment 5 : memcpy with buffer size 128
ellapsed CPU cycles for slow_memcpy : 1020

Program received signal SIGSEGV, Segmentation fault.
0x080487cc in fast_memcpy ()
```
Let's check the backtrace and find where it stopped.
```bash
(gdb) bt
#0  0x080487cc in fast_memcpy ()
#1  0x08048b65 in main ()
(gdb) disas fast_memcpy 
Dump of assembler code for function fast_memcpy:
   0x08048798 <+0>:	push   %ebp
   0x08048799 <+1>:	mov    %esp,%ebp
   0x0804879b <+3>:	sub    $0x10,%esp
   0x0804879e <+6>:	cmpl   $0x3f,0x10(%ebp)
   0x080487a2 <+10>:	jbe    0x80487f0 <fast_memcpy+88>
   0x080487a4 <+12>:	mov    0x10(%ebp),%eax
   0x080487a7 <+15>:	shr    $0x6,%eax
   0x080487aa <+18>:	mov    %eax,-0x4(%ebp)
   0x080487ad <+21>:	andl   $0x3f,0x10(%ebp)
   0x080487b1 <+25>:	jmp    0x80487e3 <fast_memcpy+75>
   0x080487b3 <+27>:	mov    0xc(%ebp),%eax
   0x080487b6 <+30>:	mov    0x8(%ebp),%edx
   0x080487b9 <+33>:	movdqa (%eax),%xmm0
   0x080487bd <+37>:	movdqa 0x10(%eax),%xmm1
   0x080487c2 <+42>:	movdqa 0x20(%eax),%xmm2
   0x080487c7 <+47>:	movdqa 0x30(%eax),%xmm3
=> 0x080487cc <+52>:	movntps %xmm0,(%edx)
   0x080487cf <+55>:	movntps %xmm1,0x10(%edx)
   0x080487d3 <+59>:	movntps %xmm2,0x20(%edx)
   0x080487d7 <+63>:	movntps %xmm3,0x30(%edx)
   0x080487db <+67>:	addl   $0x40,0x8(%ebp)
   0x080487df <+71>:	addl   $0x40,0xc(%ebp)
   0x080487e3 <+75>:	mov    -0x4(%ebp),%eax
   0x080487e6 <+78>:	lea    -0x1(%eax),%edx
   0x080487e9 <+81>:	mov    %edx,-0x4(%ebp)
   0x080487ec <+84>:	test   %eax,%eax
   0x080487ee <+86>:	jne    0x80487b3 <fast_memcpy+27>
   0x080487f0 <+88>:	cmpl   $0x0,0x10(%ebp)
   0x080487f4 <+92>:	je     0x8048807 <fast_memcpy+111>
   0x080487f6 <+94>:	pushl  0x10(%ebp)
   0x080487f9 <+97>:	pushl  0xc(%ebp)
   0x080487fc <+100>:	pushl  0x8(%ebp)
   0x080487ff <+103>:	call   0x8048763 <slow_memcpy>
   0x08048804 <+108>:	add    $0xc,%esp
   0x08048807 <+111>:	mov    0x8(%ebp),%eax
   0x0804880a <+114>:	leave  
   0x0804880b <+115>:	ret    
End of assembler dump.
```

Ok so it crashed in one of the inserted instructions at `fast_memcpy`, but what does `movntps` do?

> Moves the packed single precision floating-point values in the source operand (second operand) to the destination operand (first operand) using a non-temporal hint to prevent caching of the data during the write to memory. The source operand is an XMM register, YMM register or ZMM register, which is assumed to contain packed single precision, floating-pointing. The destination operand is a 128-bit, 256-bit or 512-bit memory location. The memory operand must be aligned on a 16-byte (128-bit version), 32-byte (VEX.256 encoded version) or 64-byte (EVEX.512 encoded version) boundary otherwise a general-protection exception (#GP) will be generated.

One of the requirements for this command requires the memory operand to be aligned on a 16 byte boundary. let's examine `edx`:
```bash
(gdb) p/x $edx
$1 = 0x804d0a8
```

So first of all it is not aligned on 16 bytes, second of all, it's important to understand what high-level variable is represented by edx

```c
			__asm__ __volatile__ (
			"movdqa (%0), %%xmm0\n"
			"movdqa 16(%0), %%xmm1\n"
			"movdqa 32(%0), %%xmm2\n"
			"movdqa 48(%0), %%xmm3\n"
			"movntps %%xmm0, (%1)\n"
			"movntps %%xmm1, 16(%1)\n"
			"movntps %%xmm2, 32(%1)\n"
			"movntps %%xmm3, 48(%1)\n"
			::"r"(src),"r"(dest):"memory");
			dest += 64;
			src += 64;
```

><i>Read further on __asm__ to understand this syntax and usage. https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html</i>

We can clearly see it is the `dest`, it is malloc'd each iteration according to our input given at the start of the program. 
If we add 8 to the previous iteration, the next malloc will return a pointer starting 8 bytes higher, and then it will be aligned.

We know it crashed on 128 bytes from the program's output, now we need to add 8 to 64 = 72.
Let's try it:
```bash
memcpy@pwnable:/tmp/writeup$ cat - > input
8
16
32
72
128
256
512
1024
2048
4096
^C
memcpy@pwnable:/tmp/writeup$ ./memcpy < input
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : specify the memcpy amount between 16 ~ 32 : specify the memcpy amount between 32 ~ 64 : specify the memcpy amount between 64 ~ 128 : specify the memcpy amount between 128 ~ 256 : specify the memcpy amount between 256 ~ 512 : specify the memcpy amount between 512 ~ 1024 : specify the memcpy amount between 1024 ~ 2048 : specify the memcpy amount between 2048 ~ 4096 : specify the memcpy amount between 4096 ~ 8192 : ok, lets run the experiment with your configuration
experiment 1 : memcpy with buffer size 8
ellapsed CPU cycles for slow_memcpy : 2256
ellapsed CPU cycles for fast_memcpy : 224

experiment 2 : memcpy with buffer size 16
ellapsed CPU cycles for slow_memcpy : 312
ellapsed CPU cycles for fast_memcpy : 286

experiment 3 : memcpy with buffer size 32
ellapsed CPU cycles for slow_memcpy : 352
ellapsed CPU cycles for fast_memcpy : 460

experiment 4 : memcpy with buffer size 72
ellapsed CPU cycles for slow_memcpy : 704
ellapsed CPU cycles for fast_memcpy : 190

experiment 5 : memcpy with buffer size 128
ellapsed CPU cycles for slow_memcpy : 998
ellapsed CPU cycles for fast_memcpy : 126

experiment 6 : memcpy with buffer size 256
ellapsed CPU cycles for slow_memcpy : 2390
Segmentation fault (core dumped)
```
We managed to pass the iteration we crashed on previously! so this really is the fix for our issue.
I will add a print of the address of `dest` each time so we'll know the offset we need to add to fix each iteration.

```c
[...]
                size = sizes[i];
                printf("experiment %d : memcpy with buffer size %d\n", i+1, size);
                dest = malloc( size );
                printf("Address of dest: 0x%p\n", dest);
[...]

```bash
memcpy@pwnable:/tmp/writeup$ ./memcpy < input
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : specify the memcpy amount between 16 ~ 32 : specify the memcpy amount between 32 ~ 64 : specify the memcpy amount between 64 ~ 128 : specify the memcpy amount between 128 ~ 256 : specify the memcpy amount between 256 ~ 512 : specify the memcpy amount between 512 ~ 1024 : specify the memcpy amount between 1024 ~ 2048 : specify the memcpy amount between 2048 ~ 4096 : specify the memcpy amount between 4096 ~ 8192 : ok, lets run the experiment with your configuration
experiment 1 : memcpy with buffer size 8
Address of dest: 0x0x943e010
ellapsed CPU cycles for slow_memcpy : 2308
ellapsed CPU cycles for fast_memcpy : 238

experiment 2 : memcpy with buffer size 16
Address of dest: 0x0x943e020
ellapsed CPU cycles for slow_memcpy : 186
ellapsed CPU cycles for fast_memcpy : 274

experiment 3 : memcpy with buffer size 32
Address of dest: 0x0x943e038
ellapsed CPU cycles for slow_memcpy : 286
ellapsed CPU cycles for fast_memcpy : 384

experiment 4 : memcpy with buffer size 72
Address of dest: 0x0x943e060
ellapsed CPU cycles for slow_memcpy : 616
ellapsed CPU cycles for fast_memcpy : 216

experiment 5 : memcpy with buffer size 128
Address of dest: 0x0x943e0b0
ellapsed CPU cycles for slow_memcpy : 1042
ellapsed CPU cycles for fast_memcpy : 130

experiment 6 : memcpy with buffer size 256
Address of dest: 0x0x943e138
ellapsed CPU cycles for slow_memcpy : 1688
```
We need to add 8 again to the previous iteration, it also looks like it's always in that offset, so we should try to add 8 to everything after 32.

```bash
memcpy@pwnable:/tmp/writeup$ cat - > input
8 
16
32
72
136
264
520
1032
2056
4102
^C
memcpy@pwnable:/tmp/writeup$ ./memcpy < input
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : specify the memcpy amount between 16 ~ 32 : specify the memcpy amount between 32 ~ 64 : specify the memcpy amount between 64 ~ 128 : specify the memcpy amount between 128 ~ 256 : specify the memcpy amount between 256 ~ 512 : specify the memcpy amount between 512 ~ 1024 : specify the memcpy amount between 1024 ~ 2048 : specify the memcpy amount between 2048 ~ 4096 : specify the memcpy amount between 4096 ~ 8192 : ok, lets run the experiment with your configuration
experiment 1 : memcpy with buffer size 8
Address of dest: 0x0x9f55010
ellapsed CPU cycles for slow_memcpy : 2180
ellapsed CPU cycles for fast_memcpy : 198

experiment 2 : memcpy with buffer size 16
Address of dest: 0x0x9f55020
ellapsed CPU cycles for slow_memcpy : 260
ellapsed CPU cycles for fast_memcpy : 248

experiment 3 : memcpy with buffer size 32
Address of dest: 0x0x9f55038
ellapsed CPU cycles for slow_memcpy : 322
ellapsed CPU cycles for fast_memcpy : 394

experiment 4 : memcpy with buffer size 72
Address of dest: 0x0x9f55060
ellapsed CPU cycles for slow_memcpy : 612
ellapsed CPU cycles for fast_memcpy : 186

experiment 5 : memcpy with buffer size 136
Address of dest: 0x0x9f550b0
ellapsed CPU cycles for slow_memcpy : 1012
ellapsed CPU cycles for fast_memcpy : 176

experiment 6 : memcpy with buffer size 264
Address of dest: 0x0x9f55140
ellapsed CPU cycles for slow_memcpy : 9858
ellapsed CPU cycles for fast_memcpy : 270

experiment 7 : memcpy with buffer size 520
Address of dest: 0x0x9f55250
ellapsed CPU cycles for slow_memcpy : 3858
ellapsed CPU cycles for fast_memcpy : 290

experiment 8 : memcpy with buffer size 1032
Address of dest: 0x0x9f55460
ellapsed CPU cycles for slow_memcpy : 32696
ellapsed CPU cycles for fast_memcpy : 402

experiment 9 : memcpy with buffer size 2056
Address of dest: 0x0x9f55870
ellapsed CPU cycles for slow_memcpy : 56520
ellapsed CPU cycles for fast_memcpy : 904

experiment 10 : memcpy with buffer size 4102
Address of dest: 0x0x9f56080
ellapsed CPU cycles for slow_memcpy : 127334
ellapsed CPU cycles for fast_memcpy : 1410

thanks for helping my experiment!
flag : ----- erased in this source code -----
```

It worked! if we run it on the actual binary running under the intended privilege we should get the flag:

```bash
memcpy@pwnable:/tmp/writeup$ nc 0 9022 < input 
Hey, I have a boring assignment for CS class.. :(
The assignment is simple.
-----------------------------------------------------
- What is the best implementation of memcpy?        -
- 1. implement your own slow/fast version of memcpy -
- 2. compare them with various size of data         -
- 3. conclude your experiment and submit report     -
-----------------------------------------------------
This time, just help me out with my experiment and get flag
No fancy hacking, I promise :D
specify the memcpy amount between 8 ~ 16 : specify the memcpy amount between 16 ~ 32 : specify the memcpy amount between 32 ~ 64 : specify the memcpy amount between 64 ~ 128 : specify the memcpy amount between 128 ~ 256 : specify the memcpy amount between 256 ~ 512 : specify the memcpy amount between 512 ~ 1024 : specify the memcpy amount between 1024 ~ 2048 : specify the memcpy amount between 2048 ~ 4096 : specify the memcpy amount between 4096 ~ 8192 : ok, lets run the experiment with your configuration

[...]

experiment 9 : memcpy with buffer size 2056
ellapsed CPU cycles for slow_memcpy : 14350
ellapsed CPU cycles for fast_memcpy : 672

experiment 10 : memcpy with buffer size 4102
ellapsed CPU cycles for slow_memcpy : 34910
ellapsed CPU cycles for fast_memcpy : 1350

thanks for helping my experiment!
flag : 1*********************************
```

