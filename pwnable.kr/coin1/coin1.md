We have no files to inspect this time! Great!

Let's try to do what the challenge says and do some netcat:

```bash
$ nc pwnable.kr 9007

	---------------------------------------------------
	-              Shall we play a game?              -
	---------------------------------------------------
	
	You have given some gold coins in your hand
	however, there is one counterfeit coin among them
	counterfeit coin looks exactly same as real coin
	however, its weight is different from real one
	real coin weighs 10, counterfeit coin weighes 9
	help me to find the counterfeit coin with a scale
	if you find 100 counterfeit coins, you will get reward :)
	FYI, you have 60 seconds.
	
	- How to play - 
	1. you get a number of coins (N) and number of chances (C)
	2. then you specify a set of index numbers of coins to be weighed
	3. you get the weight information
	4. 2~3 repeats C time, then you give the answer
	
	- Example -
	[Server] N=4 C=2 	# find counterfeit among 4 coins with 2 trial
	[Client] 0 1 		# weigh first and second coin
	[Server] 20			# scale result : 20
	[Client] 3			# weigh fourth coin
	[Server] 10			# scale result : 10
	[Client] 2 			# counterfeit coin is third!
	[Server] Correct!

	- Ready? starting in 3 sec... -
	
N=275 C=9
```
We are limited to only 60 seconds to do <i>A LOT</i> of guessing which sounds very unlikely for a human to do, so let's do it with our best friend, The Computer.

After running the netcat command multiple times i've noticed that 
2<sup>C</sup> <= N, This means that C <= log(N) meaning we can do binary search (or something similar) on `[0,...,N-1]`

But instead of searching for a specific number by comparing the values like in regular binary search, we can split `[0,...,N-1]` to half, check the sum of one of the halves, if it is divisble by 10, it means the counterfeit coin is in the other half, otherwise it is in the half we checked.

Then we can recursively do it until we reach a single element.

So let's build an algorithm for this problem:

We set some initial variables:
- low = 0
- high = N - 1
- mid = 0

Now we will run repeatedly until we reach the number of our tries or `low` becomes greater than `high`.

loop_start:
- mid = (high + low) / 2 (round it down)
- send the range [low, mid] to the program
- if the sum is divisble by 10:
    * low = mid + 1
- else
    * high = mid -1
- goto loop_start as long as low <= high and iteration_count <= C

After loop ends:
- if we found the number before our tries ended, just repeatedly send any other valid arbitrary number until we reach `C` tries.
- Send the answer


This should work, we also need to repeat this process 100 times
to get the flag, I wrote a python script to solve this challenge:
```python
import socket
import re

TRIAL_PATTERN = r"^N=(\d+) C=(\d+)$"
HOST = "0.0.0.0"
PORT = 9007


def play():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((HOST, PORT))
        print("{}".format(s.recv(2048).decode()))

        for it in range(100):
            regex_nc = re.findall(TRIAL_PATTERN, s.recv(1024).strip().decode())[0]
            N, C = int(regex_nc[0]), int(regex_nc[1])
            low, high = 0, N - 1
            iter_count = 0
            while low <= high and iter_count <= C:
                iter_count += 1
                mid = (high + low) // 2
                try_nums = [str(x) for x in range(low, mid + 1)]
                s.send((" ".join(try_nums) + "\n").encode())

                res_sum = int(s.recv(1024).strip().decode())

                if res_sum % 10 == 0:
                    low = mid + 1
                else:
                    high = mid - 1

            for _ in range(iter_count, C):
                s.send("0\n".encode())
                s.recv(1024)

            s.send("{}\n".format(low).encode())
            print("{}".format(s.recv(1024).decode()))
            if it == 99:
                res = s.recv(1024).strip().decode()  # Flag
                print("Flag: {}".format(res))


if __name__ == '__main__':
    play()
```

Also i realized my connection to the server wasn't really good so i had to log into a random machine and then run it locally on `0.0.0.0:9007`.
