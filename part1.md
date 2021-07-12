# Build your own Operating System #1_setup_booting

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/y0xuuz6m6mg86ixb81ae.jpg)

This is the first article of “_Build your own OS_” article series. In this article series, we hope to guide you how you can build your own operating system with _linux kernel_. Developing a custom operating system is not an easy task. But, you will be able to develop your own simple operating system at the end of this article series.

I will be using Ubuntu as the operating system for the development. Since we have to directly access the memory while developing, its better to try all these things in a virtual box environment for safety purposes. I will be using C and assembly languages in this article series.

So, let’s get started!

### Setting up development environment

Once you have done installing Ubuntu in a virtual box, the next step is installing some necessary packages ( **build-essential** , **nasm** , **genisoimage** , **bochs** and **bochs-sdl** ). You can simply install them by executing this command in your terminal.

```bash
sudo apt-get install build-essential nasm genisoimage bochs bochs-sdl
```

#### build-essential

This package contains GNU debugger, g++/GNU compiler collection, and some tools and libraries which are required to compile a C program.

#### nasm

Since we are using assembly, we have to use nasm(Netwide Assembler) to compile assembly programs.

#### genisoimage

We need this package to generate an ISO image file for the file system.

#### bochs

This will be the emulator that we will be using to debug our operating system.

### Booting

Booting is the process of starting a computer. After you press the power button, computer will run several programs before handover the control of the computer to the operating system.

![Alt Text](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ar3gp8v4toddg0107ymr.png)

#### BIOS

BIOS program(stands for **B**asic **I**nput **O**utput **S**ystem) is usually stored on a read only memory chip on the motherboard of the PC. The original role of the BIOS program was to export some library functions for printing to the screen, reading keyboard input etc. After all, it will transfer the control of the computer to the bootloader.

#### Bootloader

This program’s task is to handover the control of the computer to the operating system. But, due to some restrictions, the bootloader is often split into two parts. The first part of the bootloader will transfer control to the second part, which finally transfers the control to the operating system.

We will be using an existing bootloader called “_GNU GRand Unified Bootloader (GRUB)_”. Because we don’t need to write a lot of low-level codes. GRUB will load our operating system to the correct memory location.

