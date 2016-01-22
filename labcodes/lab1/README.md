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
