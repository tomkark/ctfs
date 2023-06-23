Let's view the given c code:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
	printf("Welcome to pwnable.kr\n");
	printf("Let's see if you know how to give input to program\n");
	printf("Just give me correct inputs then you will get the flag :)\n");

	// argv
	if(argc != 100) return 0;
	if(strcmp(argv['A'],"\x00")) return 0;
	if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
	printf("Stage 1 clear!\n");	

	// stdio
	char buf[4];
	read(0, buf, 4);
	if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
	read(2, buf, 4);
        if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
	printf("Stage 2 clear!\n");

	// env
	if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
	printf("Stage 3 clear!\n");

	// file
	FILE* fp = fopen("\x0a", "r");
	if(!fp) return 0;
	if( fread(buf, 4, 1, fp)!=1 ) return 0;
	if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
	fclose(fp);
	printf("Stage 4 clear!\n");	

	// network
	int sd, cd;
	struct sockaddr_in saddr, caddr;
	sd = socket(AF_INET, SOCK_STREAM, 0);
	if(sd == -1){
		printf("socket error, tell admin\n");
		return 0;
	}
	saddr.sin_family = AF_INET;
	saddr.sin_addr.s_addr = INADDR_ANY;
	saddr.sin_port = htons( atoi(argv['C']) );
	if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
		printf("bind error, use another port\n");
    		return 1;
	}
	listen(sd, 1);
	int c = sizeof(struct sockaddr_in);
	cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
	if(cd < 0){
		printf("accept error, tell admin\n");
		return 0;
	}
	if( recv(cd, buf, 4, 0) != 4 ) return 0;
	if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
	printf("Stage 5 clear!\n");

	// here's your flag
	system("/bin/cat flag");	
	return 0;
}
```
Looks like we have multiple stages to pass in order, so let's split the steps:

### Stage 1
```c

	if(argc != 100) return 0;
	if(strcmp(argv['A'],"\x00")) return 0;
	if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
	printf("Stage 1 clear!\n");	
```

- We want to supply 99 arguments to the program (not 100, since argv[0] is the command that invoked this process)
- `argv['A']` should be set to `"\x00"`
- `argv['B']` should be set to `"\x20\x0a\x0d"`

So to solve the first stage i wrote this code snippet in C:
```c
#include <stdio.h>
#include <unistd.h>

#define RUN_PATH "./input"

int main() {
    char *argv[101] = {[0] = RUN_PATH, [100] = NULL, [1 ... 99] = "1", ['A']="\x00", ['B']="\x20\x0a\x0d"};
    if (execve(RUN_PATH, argv, NULL) < 0){
      perror("Some exec error");
    }
}
```
Runnig it resulted in success:
```bash
$ gcc sol.c -o sol && ./sol
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!

```

### Stage 2
```c
char buf[4];
read(0, buf, 4);
if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
read(2, buf, 4);
if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
printf("Stage 2 clear!\n");
```

This stage tries to do 2 things:
- Read 4 bytes from stdin and compare them to `"\x00\x0a\x00\xff"`
- Read 4 bytes from stderr and compare them to `"\x00\x0a\x02\xff"`

We can try to pass this step using pipes, since we can then write to a file descriptor and then use `dup2` to duplicate the fd we wrote to to a any given fd we want.
- Create a child process, write our bytes to the pipes, then the parent reads from the other end of the pipe and calls dup2 to our wanted fd's.

```c
#include <stdio.h>
#include <unistd.h>

#define RUN_PATH "./input"

int main() {
    char *argv[101] = {[0] = RUN_PATH, [100] = NULL, [1 ... 99] = "1", ['A']="\x00", ['B']="\x20\x0a\x0d"};
    int in[2], err[2];
    pipe(in); pipe(err); // We should check for errors but mehhhhhh.
    pid_t cpid = fork();
    if (cpid == -1){
      perror("Fork failed"); return -1;
    }
    if (!cpid){
      close(in[0]); close(err[0]);
      write(in[1], "\x00\x0a\x00\xff", 4);
      write(err[1], "\x00\x0a\x02\xff", 4);
    }
    else{
      close(in[1]); close(err[1]);
      dup2(in[0], 0); dup2(err[0], 2);
      if (execve(RUN_PATH, argv, NULL) < 0){
        perror("Some exec error");
      }
    }
}
```
Run it...
```bash
$ gcc sol.c -o sol && ./sol
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
Stage 2 clear!
Segmentation fault (core dumped)
```

### Stage 3
```c
// env
if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
printf("Stage 3 clear!\n");
```

Well it's pretty straight forward, just add these env variables to our `execve` call:

```c
#include <stdio.h>
#include <unistd.h>

