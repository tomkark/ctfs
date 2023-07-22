Some env playground....

```c
#include <stdio.h>
#include <string.h>

int filter(char* cmd){
	int r=0;
	r += strstr(cmd, "flag")!=0;
	r += strstr(cmd, "sh")!=0;
	r += strstr(cmd, "tmp")!=0;
	return r;
}
int main(int argc, char* argv[], char** envp){
	putenv("PATH=/thankyouverymuch");
	if(filter(argv[1])) return 0;
	system( argv[1] );
	return 0;
}
Some env playground....

```

Well, this C program will set the PATH env variable to `/thankyouverymuch` and then check if the output contains any of the following:
- "flag": to prevent us from trying to access the flag file
- "sh": to prevent us getting a root shell using the suid of the program
- "tmp": to prevent us from executing code from tmp


But actually we can read the flag by setting some sorts of an environment variable to `/bin/cat flag` (it's important to call `/bin/cat` and not `cat` since the program overwrites PATH) and then sending our input to be the evaluation of that env variable, the `filter` function cannot read it, and it will only evaluate on the `system` call.

So the solution would be this:
`LOL="/bin/cat flag" ./cmd1 \$LOL`

It's also important to escape the `$` character so the evaluation will happen at `system` call and not on our initial call to `./cmd1`.
