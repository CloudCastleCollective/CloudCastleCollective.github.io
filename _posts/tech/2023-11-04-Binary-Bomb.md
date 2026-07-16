---
title: "Binary Bomb"
excerpt_separator: "<!--more-->"
tags:
  - ctf
categories:
  - tech
---

![center-aligned-image](https://cdn.pixabay.com/photo/2017/06/14/15/22/bomb-2402460_1280.png){: .align-center}

Example BBs by **Luong Vo** @ [https://github.com/luong-komorebi/Binary-Bomb](https://github.com/luong-komorebi/Binary-Bomb)
{: .notice--info}

This binary bomb was an assignment I was given years ago during my computer architecture and system programming course at university. I remember at the time I had absolutely no idea how to go at it, I had no clue how to read assembly and I had never ever used gdb before. Needless to say, I was not able to solve it. Like, not even remotely. \
So while a friend of mine was following the same course and had access to the material, I asked him to forward me the challenge, as I wanted to try my hand at it again. First of all because I finally have more or less an understanding of how these things work, and secondly because I wanted to play around with IDA and gdb. \
I can't provide the exact files I worked on, but a lot of repositories offer similar challenges, for example the one linked above. 

<!--more-->

| Content: | 
|---------------------|----------------------|------------------------|
| [Install](#install) |	[Phase 1](#phase-01) | [Secret Phase](#secret-phase) |	
|---------------------|----------------------|------------------------|
|                     | [Phase 2](#phase-02) | 
|---------------------|----------------------|------------------------|
|                     |	[Phase 3](#phase-03) | 
|---------------------|----------------------|------------------------|
|                     | [Phase 4](#phase-04) | 
|---------------------|---------------------|-----------------------|
|                     | [Phase 5](#phase-05)
|---------------------|---------------------|-----------------------|
|                     | [Phase 6](#phase-06)
|---------------------|---------------------|-----------------------|

# Capture the Flag

### [Install]
The challenge itself is an executable C file, compiled for Linux x64. It will run just by calling it from the terminal, meaning ``./bomb` \
To analyze what is going on, we will use the free version of IDA: IDA will give us a nice overview of the code and especially of the code flow. The free version does not give the possibility of doing dynamic analysis, but in case you have the complete version feel free to use it to go through the entire project (udate: great news, IDA Free now can do debugging!!!). To load the project into IDA it is enough to drag and drop the executable over the IDA icon, and the software will start and do all the analysis automatically.

For the dynamic analysis, we will use GDB (should be preinstalled in Linux, otherwise get it with `sudo apt install gdb`) and we will expand it with the great GDB dashboard that you can download from [this repo by Andrea Cardaci](https://github.com/cyrus-and/gdb-dashboard). The dashboard will work just by putting the provided .gdbinit file in your home folder. To pipe GDB data into it you have to do the following:
1. Start GDB in one terminal
2. In another terminal run the tty command and copy the result (e.g. /dev/ttys001). Leave this terminal open, as it is where GDB will pipe the data
3. In the GDB terminal issue the command dashboard -output /dev/ttys001 (with your tty result)
4. Debug as usual, magic will happen in the second terminal

If you are as newbie in GDB as me, here are the commands that I found most useful:

| Command |
|--------------|-----------------|	
| gdb executable_name |	Launch the debugger with the given executable |
|--------------|-----------------|	
| gdb --args executable_name parameters |	Launch the debugger with the given executable and pass the given parameters to the executable |
|--------------|-----------------|	
| run |	Start or restart the execution of the program |
|--------------|-----------------|	
| c |	Continue execution until the next breakpoint |
|--------------|-----------------|	
| ni |	Go to the next assembly instruction (step over functions) |
|--------------|-----------------|	
| si |	Go to the next assembly instruction (step in functions) |
|--------------|-----------------|	
| fin |	Run unti return of current function |
|--------------|-----------------|	
| break [*0xYYYYYYYY \|\| func_name] |	Set a breakpoint to the given address or function |
|--------------|-----------------|	
| i b |	Infos on existing breakpoints |
|--------------|-----------------|	
| delete Y |	Delete breakpoint number Y |
|--------------|-----------------|	
| x/Yx [d \|\| w \|\| ...] [$reg_name \|\| 0xYYYYYYYY] 	| Show Y elements as double/word/... from the given register or address |
|--------------|-----------------|	

As a last detail, the version of the bomb that I used accepted as a parameter a file containing the already found keys, one per line. This makes it easier to reach the current point of the analysis without having to retype all the keys after every restart. 

### [Phase 01]
**String comparison.** \
This phase is really short and does not require the use of GDB as IDA does all the needed work for us.

![center-aligned-image](/images/binarybomb/phase1.png){: .align-center}

The only thing that happens in the phase is loading a register with the address of a string, and invoke a method called strings_not_equal. It is easy to guess what that method does, and we can see the we will reach the end of the phase without exploding if the return value of that method is 0. Therefore we can assume that the key we are looking for is the one that is passed as argument. In IDA, you can double click on it to reach its address, and copy the entire string from there. 

### [Phase 02]
**Looping.** \
This phase is pretty clear on what is the format of the expected output, as it reads it invoking a method called read_six_numbers. \
It gets slightly trickier when it comes to analyze how those six integers are analyzed, because it happens in a loop where each element depends from the previous one.

![center-aligned-image](/images/binarybomb/phase2.png){: .align-center}

The loop iterates over $ebx, which is initialized to 1 before the loop and is incremented one by one until it reaches the value 6, at that point the loop is over. This hints on the fact that the loop will check every element of the input. \
What decides if the bomb explodes or not is the following line: `cmp [rbp+4], eax`. In assembly, cmp subtracts the second operand from the first and just sets the flags without storing the result of the operation. It means that if the two operands are the same, the Zero Flag will be set, and this is what the following instruction is doing: go on with the next iteration of the loop only if $eax and [rbp+4] contain the same value. \
It helps going through the loop a couple of times using gdb and looking what is stored in $eax and in $rbp: to do that you can reach the compare instruction and then look at the registers with `x/x $rbp`, `x/x $rbp+4` and `x/x $eax`. You will see that $rbp contains the current input element, and $rbp+4 the following one, while $eax contains another value to which our input should correspond to defuse the bomb. \
Let's analyze the loop: in each iteration $eax is initialized to the current iteration value, then it has added the value of the current input, and that should correspond to the value of the next input. If the comparison is passed, our counter is incremented and so is the pointer to the current input. The corresponding pseudocode is the following one:

```python
eax = 1 + rbp[0]
for i = 1..5:
    if eax != rbp[i]
        explode
    eax = eax + i + 1
```

To get the key, start with a random value as rbp[0] and work through the loop to get the following 5 elements needed to pass this phase. An example of a working sequence is 61 ; 62 ; 64 ; 67 ; 71 ; 76. 

### [Phase 03]
**Switch Statement** \
Here is when IDA starts getting really useful, thanks to all the data that it extracts during its initial analysis and inserts exactly where we need it. An example is for the input of this function: the bomb uses a library function (sscanf), that we can look up on msdn to discover that it takes a pointer to a buffer and a pointer to a format as parameters. IDA provides us with the format, that is stored as a string: %d %c %d. Looking up the string formatting in C, we see that this corresponds to an integer, a char and an integer again.

![center-aligned-image](/images/binarybomb/phase3_1.png){: .align-center}

If the format of the input is respected, the next instruction checks the first input. We know it's the input because that's the pointer to the buffer that we passed to the sscanf method, and we know it's the first one because we can look into it with GDB using `x/4x $rsp+0x28-0x18`. This value should be <= 7, to cover the switch cases. If it is larger, it will fall directly to the default case, making the bomb explode.

![center-aligned-image](/images/binarybomb/phase3_2.png){: .align-center}

Thanks to IDA we know that we are going through a switch clause (jumping to different branches with the instruction `jmp ds:off_402620[rax*8]`), and we can look into each branch. The branches all have the same structure: moving a value into $eax, comparing a value to a memory location and if the result sets the zero flag (meaning the values are the same) jumping to another comparision of the lowest bits of $eax. The memory location that we compare is also a buffer that we passed to sscanf, and it is where our second integer is stored (quickly confirmed with GDB `x/4x $rsp+0x28-0x14`). The value loaded into $eax is an ASCII value, and that should correspond to our char input. \
To pass this phase, it is enough to go through the 8 branches of the switch clause (well, picking one is enough), and using as input the number of that branch and the char and number defined in that same branch. In case of laziness, here is the complete table:

```bash
branch    hex   char    hex     int
  0       0x71   	q    	0x2B1 	689
  1      	0x67   	g    	0x1BF 	447
  2      	0x61   	a    	0x15B 	347
  3      	0x66   	f    	0x2EF 	751
  4      	0x71   	q    	0x2D4 	724
  5      	0x7A   	z    	0x367 	871
  6      	0x69   	i    	0x1C9 	457
  7      	0x62   	b    	0x213 	531
```

### [Phase 04]
**Recursion** \
The phase requires an input made by two numbers, as indicated by the "%d %d" in the parameter passed to sscanf (this can be seen with IDA). The second number passed is the first one checked (look at $eax with GDB), and it can only take the value 2, 3 or 4.

![center-aligned-image](/images/binarybomb/phase4.png){: .align-center}

If this check is passed, the executable goes on to run func4, which is a recursive method. I honestly have no clue what the method does , but after running some tests with GDB, it is clear that it always returns the same values depending on the input. This means that with input 2 the output is 40, with input 3 it is 60 and with input 4 it is 80. This output is the first value that you want to pass to defuse the bomb. 

### [Phase 05]
**Array** \
This phase requires a string of length 6, judging by the method that is invoked to check the input. Then there is a loop, and we can assume we are looping over each element of the input. Somehow, these elements are used to build a value in $ecx that has to correspond to 0x32 (== 50) after the loop ends in order to defuse the bomb.

![center-aligned-image](/images/binarybomb/phase5_1.png){: .align-center}

The value in $ecx is built by repeatedly adding to itself a value of an array stored in memory. Which value is picked depends on current char of our input.

![center-aligned-image](/images/binarybomb/phase5_2.png){: .align-center}

The values of the array are the ones reported in the figure. To defuse the bomb, we have to pick 6 values that summed give 50. The trick is that these values are picked as their index in the array, and the index is given by the lowest byte of the ASCII representation of the current char. This index is obtained by moving the input to $edx (which is the lowest part of $rdx, used later to access the array) and masking out everything apart from the lowest byte, ensuring that the value will be between 0x0 and 0xF. A way to get the correct sum is 10 + 10 + 10 + 10 + 1 + 9, which corresponds to the string 'aaaacf'. 

### [Phase 06]
**Linked List** \
Input of this phase is 6 numbers, assuming that we trust the name of the method that reads in data. The input is then run through a tricky loop: the first part just checks that the given numbers are between 1 and 6, exploding the bomb for any other value. The second part is making sure that the numbers do not repeat themselves in the input, and I realized this just by running through the loop multiple times with test values and seeing what was the behavior. \
When all the checks are passed successfully, we move on to actually using the input. IDA makes it clear that we are accessing a structure in memory at a fix offset, and we can look at it with GDB:

![center-aligned-image](/images/binarybomb/phase6_1.png){: .align-center}

The structure is a linked list that contains a value, an index and the address of the next element. The problem is now to understand how this object is used. Let's start by saying that the current input is stored in $ecx. We iterate over the input until we find the position of value 1 in the input (in this case, it will be in fifth position to indicate the fifth node). There is a lot of stuff going on with the node (again, if you know what it is please let me know), but the next important instructions for us are `cmp [rbx], eax; jle loc_401248`

![center-aligned-image](/images/binarybomb/phase6_2.png){: .align-center}

Among another bunch of stuff, what is happening here is that we are comparing the value of the node we gave as input with another value of those nodes, and we will do this with all nodes. It takes some iterations with GDB to see that the nodes we are comparing with the ones given by our input are in increasing order. This means that we have to give as input the order of nodes from the smallest to the highest. The sequence is the following: 5 ; 3 ; 6 ; 1 ; 2 ; 4 

### [Secret Phase]
**Double Linked List + Recursion** \
By exporting the strings of the bomb (for example with the command `strings bomb`) or by looking at the functions recognized by IDA, we can see that there is a secret phase method. By doing a cross reference with IDA we identify the caller of the `secret_phase` function, which is the `phase_defused` method called after each phase of the bomb. However, to invoke this method we first need to pass the correct input to reach the calling instruction.

![center-aligned-image](/images/binarybomb/secretphase_1.png){: .align-center}

To access the phase we see that the program re-reads the current input expecting two numbers and a string (%d %d %s). The only phase that takes a similar input is the fourth, that accepts %d %d: it means that by adding a string to this phase we will be able to access the secret phase. What we have to add is the string "DrEvil", as showed by IDA in the image above.

Finally in the secret phase itself, we have to pass an input. The input string will be casted to long with strtol, and the result will have to be smaller or equal to 0x3E8 to avoid an explosion.

![center-aligned-image](/images/binarybomb/secretphase_2.png){: .align-center}

After some tests and the help of GDB to check what is returned by strtol in $eax, it becomes clear that the accepted values are between 1 and 1001. This value is then used by the method fun7, which also takes a pointer to a data structure as an argument. Also we know that the value returned by the function has to be equal to 7 to successfully defuse the bomb.

![center-aligned-image](/images/binarybomb/secretphase_3.png){: .align-center}

This data structure is a list where each element contains a value, and two pointers to other nodes. The value is at node_address, the first pointer at node_address+8, the second pointer at node_address+16. Here is the content:

```bash
node  value   1. linked node  2. linked node
n1 	  0x24 	        n21 	          n22
n21 	0x8 	        n31 	          n32
n22 	0x32 	        n33 	          n34
n32 	0x16 	        n43 	          n44
n33 	0x2D 	        n45 	          n46
n31 	0x6 	        n41 	          n42
n34 	0x6B 	        n47 	          n48
n45 	0x28 	        - 	            -
n41 	0x1 	        - 	            -
n47 	0x63 	        - 	            -
n44 	0x23 	        - 	            -
n42 	0x7 	        - 	            -
n43 	0x14 	        - 	            -
n46 	0x2F 	        - 	            -
n48 	0x3E9 	      - 	            -
```
We enter fun7 knowing that we have to keep an eye on $eax, that will contain the return value, and on the data structure, which is pointed at by $rdi (easily checked with GDB). The function is recursive, and we can see that we have to reach the value of 7 by either using `add eax, eax` or `lea eax, [rax+rax+1]`. The second case corresponds to `$eax = 2*$eax + 1`.\
What we have to do is to jump along the nodes of the data structure the correct amount of times to have $eax set to the current value. Starting from n1, the input value is compared to the value of the current node and the following node is determined depending on whether the zero flag is set or not. We want to always fall on the case where our input is bigger than the node value, so that we can get to 7 by doing `2 * ( 2 * ( 2 * eax + 1 ) + 1 ) + 1`. \
Also we have to pay attention that our input corresponds to the value of the last node we want to visit, otherwise $eax will be screwed up. It seems that it is enough to visit three nodes to achieve the result, but running it with GDB shows that we miss one iteration. Plus, to exit the recursion our last value has to be equal to what is stored in $esi, which corresponds to the value of the last node. So with 4 jumps we have to reach n48, and leaves us with only a path: n1, n22, n34, n48. This also means that our input has to be larger than 0x24, than 0x32 and 0x6B, and also be equal to 0x3E9, which clearly means that it has to be 1001 (= 0x3E9). 
