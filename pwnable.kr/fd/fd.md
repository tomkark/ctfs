# <u> Solution for <b>fd</b> ctf </u>

When starting the challenge we see the source code of the `fd.c` file as follows: 
```c

char buf[32];
int main(int argc, char* argv[], char* envp[]){
	if(argc<2){
		printf("pass argv[1] a number\n");
		return 0;
	}
	int fd = atoi( argv[1] ) - 0x1234;
	int len = 0;
	len = read(fd, buf, 32);
	if(!strcmp("LETMEWIN\n", buf)){
		printf("good job :)\n");
		system("/bin/cat flag");
		exit(0);
	}
	printf("learn about Linux file IO\n");
	return 0;

}

```

First thing we should notice, is the subtraction:
`int fd = atoi( argv[1] ) - 0x1234;`

Later on wee see that we read 32 bytes from the buffer and then compare the contents of it to "LETMEWIN".

So in order to manipulate the `read` to read from some sort of an input we will give, we can make sure it is set to 0 (meaning it will read from `stdin`)
and then we can input "LETMEWIN" to get the flag.

So our input should just be the decimal representation of `0x1234` since the substraction will zero out the fd variable, afterwards just input "LETMEWIN" and you get the flag.