Download GRUB: [https://github.com/littleosbook/littleosbook/raw/master/files/stage2\_eltorito](https://github.com/littleosbook/littleosbook/raw/master/files/stage2_eltorito)

### Compiling the operating system

In here we use little bit of assembly code. This program will write a very specific number `0xCAFEBABE` to the `eax` register. Save the following code in a file called `loader.s`

```assembly
    global loader                   ; the entry symbol for ELF

    MAGIC_NUMBER equ 0x1BADB002     ; define the magic number constant
    FLAGS        equ 0x0            ; multiboot flags
    CHECKSUM     equ -MAGIC_NUMBER  ; calculate the checksum
                                    ; (magic number + checksum + flags should equal 0)

    section .text:                  ; start of the text (code) section
    align 4                         ; the code must be 4 byte aligned
        dd MAGIC_NUMBER             ; write the magic number to the machine code,
        dd FLAGS                    ; the flags,
        dd CHECKSUM                 ; and the checksum

    loader:                         ; the loader label (defined as entry point in linker script)
        mov eax, 0xCAFEBABE         ; place the number 0xCAFEBABE in the register eax
    .loop:
        jmp .loop                   ; loop forever
```

The file `loader.s` can be compiled into a 32 bits ELF [18] object file with the following command:

```bash
nasm -f elf32 loader.s
```

### Linking the Kernel

Next, GRUB needs to load the Kernel into the memory. It should be loaded into a memory address larger than or equal to `0x00100000` (1 megabyte (MB)), because addresses lower than 1 MB are used by GRUB itself, BIOS and memory-mapped I/O. In order to do that, we will be using the following code.

```
ENTRY(loader)                /* the name of the entry label */

SECTIONS {
    . = 0x00100000;          /* the code should be loaded at 1 MB */

    .text ALIGN (0x1000) :   /* align at 4 KB */
    {
        *(.text)             /* all text sections from all files */
    }

    .rodata ALIGN (0x1000) : /* align at 4 KB */
    {
        *(.rodata*)          /* all read-only data sections from all files */
    }

    .data ALIGN (0x1000) :   /* align at 4 KB */
    {
        *(.data)             /* all data sections from all files */
    }

    .bss ALIGN (0x1000) :    /* align at 4 KB */
    {
        *(COMMON)            /* all COMMON sections from all files */
        *(.bss)              /* all bss sections from all files */
    }
}
```

Save the linker script into a file called `link.ld`. And then, the executable can now be linked with the following command:

```bash
ld -T link.ld -melf_i386 loader.o -o kernel.elf
```

The final executable file will be generated, called `kernel.elf`.

Then, copy the downloaded file (stage2\_eltorito) in to the same project location as `loader.s` and `link.ld`.

### Building an ISO image file

ISO image is the type that can be loaded by a virtual or physical machine. So, we will create the kernel ISO image with the `genisoimage` package. Before generating an ISO, we need to create a specific folder structure and copy the bootloader and the kernel into different locations. Which can be done by executing following commands.

```
mkdir -p iso/boot/grub                 # create the folder structure
cp stage2_eltorito iso/boot/grub/      # copy the bootloader
cp kernel.elf iso/boot/                # copy the kernel
```

Finally, a configuration file `menu.lst` for GRUB must be created. This file tells GRUB where the kernel is located and configures some options:

```
    default=0
    timeout=0

    title os
    kernel /boot/kernel.elf
```

Place the file `menu.lst` in the folder `iso/boot/grub/`. Then, the contents of the `iso` folder should now look like the following figure:

```
iso
 |-- boot
       |-- grub
       |     |-- menu.lst
       |     |-- stage2_eltorito
       |-- kernel.elf
```

If your folder structure is correct, you can generate your ISO image with the following command.

```
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
```

Now you have successfully created the `os.iso` file.

### Running your OS in Bochs

Now we can run the OS in the Bochs emulator using the `os.iso` ISO image. One last thing before that, Bochs needs a configuration file to start and an example of a simple configuration file is given below:

```
    megs:            32
    display_library: sdl
    romimage:        file=/usr/share/bochs/BIOS-bochs-latest
    vgaromimage:     file=/usr/share/bochs/VGABIOS-lgpl-latest
    ata0-master:     type=cdrom, path=os.iso, status=inserted
    boot:            cdrom
    log:             bochslog.txt
    clock:           sync=realtime, time0=local
    cpu:             count=1, ips=1000000
```

Save this file as `bochsrc.txt` in the root directory of your project(same location as `loader.s`). And run the following command to boot your os in Bochs emulator.

```
bochs -f bochsrc.txt -q
```

Extra: If you see any error in your terminal try changing `display\_library: sdl` to `display\_library: sdl2` . And try!

If you see some text on the bochs emulator, containing “Booting os”, quit the emulator. If emulator has no text on the screen, type `continue` in the terminal and hit enter. It will boot the OS.

After quitting the Bochs, you can check the log generated by the Bochs. You can open it in a text editor or else execute the following command:

```
cat bochslog.txt
```

You should now see the contents of the registers of the CPU simulated by Bochs somewhere in the output. If you find `RAX=00000000CAFEBABE` or `EAX=CAFEBABE` (depending on if you are running Bochs with or without 64 bit support) in the output;

Congratulations!!! Your OS has successfully booted!

You can download a completed code that I have created for booting my OS from: [https://github.com/nsadisha/lemonOS/tree/setup\_booting\_os](https://github.com/nsadisha/lemonOS/tree/setup_booting_os)

Hope you have successfully booted your OS and hope to catch you in the next article.

Thank you!
