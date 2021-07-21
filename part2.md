![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uqrjqcxdty6gpvqhrtrf.jpg)

In the [previous article](https://dev.to/nsadisha/build-your-own-operating-system-1setupbooting-171-temp-slug-9396464), we guided you to setup the booting part of our operating system. In this article, we are going to implement C language to our project instead of Assembly language. Assembly is a very good programming for interacting with CPU and other hardware resources. But, C is much more human-friendly language when compared with Assembly language. So, we decided to use C as much as possible to make the development process easier and assembly language will be used only where it make sense.

So, let’s get started!

### Setting up a stack

Since all non-trivial(not lightweight) C programs use a stack, and setting up a stack is not harder than to make the `esp` register point to the end of an area of free memory. So far, in this development process, the only things in memory are GRUB, BIOS, the OS kernel, and some memory mapped I/Os. This is not a good thing to do; because, we don’t know how much memory is available or if the `esp` pointed memory area is used by something else.

Reserving a piece of uninitialized memory in the `bss` section in the ELF file of the kernel will be a solution. And also, this will reduce the OS executable size.

```assembly
KERNEL_STACK_SIZE equ 4096 ; size of stack in bytes

    section .bss
    align 4 ; align at 4 bytes
    kernel_stack: ; label points to beginning of memory
        resb KERNEL_STACK_SIZE ; reserve stack for the kernel
```

Add this section to `loader.s` file.

And then, we need to setup the stack pointer by pointing `esp` to the end of the `kernel\_stack` memory. In order to do that, you need to add the following statement inside the `loader:` block you your `loader.s` file.

```assembly
mov esp, kernel_stack + KERNEL_STACK_SIZE ; point esp to the start of the
                                                ; stack (end of memory area)
```

After all, loader.s file will look like this:

```assembly
    global loader                   ; the entry symbol for ELF
    extern sum_of_three

    MAGIC_NUMBER equ 0x1BADB002     ; define the magic number constant
    FLAGS        equ 0x0            ; multiboot flags
    CHECKSUM     equ -MAGIC_NUMBER  ; calculate the checksum
                                    ; (magic number + checksum + flags should equal 0)

    section .text:                  ; start of the text (code) section
    align 4                         ; the code must be 4 byte aligned
        dd MAGIC_NUMBER             ; write the magic number to the machine code,
        dd FLAGS                    ; the flags,
        dd CHECKSUM                 ; and the checksum
        
    KERNEL_STACK_SIZE equ 4096                  ; size of stack in bytes

    section .bss
    align 4                                     ; align at 4 bytes
    kernel_stack:                               ; label points to beginning of memory
        resb KERNEL_STACK_SIZE                  ; reserve stack for the kernel

    loader:                         ; the loader label (defined as entry point in linker script)
        mov eax, 0xCAFEBABE         ; place the number 0xCAFEBABE in the register eax
        mov esp, kernel_stack + KERNEL_STACK_SIZE   ; point esp to the start of the
                                                ; stack (end of memory area)
    
    push dword 3
    push dword 2
    push dword 1
    call sum_of_three
    

    loader:                         ; the loader label (defined as entry point in linker script)
        mov eax, 0xCAFEBABE         ; place the number 0xCAFEBABE in the register eax
    .loop:
        jmp .loop                   ; loop forever
```

### Calling C code from the Assembly

Since we are using C language, we need to call the C code from the Assembly code. There are many different ways to do that. But in here, we will use [cdecl](https://en.wikipedia.org/wiki/X86_calling_conventions#cdecl) calling convention. According to this convention, the arguments of the function should be pushed on the stack in a right-to-left order, that is, you push the rightmost argument first. And, the return value of the function is placed in the `eax` register.

For example, to call the following function in Assembly:

```C
/* The C function */
    int sum_of_three(int arg1, int arg2, int arg3)
    {
        return arg1 + arg2 + arg3;
    }
```

You have to call it like this.

```assembly
; The assembly code
external sum_of_three ; the function sum_of_three is defined elsewhere

    push dword 3 ; arg3
    push dword 2 ; arg2
    push dword 1 ; arg1
    call sum_of_three ; call the function, result will be in eax
```

In your project directory, create an empty file called `kmain.c`. You can do it with `touch kmain.c` command. You can keep this file empty for now.

### Compiling the C code

The next step will be this. For normal compilations, we can use `gcc fileName.c -o objectName`. But, in this case, we are compiling them for an operating system. So, we have to use a lot of flags as below.

```
-m32 -nostdlib -nostdinc -fno-builtin -fno-stack-protector -nostartfiles -nodefaultlibs
```

And, we recommend you to turn on all warnings and treat warnings as errors by adding these flaags:

```
-Wall -Wextra -Werror
```

### Build tools

This is the last step for this article. In this step, we are going to build the OS. Previously, we used a lot of commands to compile each and every file separately, build the ISO image, and run the OS in `bochs` emulator. But, we can do it in an easier way. In order to do that, you have to create a separate file to execute those commands. Execute `touch Makefile` to create the file.

```make
    OBJECTS = loader.o kmain.o
    CC = gcc
    CFLAGS = -m32 -nostdlib -nostdinc -fno-builtin -fno-stack-protector \
             -nostartfiles -nodefaultlibs -Wall -Wextra -Werror -c
    LDFLAGS = -T link.ld -melf_i386
    AS = nasm
    ASFLAGS = -f elf

    all: kernel.elf

    kernel.elf: $(OBJECTS)
        ld $(LDFLAGS) $(OBJECTS) -o kernel.elf

    os.iso: kernel.elf
        cp kernel.elf iso/boot/kernel.elf
        genisoimage -R                              \
                    -b boot/grub/stage2_eltorito    \
                    -no-emul-boot                   \
                    -boot-load-size 4               \
                    -A os                           \
                    -input-charset utf8             \
                    -quiet                          \
                    -boot-info-table                \
                    -o os.iso                       \
                    iso

    run: os.iso
        bochs -f bochsrc.txt -q

    %.o: %.c
        $(CC) $(CFLAGS)  $< -o $@

    %.o: %.s
        $(AS) $(ASFLAGS) $< -o $@

    clean:
        rm -rf *.o kernel.elf os.iso
```

Save this file in `Makefile`. Note that you have to do all the indentations with tabs, not with spaces.

After all of these steps, your file structure should look like this.

```
    .
    |-- bochsrc.txt
    |-- iso
    | |-- boot
    | |-- grub
    | |-- menu.lst
    | |-- stage2_eltorito
    |-- kmain.c
    |-- loader.s
    |-- Makefile
```

Now, you should be able to run the OS in the `bochs` emulator by executing the simple command `make run`. This will compile the kernel and boot it up. Then check the `bochslog.txt` to find `RAX=00000000CAFEBABE` or `EAX=CAFEBABE` to make sure that your OS has successfully booted.

You can download a completed code that I have created for booting my OS from: [here](https://github.com/nsadisha/lemonOS/tree/implement_with_c)

Hope you have successfully implemented C to OS and hope to catch you in the next article.

Thank you!