#define RUN_PATH "./input"

int main() {
    char *argv[101] = {[0] = RUN_PATH, [100] = NULL, [1 ... 99] = "1", ['A']="\x00", ['B']="\x20\x0a\x0d"};
    int in[2], err[2];
    pipe(in); pipe(err); // We should check for errors but mehhhhhh.
    pid_t cpid = fork();
    if (cpid == -1){
      perror("Fork failed"); return -1;
    }
    if (!cpid){
      close(in[0]); close(err[0]);
      write(in[1], "\x00\x0a\x00\xff", 4);
      write(err[1], "\x00\x0a\x02\xff", 4);
    }
    else{
      close(in[1]); close(err[1]);
      dup2(in[0], 0); dup2(err[0], 2);

      char* envp[2] = {"\xde\xad\xbe\xef=\xca\xfe\xba\xbe", NULL};

      if (execve(RUN_PATH, argv, envp) < 0){
        perror("Some exec error");
      }
    }
}
```

And...

```bash
$ gcc sol.c -o sol && ./sol
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
Stage 2 clear!
Stage 3 clear!
```

### Stage 4

This is also pretty obvious, as we just need to write to `\x0a` 4 times `\x00`:
```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

#define RUN_PATH "./input"

int main() {
    char *argv[101] = {[0] = RUN_PATH, [100] = NULL, [1 ... 99] = "1", ['A']="\x00", ['B']="\x20\x0a\x0d"};
    int in[2], err[2];
    pipe(in); pipe(err); // We should check for errors but mehhhhhh.
    pid_t cpid = fork();
    if (cpid == -1){
      perror("Fork failed"); return -1;
    }
    if (!cpid){
      close(in[0]); close(err[0]);
      write(in[1], "\x00\x0a\x00\xff", 4);
      write(err[1], "\x00\x0a\x02\xff", 4);

      char buf[4] = "\x00\x00\x00\x00";
      FILE* fp = fopen("\x0a", "w");
      if(!fp) return -1;
      fwrite(buf, 4, 1, fp); // Please whoever reads this, do check errors
      fclose(fp);
    }
    else{
      close(in[1]); close(err[1]);
      dup2(in[0], 0); dup2(err[0], 2);

      char* envp[2] = {"\xde\xad\xbe\xef=\xca\xfe\xba\xbe", NULL};

      if (execve(RUN_PATH, argv, envp) < 0){
        perror("Some exec error");
      }
    }
}
```
```bash
$ gcc sol.c -o sol && ./sol
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
Stage 2 clear!
Stage 3 clear!
Stage 4 clear!
bind error, use another port
```

### Stage 5

This is the last stage, we need to create a socket and send data over it to the `input` process. So let's just do some socket programming and connect to the socket created in `input`
in the child scope, but sleep 2 seconds before actually connecting to make sure everything in `input` works well and socket creation succeeds.

Notice we also need to set argv['C'] to some port number, so let's set it to '11111':
```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#define RUN_PATH "./input"

