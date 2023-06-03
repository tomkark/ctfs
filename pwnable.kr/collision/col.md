# <u> Solution for <b>collision</b> ctf </u>

Again, let's look curiously in the source provided:
```c
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
        int* ip = (int*)p;
        int i;
        int res=0;
        for(i=0; i<5; i++){
                res += ip[i];
        }
        return res;
}

int main(int argc, char* argv[]){
        if(argc<2){
                printf("usage : %s [passcode]\n", argv[0]);
                return 0;
        }
        if(strlen(argv[1]) != 20){
                printf("passcode length should be 20 bytes\n");
                return 0;
        }

        if(hashcode == check_password( argv[1] )){
                system("/bin/bash echo -c 'hooray'");
                return 0;
        }
        else
                printf("wrong passcode.\n");
        return 0;
}

```

We immediately notice we have some sort of a verification function, that is called from 
main, does some stuff and then returns some value to compare to the given hex (`hashcode`)

We see that in `check_password` we cast the given char array pointed to by `p` to an integer 
array, this could be confusing for some, but the main idea you should get is that if 
`'1111'` is stored in a memory block for example, it is stored in binary, looking like this:

<b> `00110001 00110001 00110001 00110001` </b>

P.S:  
<i><u>'1' is 49 in ASCII  
A char occupies a single byte</i></u>

Since int takes up 4 bytes in memory, the 4 bytes will represent a single integer, this is why 20 bytes in char representation, are actually 5 integers.

So now we know we should input 20 bytes that should represent 5 integers that sum up to 
`0x21DD09EC`.

Let's do some calculations to help ourselves:

```python
$ i
Python 3.10.6 (main, Mar 10 2023, 10:55:28) [GCC 11.3.0]
Type 'copyright', 'credits' or 'license' for more information
IPython 8.14.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]: hashcode = '0x21DD09EC'

In [2]: int(hashcode, 16)
Out[2]: 568134124
```

Let's divide this number by 5, if the number isn't divisble by 5, we can just divide by 4, round down, and find the remainder.
```python
In [3]: int(hashcode, 16) / 5
Out[3]: 113626824.8

In [4]: int(hashcode, 16) // 5
Out[4]: 113626824

In [5]: # Remainder

In [6]: int(hashcode, 16) - 4 * (int(hashcode, 16) // 5)
Out[6]: 113626828
```

Let's convert them back to hex and we should get that `4 * 0x6c5cec8 + 0x6c5cecc` sums up to our hashcode!

Now last step left to do, feed the `col` executable the specific combination of bytes.
Since data is stored in little-endian (at least on modern systems), we need to reverse byte order and input it to col.

```
./col `echo -ne` '\xc8\xce\xc5\06\xc8\xce\xc5\06\xc8\xce\xc5\06\xc8\xce\xc5\06\xcc\xce\xc5\x06'`
```

And that's it! We got the flag.
