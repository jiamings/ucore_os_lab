# UCore OS Lab 1
## Exercise 1
### Prerequisites
This Makefile uses variables defined by `tools/function.mk`, which are called through the [`call`](https://www.gnu.org/software/make/manual/html_node/Call-Function.html#Call-Function) function. 
These function also use [`foreach`](https://www.gnu.org/software/make/manual/html_node/Foreach-Function.html#Foreach-Function) and [`eval`](https://www.gnu.org/software/make/manual/html_node/Foreach-Function.html#Foreach-Function) function.
Here are some of the commands:

 - `read_packet`: add prefix `__objs_` to list of arguments.
 - `totarget`: add prefix `$(BINDIR)` to argument. (In this case `$(BINDIR)` is `bin/`)
 - `create_target`: add packets and objects to target argument.

### How to generate `ucore.img`?
First, we observe that `ucore.img` is created in the following commands:
```[bash]
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)
```

The target `$(UCOREIMG)` depends on two targets `$(kernel)` and `$(bootblock)`.

We can then recursively analyze how the targets `$(kernel)` and `$(bootblock)` are created.

#### Target `$(kernel)` (alias for `bin/kernel`)
To build `$(kernel)` the following variables are provided:

 - `KINCLUDE`: include directories for target.
 - `KSRCDIR`: source directories for target.
 - `KCFLAGS`: define the flags for `gcc` to compile. In this case `-I` is used.
 - `KOBJ`: the packets from `kernel` and `libs` target. 

`$(kernel)` has the following dependencies:

 - `kernel.ld` (exists)
 - `obj/kern/*/*.o` where * denotes the file from folders `kern` and `lib`.

To build `*.o` the following command is issued:

```[bash]
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
```
where `add_files_cc` depends on `add_files`, which depends on `do_add_files_to_packet`.
`listf_cc` provides a list of `.c` and `.s` files within `$(KSRCDIR)`, so only files in `kern/*` are compiled.

Now that we have all the dependencies, we can use the following command to build `$(kernel)`:

```[bash]
$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
```

The function of the commands are as follows:

 - `@echo + ld $@` does nothing but display `ld bin/kernel`.
 - `$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)` builds target `$(kernel)`.
 - `$(OBJDUMP)` does nothing but display `cc kern/*/*.c` etc.

#### Target `$(bootblock)` (alias for `bin/bootblock`)
`$(bootblock)` has three dependencies:
 - `bootasm.o` (toobj)
 - `bootmain.o` (toooj)
 - `bin/sign` (totarget)

Then the following commands are invoked:
```[bash]
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
```

The flag `-Ttext` assigns the start position of the code; `-N` sets code and data with read and write access.

`$(OBJDUMP)` command still writes to the terminal (`cc boot/*`), whereas
`$(OBJCOPY)` command copies `bootblock.o` to `bootblock.out`, with `-S` (strip-all) and `-O [bfdname]` (output-target).

Finally `@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)` calls `bin/sign obj/bootblock.out bin/bootblock`.

#### Merging `$(kernel)` and `$(bootblock)` to `$(UCOREIMG)`
By calling `man dd` we see that `dd` copy and converts a file. The following commands call `dd`:
```[bash]
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
```

The arguments used for `dd` are as follows:
 - `if = FILE` (read from FILE instead of stdin)
 - `of = FILE` (write to FILE instaed of stdout)
 - `count = N` (copy only N input blocks)
 - `seek = N` (skip N obs(512)-sized blocks at start of output)
 - `conv = CONVS` (convert the file as per the comma separated symbol list)

So basically these arguments does the following:
 - `dd if=/dev/zero of=bin/ucore.img count=10000` (fill 10000 blocks of 0, where each block has size 512)
 - `dd if=bin/bootblock of=bin/ucore.img conv=notrunc` (write bootblock to the first block)
 - `dd if=bin/kernel of=bin/ucore.img conv=notrunc seek=1` (skip 1 block (bootblock), and write kernel from the second block).

And this gives us the `ucore.img` file.

### What are the requirements for a valid sector?
From lines 31-32 of `tools/sign.c`, we find that a sector is valid when it contains 512 bytes, and the last 2 bytes of the sector is `0x55AA`.

## Exercise 2
### Use GDB to debug QEMU
#### Tracking BIOS from the first command and setting pointer at `0x7c00`
Using the hint, we add this target to our Makefile:
```[bash]
lab1-mon: $(UCOREIMG)
	$(V)$(TERMINAL) -e "$(QEMU) -S -s -d in_asm -D $(BINDIR)/q.log -monitor stdio -hda $< -serial null"
	$(V) sleep 2
	$(V)$(TERMINAL) -e "gdb -q -x tools/lab1init"
```

The first command activates `$(QEMU)`, where as the third command activates gdb and uses commands in `tools/lab1init` to initialize.
In order to break at address 0x7c00, we need to set `b *0x7c00` in the `tools/lab1init` file. Then we use 
```
define hook-stop
x/i $pc
end
```
to disassemble the code of PC register when we break. And then we can use `ni` or `nexti` to go to the exact next step in assembley.

#### Compare the execution code with `bootasm.S`
We use step-by-step running in gdb for the first steps and get the following:
```
0x00007c00 cli
0x00007c01 cld
0x00007c02 xor %ax, %ax
0x00007c04 mov %ax, %ds
0x00007c06 mov %ax, %es
0x00007c08 mov %ax, %ss
```
which is exactly the same as the lines 16-23 of `bootasm.S`. Continuing we can find that the commands executed are the same as `bootasm.S` up until `call bootmain`. 

#### Comparing `bin/q.log` with `bootasm.S`
Adding `-d in_asm -D $(BINDIR)/q.log` would generate log file indicating the executed commands, `q.log` contains a lot of other commands before `0x7c00` is executed. In fact the executed commands of `bootasm.S` are at the end of the log file.

There is an error with the answer: `x/10i $pc` does not put commands into bin/q.log file. The commands can be logged only when they are executed.

## Exercise 3
### How bootloader enters protected mode
#### Switching to A20
A20 is needed for the program to switch to real mode to protected mode. In real mode, the program treats physical memory as two parts - code and data.   The operating system is no different from the user program, so if a program tries to access data in other programs or the OS, it would be a disaster. So protected mode is neeeded, and that is where A20 comes to play.
Only in A20 mode, the 32-bit address is utilized.

How to start A20? From the code and comments in `bootasm.S` we can figure out the following:

 - Setup the important data segment registers (DS, ES, SS)
 - 0xd1 -> port 0x64 (seta20.1)
 - 0xdf -> port 0x60 (seta20.2)

[This site](http://docs.huihoo.com/gnu_linux/own_os/booting-a20_4.htm) provides more details on why we need A20.

#### Load GDT table
This code does the job of loading a bootstrap GDT:

```
lgdt gdtdesc
```

#### Enter protected mode
First, set the PE bit of `cr0` to 1:

```
movl %cr0, %eax
orl $CR0_PE_ON, %eax
movl %eax, %cr0
```

Then make a long jump to change the `pc` register, which changes 16-bit mode to 32-bit mode.

```
ljmp $PROT_MODE_CSEG, $protcseg
.code32
protcseg:
```

Use the adder register `AX` to setup the pointers to stacks, including data segment (`DS`), extra segment (`ES`), stack segments (`SS`),  more segments (`FS` and `GS`), set base pointer to zero, and set stack pointer to $start. 
```
        movw $PROT_MODE_DSEG, %ax
        movw %ax, %ds
        movw %ax, %es
        movw %ax, %fs
        movw %ax, %gs
        movw %ax, %ss
        movl $0x0, %ebp
        movl $start, %esp
```

Finally, call the bootmain program.

```
call bootmain
```

## Exercise 4
### How does bootloader read sectors?
According to the function `readsect()`, reading from a sector requires the following steps:

- Wait for the disk to be ready (see `waitdisk()` function)
- Send the command for reading a sector
 + `outb(0x1F2, 1)` claim that 1 sector is visited.
 + `outb(0x1F3, secno & 0xFF)` `outb(0x1F4, (secno >> 8) & 0xFF)` `outb(0x1F5, (secno >> 16) & 0xFF)` denotes the 0-7, 8-15, 16-23 bit of LBA parameter.
 + `outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0)` is the 24-27 bit
 + `outb(0x1F7, 0x20)` gives the 0x20 command.
- Wait for the disk to be ready
- `insl(0x1F0, dst, SECTSIZE / 4)` reads the sector data from port 0x1F0. 

### How does bootloader load ELF format OS?
According to the function `bootmain()`, loading a ELF format OS requires the following steps:

 - Read the first page of disk, and check whether it is a valid ELF (according to ELF magic)
 - Read each program segment into the designated virtual address `(ph->p_va)`, which has size `(ph->p_memsz)` and offset `(ph->p_offset)`.
 - Call the entry point from the ELF entry header from address (`ELFHDR->e_entry`)

Since the function does not return, so the mark `bad` cannot be reached unless ELF is invalid.

## Exercise 5
### Implement `kern/debug/kdebug.c::print_stackframe()`
The implementation is rather simple, just follow the comments. One thing to note is when using a pointer reference, you should force it to `(uint32_t *)`, and let arguments, ebp, eip all of type `uint32_t`.

The last line is where the `bootmain` function is called. Both `ebp` and `eip` are 0x00007bf8, since the call stack starts from 0x00007c00.

## Exercise 6
### About IDT
Each Interrupt Descriptor Table (IDT) element contains 8 bytes. The 2-3 bytes are SELECTOR, the 0-1 bytes and 6-7 bytes are OFFSET. (According to the figure in 2.3.3.2).

### Code for `idt_init`
Here we need to understand `SETGATE` macro from `kern/mm/mmu.h`. There are some requirements for using `SETGATE`:

 - The gate is an interrupt gate.
 - The DPL should be kernel DPL except for one. Otherwise user can invoke interrupts so long as they wish.
 - The `T_SWITCH_TOK` indicates the descriptor from user state to kernel state, only this is set as user DPL.

### Code for `trap`
It is indeed too simple.