int main() {
    char *argv[101] = {[0] = RUN_PATH, [100] = NULL, [1 ... 99] = "1", ['A']="\x00", ['B']="\x20\x0a\x0d", ['C']="11111"};
    int in[2], err[2];
    pipe(in); pipe(err); // We should check for errors but mehhhhhh.
    pid_t cpid = fork();
    if (cpid == -1){
      perror("Fork failed"); return -1;
    }
    if (!cpid){
      close(in[0]); close(err[0]);
      write(in[1], "\x00\x0a\x00\xff", 4);
      write(err[1], "\x00\x0a\x02\xff", 4);

      char buf[4] = "\x00\x00\x00\x00";
      FILE* fp = fopen("\x0a", "w");
      if(!fp) return -1;
      fwrite(buf, 4, 1, fp); // Please whoever reads this, do check errors
      fclose(fp);

      sleep(2);
      struct sockaddr_in addr;
      int client_fd;
      char msg[4] = "\xde\xad\xbe\xef";
      if ((client_fd = socket(AF_INET, SOCK_STREAM, 0)) < 0){
          return -1;
      }
      addr.sin_family = AF_INET;
      addr.sin_port = htons(atoi(argv['C']));
      addr.sin_addr.s_addr = inet_addr("127.0.0.1");

      if (connect(client_fd, (struct sockaddr*)&addr, sizeof(addr)) < 0){
          return -1;
      }
      send(client_fd, msg, 4, 0);
      close(client_fd);
    }
    else{
      close(in[1]); close(err[1]);
      dup2(in[0], 0); dup2(err[0], 2);

      char* envp[2] = {"\xde\xad\xbe\xef=\xca\xfe\xba\xbe", NULL};

      if (execve(RUN_PATH, argv, envp) < 0){
        perror("Some exec error");
      }
    }
}
```

And if we run it locally we get:
```bash
$ gcc sol.c -o sol && ./sol
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
Stage 2 clear!
Stage 3 clear!
Stage 4 clear!
Stage 5 clear!
```
Now we need to remember that we don't have write permission in the home directory of the remote machine, so let's try to do that in /tmp but changing `RUN_PATH` to the actual input exe path.

```bash
input2@pwnable:~$ mkdir /tmp/sol
input2@pwnable:~$ cd /tmp/sol
input2@pwnable:/tmp/sol$ cat - > sol.c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#define RUN_PATH "./input"

int main() {
    char *argv[101] = {[0] = RUN_PATH, [100] = NULL, [1 ... 99] = "1", ['A']="\x00", ['B']="\x20\x0a\x0d", ['C']="11111"};
    int in[2], err[2];
    pipe(in); pipe(err); // We should check for errors but mehhhhhh.
    pid_t cpid = fork();
    if (cpid == -1){
      perror("Fork failed"); return -1;
    }
    if (!cpid){
      close(in[0]); close(err[0]);
      write(in[1], "\x00\x0a\x00\xff", 4);
      write(err[1], "\x00\x0a\x02\xff", 4);

      char buf[4] = "\x00\x00\x00\x00";
      FILE* fp = fopen("\x0a", "w");
      if(!fp) return -1;
      fwrite(buf, 4, 1, fp); // Please whoever reads this, do check errors
      fclose(fp);

      sleep(2);
      struct sockaddr_in addr;
      int client_fd;
      char msg[4] = "\xde\xad\xbe\xef";
      if ((client_fd = socket(AF_INET, SOCK_STREAM, 0)) < 0){
          return -1;
      }
      addr.sin_family = AF_INET;
      addr.sin_port = htons(atoi(argv['C']));
      addr.sin_addr.s_addr = inet_addr("127.0.0.1");

      if (connect(client_fd, (struct sockaddr*)&addr, sizeof(addr)) < 0){
          return -1;
      }
      send(client_fd, msg, 4, 0);
      close(client_fd);
    }
    else{
      close(in[1]); close(err[1]);
      dup2(in[0], 0); dup2(err[0], 2);

      char* envp[2] = {"\xde\xad\xbe\xef=\xca\xfe\xba\xbe", NULL};

      if (execve(RUN_PATH, argv, envp) < 0){
        perror("Some exec error");
      }
    }
}            
^C
input2@pwnable:/tmp/sol$ vi sol.c # Editing RUN_PATH to be /home/input2/input
input2@pwnable:/tmp/sol$ gcc sol.c -o sol
input2@pwnable:/tmp/sol$ ./sol
Welcome to pwnable.kr
Let's see if you know how to give input to program
Just give me correct inputs then you will get the flag :)
Stage 1 clear!
Stage 2 clear!
Stage 3 clear!
Stage 4 clear!
Stage 5 clear!
```

Welp, after a bit of looking on why the flag wasn't printed out, I found out that it tried to `/bin/cat flag` but in the current working directory which is invalid since the actual flag is in `~/flag`, so let's just create a symlink from `~/flag` to `/tmp/sol/flag`
`ln -s ~/flag /tmp/sol/flag` and now we should have the flag!

