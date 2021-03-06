###break at 0x7c00
bios passes its control to bootloader at 0x7c00. This is a fixed agreement for PC.  

```  
(gdb) b *0x7c00  
Breakpoint 1 at 0x7c00  
(gdb) c  
Continuing.  
The target architecture is assumed to be i8086  
[   0:7c00] => 0x7c00:	cli  


Breakpoint 1, 0x00007c00 in ?? ()  
(gdb)  
```  
we are now in bootloader. In the previous section, we understand that BIOS once enabled A20 and switched to protected mode, but later in code, it disabled them again. So we start with 16-bit real mode with A20 disabled.  


###sanitizing
```  
0x7c00:  cli  
```  
stop interrupt  


```  
0x7c01:	cld  
```  
clear data flag  

```  
0x7c02:	xor    %ax,%ax  
0x7c04:	mov    %ax,%ds  
0x7c06:	mov    %ax,%es  
0x7c08:	mov    %ax,%ss  
```  
clear segment registers  


###enable A20
start of function _seta20.1_  
```  
0x7c0a:	in     $0x64,%al  
0x7c0c:	test   $0x2,%al  
0x7c0e:	jne    0x7c0a  
```  
This is the "classical" way of enabling A20 (compared with fast A20, explained in [1_ex2.md](https://github.com/Philip-Li/6.828-MIT/blob/master/hw1/1_ex2.md))  

It pulls from port 0x64 (keyboard controller) until keyboard answers (because cpu and keyboard are two devices running in parallel). Keyboard saves its status in a status controller which can be read from port 0x64, where bit 0 is output buffer status (0: empty buffer, don't read; 1: readable) and bit 1 input buffer (0: empty, writable; 1: full, don't write)  
So _0x2(10b)_ means not writable and not readable aka. keyboard is busy  


For more information on keyboard controller, refer to [this](https://www.win.tue.nl/~aeb/linux/kbd/scancodes-11.html) and [this](https://www.win.tue.nl/~aeb/linux/kbd/A20.html)


As for assembly commands, _test_ substracts two operands and sets _ZF_ to 0 if the difference is 0. _jne_ or _jnz_ (same thing, notice that source code uses jnz but compiler translates to jne) reads from _ZF_ to make a conditional jump.


For explaination of _jne_ and _jnz_, see [here](http://stackoverflow.com/questions/14267081/difference-between-je-jne-and-jz-jnz)


```  
0x7c10:	mov    $0xd1,%al  
0x7c12:	out    %al,$0x64
```
once the keyboard is not busy. Tell the keyboard it's ready to write. Explained line by line.  


```  
0x7c10:	mov    $0xd1,%al  
```  
0xd1 is write output port command. Detailed commands see [here](https://www.win.tue.nl/~aeb/linux/kbd/scancodes-11.html#kccd1)


```  
0x7c12:	out    %al,$0x64  
```  
write to output port 0x64 (status port).


End of function _seta20.1_

Start of function _seta20.2_  
```  
0x7c14:	in     $0x64,%al  
0x7c16:	test   $0x2,%al  
0x7c18:	jne    0x7c14  
```  
This is exactly like first half of _seta20.1_, keeping pulling status from keyboard.  


```  
0x7c1a:	mov    $0xdf,%al  
0x7c1c:	out    %al,$0x60  
```  
Now, enable A20. _0xdf_ is to enable A20. port _0x60_ is data buffer port.  


###enable protected mode
```  
0x7c1e:	lgdtw  0x7c64  
```  
loads global descriptor table. 


```  
0x7c23:	mov    %cr0,%eax  
0x7c26:	or     $0x1,%eax  
0x7c2a:	mov    %eax,%cr0  
```  
This is the same as bios, described in [ex2](https://github.com/Philip-Li/6.828-MIT/blob/master/hw1/1_ex2.md). Basically, enabling protected mode by turning protected mode bit on in _cr0_.


```  
0x7c2d:	ljmp   $0x8,$0x7c32  
```  
long jump to refresh _cs_, so that _cs_ points to the correct selector. see [ex2](https://github.com/Philip-Li/6.828-MIT/blob/master/hw1/1_ex2.md) for detail.  


Now we are in 32-bit mode.


###setting up data segment
```  
0x7c32:	mov    $0x10,%ax  
0x7c36:	mov    %eax,%ds  
0x7c38:	mov    %eax,%es  
0x7c3a:	mov    %eax,%fs  
0x7c3c:	mov    %eax,%gs  
0x7c3e:	mov    %eax,%ss  
```  
Sets up data segments. 


###on to C code
```  
0x7c40:	mov    $0x7c00,%esp  
0x7c45:	call   0x7d0b  
```  
Moves magical address _0x7c00_ to _sp_. Now cpu is using segmentation to address, so _ss_:_sp_ is used to address stack. Then it calls _bootmain_ in C, which is the function in kernel.  


Assembly code should end here.


###failure spin
```  
spin:  
  jmp spin  
```  
If boot failed, cpu will spin forever in an empty loop.  


###GDT setup in assembly
***GDT description structure***  
GDT is loaded by _lgdt_ with a GDT description structure containing two parts:  
![GDT description structure](http://wiki.osdev.org/images/7/77/Gdtr.png)

```  
gdtdesc:  
  .word   0x17                            # sizeof(gdt) - 1  
  .long   gdt                             # address gdt  
```  

The first part is a 2-byte size, which is sizeof(gdt) - 1. This is because _size_ can only be 65535 bytes and GDT can be up to 65536. So substract 1 to prevent overflow.


The second part is the 4-byte address of the GDT.  

***GDT***
At least three items in GDT:    
1. null descriptor ([source](http://wiki.osdev.org/GDT_Tutorial) says its used to store pointer to GDT itself. Not sure why.)  
2. code segment descriptor  
3. data segment descriptor  

This corresponds to code  
```  
gdt:  
  SEG_NULL  
  SEG(STA_X|STA_R, 0x0, 0xffffffff)  
  SEG(STA_W, 0x0, 0xffffffff)  
```  
which are null descriptor, code seg descriptor, and data seg descriptor.
code segment is executable and readable. Data segment is writable. Both of them has 4GB size.  


_'mmu.h'_ defines _SEG_ and _SEG\_NULL_ as  

```  
#define SEG_NULL						\  
	.word 0, 0;						\  
	.byte 0, 0, 0, 0  
#define SEG(type,base,lim)  					\  
	.word (((lim) >> 12) & 0xffff), ((base) & 0xffff);	\  
	.byte (((base) >> 16) & 0xff), (0x90 | (type)),		\  
		(0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)  
```  

_.word_ is 4 bytes, so the descriptor is 8-byte or 64-bit wide.  
This expains two magic number _0x08_ and _0x10_. code segment descriptor starts at _0x08_ because each descriptor is 8-byte and starts from 0x0 (the null segment descriptor). and data segment descriptor starts at 0x10 (16 in decimal).  


_SEG\_NULL_ is simply filled with zeros.  


The function passes 0xffffffff (4GB) as argument, but this is only for readability. This value is shifted by 12 to become 0xfffff paired with 4K page size to make address space 4GB.

![Global Descriptor Structure](http://wiki.osdev.org/images/f/f3/GDT_Entry.png)

***limit***

```  
(lim >> 12) & 0xffff
```  
shifted by 12 to have correct limit value.
masked with 0xffff because first half of _limit_ takes up 16 bit (0 to 15).  


```  
(lim >> 28) & 0xf
```  
shifted by 28, that is 12 + 16, the initial 12 bits and 16 bits for the first half.
masked with 0xf because second half of _limit_ takes up 4 bit(48-51).  


***base***
```  
base & 0xffff  
```  
masked with 0xffff because first half of _base_ takes up 16 bit (16 to 31).  


```  
(base >> 16) & 0xff  
```  
shifted by 16, because first half takes up 16 bits.  
masked with 0xff for the second part takes up 8 bits (32 to 39).  


```  
(base >> 24) & 0xff  
```   
shifed by 24 (16 for first half + 8 for second half).  
masked with 0xff for the third part takes up 8 bits (56 to 63).


***access byte***  
![access bits and flags](http://wiki.osdev.org/images/1/1b/Gdt_bits.png)


```  
0x90 | type  
```  
0x90 or 1001 0000b turns on present bit (8th bit) and 1 for reserved bit (5th bit). Present bit must be turned on if the segment is valid.  


_STA\_X_ is 0x8 or 1000b, turning on execution bit (4th bit)
_STA\_R_ and _STA\_W_ is 0x2 or 10b, turning on RW bit. for code segment, this means readable (never writable). for data segment, this means writable (read is always allowed). Whether a segment is a data or code segment depends on execution bit. 


***flags***  
```  
0xC0 | ((lim >> 28) & 0xf)
```  
_flags_ takes up the upper four bits of a bite, which third part of _limit_ takes up the lower half.  

0xC0 is 1100 0000b, which turns on _granularity_ bit (4K instead of 1B) and _size_ bit for 32 bit protected mode (off for 16 bit protected mode)  


The detailed description of descriptor bits can be found [here](http://wiki.osdev.org/GDT)


***.word & .byte***  
here, one word is 2 byte so its  
(_low to high_)  
.word 8-bits, 8-bits  
.byte 8-bits, 8-bits, 8-bits, 8-bits  


or  
.word (first part of _limit_), (first part of _base_)  
.byte (second part of _base_), (type), (flags, third part of _limit_), (third part of _base_)  


***alignment***  
```  
.p2align 2  
```  
p2align means alignment to the power of 2 bytes, so _palign 2_ is 4 bytes alignment.  
I'm not sure why it forces 4-byte alignment. Mostly, alignment is used as optimization, but it can be required here. Not sure.