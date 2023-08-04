```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <fcntl.h>
char flag[100];
char password[100];
char* key = "3\rG[S/%\x1c\x1d#0?\rIS\x0f\x1c\x1d\x18;,4\x1b\x00\x1bp;5\x0b\x1b\x08\x45+";
void calc_flag(char* s){
	int i;
	for(i=0; i<strlen(s); i++){
		flag[i] = s[i] ^ key[i];
	}
	printf("%s\n", flag);
}
int main(){
	FILE* fp = fopen("/home/blukat/password", "r");
	fgets(password, 100, fp);
	char buf[100];
	printf("guess the password!\n");
	fgets(buf, 128, stdin);
	if(!strcmp(password, buf)){
		printf("congrats! here is your flag: ");
		calc_flag(password);
	}
	else{
		printf("wrong guess!\n");
		exit(0);
	}
	return 0;
}
```

I wasted so much time trying to exploit this code, a buffer overflow looked pretty trivial here so i was trying to see if i can abuse it, but there's no buffer overflow here, The '128' and '100' in the code is there just to confuse us.

If we were to go to the VM and try to 'cat password' we'd get:
```bash
blukat@pwnable:~$ cat password 
cat: password: Permission denied
```

Trying to view the permission for it:
```bash
blukat@pwnable:~$ ls -l password 
-rw-r----- 1 root blukat_pwn 33 Jan  6  2017 password
```

And the trick for this challenge was to notice the group of the current user:
```bash
blukat@pwnable:~$ id
uid=1104(blukat) gid=1104(blukat) groups=1104(blukat),1105(blukat_pwn)
```

Welp, this is just infuriating, we were in `blukat_pwn` all this time.... The contents of password were literally "cat: password: Permission denied", we can even see it using 
`less` or `cat password 2> /dev/null` to make sure our stderr is not displayed.
