We have the given source code `mistake.c`:
```c
#include <stdio.h>
#include <fcntl.h>

#define PW_LEN 10
#define XORKEY 1

void xor(char* s, int len){
	int i;
	for(i=0; i<len; i++){
		s[i] ^= XORKEY;
	}
}

int main(int argc, char* argv[]){
	
	int fd;
	if(fd=open("/home/mistake/password",O_RDONLY,0400) < 0){
		printf("can't open password %d\n", fd);
		return 0;
	}

	printf("do not bruteforce...\n");
	sleep(time(0)%20);

	char pw_buf[PW_LEN+1];
	int len;
	if(!(len=read(fd,pw_buf,PW_LEN) > 0)){
		printf("read error\n");
		close(fd);
		return 0;		
	}

	char pw_buf2[PW_LEN+1];
	printf("input password : ");
	scanf("%10s", pw_buf2);

	// xor your input
	xor(pw_buf2, 10);

	if(!strncmp(pw_buf, pw_buf2, PW_LEN)){
		printf("Password OK\n");
		system("/bin/cat flag\n");
	}
	else{
		printf("Wrong Password\n");
	}

	close(fd);
	return 0;
}
```
From a first look, this seems like the flow of the code:
- try to open `password`
- sleep
- try to read `password`'s contents
- ask the user for input (only first 10 chars for strings)
- xor the input
- compare it to `password`'s contents

First thought is probably to debug the program, but we cannot, as `password` needs higher permissions for opening, but running `mistake` through gdb will keep the permissions
as the `mistake` user since gdb can't run a setuid/setgid programs without root permissions.

I tried to run  the executable multiple times and noticed it waited for some kind of an input before actually printing out `"input password: "`, so I went ahead and tried to inspect the code. 

After inspecting the code for some time i noticed a weird thing, if we look at the first `if` logically, it should open a file, store the return value in fd and check if it's less than 0,
which should look like this: `if((fd=open(...)) < 0)`, but the actual code looks like this: `if(fd=open(...) < 0)` which is semantically wrong. 

As a result of the nature of operator priority in C, it will store the comparison result in `fd`, meaning either `0` or `1`.

This explains why it waited for an input before the `printf` call, because when the open succeeds, the comparison returns 0, meaning `fd` has the value 0, And we already know that fd 0 represents stdin.

So now we know we control both `pw_buf` and `pw_buf2`.

- The character '1' in ascii has the decimal value of 49.
- 49 ^ 1 = 48
- 48 is the decimal number of '0'
- We can enter a string of length 10 that only has ones for the first input, and then a string that only has 0's for the second one.

This solves the challenge and gives us the flag.
