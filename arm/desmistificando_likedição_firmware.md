# Do nada ao main(): Desmistificando a linkedição do firmware
-------------------------------------------------------------

#### Original de *François Baldassari*

[Da última vez](/blog/zero-to-main-1), falamos sobre inicializar um ambiente C em um MCU antes de invocar nossa função `main`. Uma coisa que damos como certo foi o fato de que funções e dados acabam no lugar certo em nosso binário. Hoje, vamos nos aprofundar em como isso acontece aprendendo sobre regiões de memória e scripts de linker.

Você pode se lembrar das seguintes coisas acontecendo de forma automática:

1. Usamos variáveis como `&_ebss`, `&_sdata`, …etc. saber onde cada uma de nossas seções foi colocada em flash e definir onde algumas precisavam ir na RAM.
2. Um ponteiro para nosso `ResetHandler` foi encontrado no endereço 0x00000004 para o MCU encontrar.

Embora essas coisas sejam verdadeiras para muitos projetos, elas são mantidas juntas, na melhor das hipóteses, por convenção, na pior, por gerações de engenharia de copiar/colar. Você descobrirá que alguns MCUs têm mapas de memória diferentes, alguns scripts de inicialização nomeiam essas variáveis de maneira diferente e alguns programas têm mais ou menos segmentos.

Como não são padronizados, essas coisas precisam ser especificadas em algum lugar do nosso projeto. No caso de projetos vinculados a uma ferramenta do tipo Unix-`ld`\-like, esse em algum lugar é o script do vinculador.

Mais uma vez, usaremos nosso programa “minimal” simples, disponível [no Github](https://github.com/memfault/zero-to-main/blob/master/minimal).

Breve cartilha sobre Linkedição
-------------------------------

A vinculação é o último estágio na compilação de um programa. Ele pega vários arquivos de objetos compilados e os mescla em um único programa, preenchendo endereços para que tudo esteja no lugar certo.

Antes de vincular, o compilador terá seus arquivos de origem um por um e os compilado em código de máquina. No processo, ele deixa espaços reservados para endereços, pois (1) não sabe onde o código terminará dentro da estrutura mais ampla do programa e (2) não sabe nada sobre símbolos fora do arquivo ou unidade de compilação atual.

O vinculador pega todos esses arquivos de objeto e os mescla junto com dependências externas, como a C Standard Library, em seu programa. Para descobrir quais bits vão para onde, o vinculador depende de um script de vinculador - um modelo para seu programa. Por fim, todos os espaços reservados são substituídos por endereços.

Podemos ver isso em jogo em nosso programa mínimo. Vamos seguir o que acontece com nossa função `main` em `minimal.c` por exemplo. O compilador o compila em um arquivo de objeto com:

```bash
usuario@maquina$:\arm-none-eabi-gcc -c -o build/objs/a/b/c/minimal.o minimal.c <CFLAGS>
```

Podemos despejar símbolos em `minimal.o` para ver `main` dentro dele:

```bash
usuario@maquina$:\arm-none-eabi-nm build/objs/a/b/c/minimal.o
...
00000000 T main
...
```

Como esperado, ainda não tem endereços. Em seguida, vinculamos tudo a:

```bash
usuario@maquina$:\arm-none-eabi-gcc <LDFLAGS> build/objs/a/b/c/minimal.o <other object files> -o build/minimal.elf
```

And dump the symbols in the resulting `elf` file:
```bash
usuario@maquina$:\arm-none-eabi-nm build/minimal.elf
...
00000294 T main
...
```
O linker fez seu trabalho, e nosso símbolo `main` recebeu um endereço.

O vinculador geralmente faz um pouco mais do que isso. Por exemplo, ele pode gerar informações de depuração, coletar lixo de seções de código não utilizadas ou executar a otimização de todo o programa (também conhecida como Link-Time Optimization ou LTO). Por causa desta conversa, não abordaremos esses tópicos.

Para obter mais informações sobre o vinculador, há um ótimo tópico em [Stack Overflow](https://stackoverflow.com/questions/3322911/what-do-linkers-do).

Anatomia de um **Script de Ligação**
-------------------------------------------------- -------

Um **script de ligação** contém quatro coisas:

* Layout de memória: qual memória está disponível e onde
* Definições de seção: que parte de um programa deve ir para onde
* Opções: comandos para especificar arquitetura, ponto de entrada, etc. se necessário
* Símbolos: variáveis para injetar no programa na hora do link

## Layout de memória
-------------------------------

In order to allocate program space, the linker needs to know how much memory is available, and at what addresses that memory exists. This is what the `MEMORY` definition in the **script de ligação** is for.

The syntax for MEMORY is defined in the [binutils docs](https://sourceware.org/binutils/docs/ld/MEMORY.html#MEMORY) and is as follow:

    MEMORY
      {
        name [(attr)] : ORIGIN = origin, LENGTH = len
        …
      }
    

Where

*   `name` is a name you want to use for this region. Names do not carry meaning, so you’re free to use anything you want. You’ll often find “flash”, and “ram” as region names.
*   `(attr)` are optional attributes for the region, like whether it’s writable (`w`), readable (`r`), or executable (`x`). Flash memory is usually `(rx)`, while ram is `rwx`. Marking a region as non-writable does not magically make it write protected: these attributes are meant to describe the properties of the memory, not set it.
*   `origin` is the start address of the memory region.
*   `len` is the size of the memory region, in bytes.

The memory map for the SAMD21G18 chip we’ve got on our board can be found in its [datasheet](http://ww1.microchip.com/downloads/en/DeviceDoc/SAMD21-Family-DataSheet-DS40001882D.pdf) in table 10-1, reproduced below.

SAMD21G18 Memory Map

Memory

Start Address

Size

Internal Flash

0x00000000

256 Kbytes

Internal SRAM

0x20000000

32 Kbytes

  

Transcribed into a `MEMORY` definition, this gives us:

    MEMORY
    {
      rom      (rx)  : ORIGIN = 0x00000000, LENGTH = 0x00040000
      ram      (rwx) : ORIGIN = 0x20000000, LENGTH = 0x00008000
    }
    

## Section Definitions
-------------------------------------------

Code and data are bucketed into sections, which are contiguous areas of memory. There are no hard rules about how many sections you should have, or what they should be, but you typically want to put symbols in the same section if:

1.  They should be in the same region of memory, or
2.  They need to be initialized together.

In our previous post, we learned about two types of symbols that are initialized in bulk:

1.  Initialized static variables which must be copied from flash
2.  Uninitialized static variables which must be zeroed.

Our **script de ligação** concerns itself with two more things:

1.  Code and constant data, which can live in read-only memory (e.g. flash)
2.  Reserved sections of RAM, like a stack or a heap

By convention, we name those sections as follow:

1.  `.text` for code & constants
2.  `.bss` for uninitialized data
3.  `.stack` for our stack
4.  `.data` for initialized data

The [elf spec](http://refspecs.linuxbase.org/elf/elf.pdf) holds a full list. Your firmware will work just fine if you call them anything else, but your colleagues may be confused and some tools may fail in odd ways. The only constraint is that you may not call your section `/DISCARD/`, which is a reserved keyword.

First, let’s look at what happens to our symbols if we do not define any of those sections in the **script de ligação**.

    MEMORY
    {
      rom      (rx)  : ORIGIN = 0x00000000, LENGTH = 0x00040000
      ram      (rwx) : ORIGIN = 0x20000000, LENGTH = 0x00008000
    }
    
    SECTIONS
    {
        /* empty! */
    }
    

The linker is perfectly happy to link our program with this. Probing the resulting elf file with objdump, we see the following:

    $ arm-none-eabi-objdump -h build/minimal.elf
    build/minimal.elf:     file format elf32-littlearm
    
    SYMBOL TABLE:
    no symbols
    

No symbols! While the linker is able to make assumptions that will allow it to link in symbols with little information, but it at least needs to know either what the entry point should be, or what symbols to put in the text section.

### `.text` Section[](#text-section)

Let’s start by adding our `.text` section. We want that section in ROM. The syntax is simple:

    SECTIONS
    {
        .text :
        {
    
        } > rom
    }
    

This defines a section named `.text`, and adds it to the ROM. We now need to tell the linker what to put in that section. This is accomplished by listing all of the sections from our input object files we want in `.text`.

To find out what sections are in our object file, we can once again use `objdump`:

    $ arm-none-eabi-objdump -h
    build/objs/a/b/c/minimal.o:     file format elf32-littlearm
    
    Sections:
    Idx Name          Size      VMA       LMA       File off  Algn
      0 .text         00000000  00000000  00000000  00000034  2**1
                      CONTENTS, ALLOC, LOAD, READONLY, CODE
      1 .data         00000000  00000000  00000000  00000034  2**0
                      CONTENTS, ALLOC, LOAD, DATA
      2 .bss          00000000  00000000  00000000  00000034  2**0
                      ALLOC
      3 .bss.cpu_irq_critical_section_counter 00000004  00000000  00000000  00000034
    2**2
                      ALLOC
      4 .bss.cpu_irq_prev_interrupt_state 00000001  00000000  00000000  00000034
    2**0
                      ALLOC
      5 .text.system_pinmux_get_group_from_gpio_pin 0000005c  00000000  00000000
    00000034  2**2
                      CONTENTS, ALLOC, LOAD, READONLY, CODE
      6 .text.port_get_group_from_gpio_pin 00000020  00000000  00000000  00000090
    2**1
                      CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
      7 .text.port_get_config_defaults 00000022  00000000  00000000  000000b0  2**1
                      CONTENTS, ALLOC, LOAD, READONLY, CODE
      8 .text.port_pin_set_output_level 0000004e  00000000  00000000  000000d2  2**1
                      CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
      9 .text.port_pin_toggle_output_level 00000038  00000000  00000000  00000120
    2**1
                      CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
     10 .text.set_output 00000040  00000000  00000000  00000158  2**1
                      CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
     11 .text.main    0000002c  00000000  00000000  00000198  2**2
                      CONTENTS, ALLOC, LOAD, RELOC, READONLY, CODE
    

We see that each of our symbol has a section. This is due to the fact that we compiled our firmware with the `-ffunction-sections` and `-fdata-sections` flags. Had we not included them, the compiler would have been free to merge several functions into a single `text.<some identifier>` section.

To put all of our functions in the `.text` section in our **script de ligação**, we use the following syntax: `<filename>(<section>)`, where `filename` is the name of the input files whose symbols we want to include, and `section` is the name of the input sections. Since we want all `.text...` sections in all files, we use the wildcard `*`:

    .text :
    {
        KEEP(*(.vector*))
        *(.text*)
    } > rom
    

Note the `.vector` input section, which contains functions we want to keep at the very start of our `.text` section. This is so the `Reset_Handler` is where the MCU expects it to be. We’ll talk more about the vector table in a future post.

Dumping our elf file, we now see all of our functions (but no data)!

    $ arm-none-eabi-objdump -t build/minimal.elf
    
    build/minimal.elf:     file format elf32-littlearm
    
    SYMBOL TABLE:
    00000000 l    d  .text  00000000 .text
    ...
    00000000 l    df *ABS*  00000000 minimal.c
    00000000 l     F .text  0000005c system_pinmux_get_group_from_gpio_pin
    0000005c l     F .text  00000020 port_get_group_from_gpio_pin
    0000007c l     F .text  00000022 port_get_config_defaults
    0000009e l     F .text  0000004e port_pin_set_output_level
    000000ec l     F .text  00000038 port_pin_toggle_output_level
    00000124 l     F .text  00000040 set_output
    00000000 l    df *ABS*  00000000 port.c
    00000190 l     F .text  00000028 system_pinmux_get_config_defaults
    00000000 l    df *ABS*  00000000 pinmux.c
    00000208 l     F .text  0000005c system_pinmux_get_group_from_gpio_pin
    00000264 l     F .text  00000110 _system_pinmux_config
    00000164 g     F .text  0000002c main
    000001b8 g     F .text  0000004e port_pin_set_config
    00000374 g     F .text  00000040 system_pinmux_pin_set_config
    ...
    

### `.bss` Section[](#bss-section)

Now, let’s take care of our `.bss`. Remember, this is the section we put uninitialized static memory in. `.bss` must be reserved in the memory map, but there is nothing to load, as all variables are initialized to zero. As such, this is what it should look like:

    SECTION {
        ...
        .bss (NOLOAD) :
        {
            *(.bss*)
            *(COMMON)
        } > ram
    }
    

You’ll note that the `.bss` section also includes `*(COMMON)`. This is a special input section where the compiler puts global uninitialized variables that go beyond file scope. `int foo;` goes there, while `static int foo;` does not. This allows the linker to merge multiple definitions into one symbol if they have the same name.

We indicate that this section is not loaded with the `NOLOAD` property. This is the only section property used in modern **script de ligação**s.

### `.stack` Section[](#stack-section)

We do the same thing for our `.stack` memory, since it is in RAM and not loaded. As the stack contains no symbols, we must explicitly reserve space for it by indicating its size. We also must align the stack on an 8-byte boundary per ARM Procedure Call Standards ([AAPCS](https://static.docs.arm.com/ddi0403/ec/DDI0403E_c_armv7m_arm.pdf)).

In order to achieve these goals, we turn to a special variable `.`, also known as the “location counter”. The location counter tracks the current offset into a given memory region. As sections are added, the location counter increments accordingly. You can force alignment or gaps by setting the location counter forward. You may not set it backwards, and the linker will throw an error if you try.

We set the location counter with the `ALIGN` function, to align the section, and use simple assignment and arithmetic to set the section size:

    STACK_SIZE = 0x2000; /* 8 kB */
    
    SECTION {
        ...
        .stack (NOLOAD) :
        {
            . = ALIGN(8);
            . = . + STACK_SIZE;
            . = ALIGN(8);
        } > ram
        ...
    }
    

Only one more section to go!

### `.data` Section[](#data-section)

The `.data` section contains static variables which have an initial value at boot. You will remember from our previous article that since RAM isn’t persisted while power is off, those sections need to be loaded from flash. At boot, the `Reset_Handler` copies the data from flash to RAM before the `main` function is called.

To make this possible, every section in our **script de ligação** has two addresses, its _load_ address (LMA) and its _virtual_ address (VMA). In a firmware context, the LMA is where your JTAG loader needs to place the section and the VMA is where the section is found during execution.

You can think of the LMA as the address “at rest” and the VMA the address during execution i.e. when the device is on and the program is running.

The syntax to specify the LMA and VMA is relatively straightforward: every address is two part: AT . In our case it looks like this:

    .data :
    {
        *(.data*);
    } > ram AT > rom  /* "> ram" is the VMA, "> rom" is the LMA */
    

Note that instead of appending a section to a memory region, you could also explicitly specify an address like so:

    .data ORIGIN(ram) /* VMA */ : AT(ORIGIN(rom)) /* LMA */
    {
        . = ALIGN(4);
        _sdata = .;
        *(.data*);
        . = ALIGN(4);
        _edata = .;
    }
    

Where `ORIGIN(<region>)` is a simple way to specify the start of a region. You can enter an address in hex as well.

And we’re done! Here’s our complete **script de ligação** with every section:

### Complete **script de ligação**[](#complete-linker-script)

    MEMORY
    {
      rom      (rx)  : ORIGIN = 0x00000000, LENGTH = 0x00040000
      ram      (rwx) : ORIGIN = 0x20000000, LENGTH = 0x00008000
    }
    
    STACK_SIZE = 0x2000;
    
    /* Section Definitions */
    SECTIONS
    {
        .text :
        {
            KEEP(*(.vectors .vectors.*))
            *(.text*)
            *(.rodata*)
        } > rom
    
        /* .bss section which is used for uninitialized data */
        .bss (NOLOAD) :
        {
            *(.bss*)
            *(COMMON)
        } > ram
    
        .data :
        {
            *(.data*);
        } > ram AT >rom
    
        /* stack section */
        .stack (NOLOAD):
        {
            . = ALIGN(8);
            . = . + STACK_SIZE;
            . = ALIGN(8);
        } > ram
    
        _end = . ;
    }
    

You can find the full details on **script de ligação** sections syntax in the [ld manual](https://sourceware.org/binutils/docs/ld/SECTIONS.html#SECTIONS).

## Variables
-----------------------

In the first post, our `ResetHandler` relied on seemingly magic variables to know the address of each of our sections of memory. It turns out, those variable came

In order to make section addresses available to code, the linker is able to generate symbols and add them to the program.

You can find the syntax in the [linker documentation](https://sourceware.org/binutils/docs/ld/Simple-Assignments.html#Simple-Assignments), it looks exactly like a C assignment: `symbol = expression;`

Here, we need:

1.  `_etext` the end of the code in `.text` section in flash.
2.  `_sdata` the start of the `.data` section in RAM
3.  `_edata` the end of the `.data` section in RAM
4.  `_sbss` the start of the `.bss` section in RAM
5.  `_ebss` the end of the `.bss` section in RAM

They are all relatively straightforward: we can assign our symbols to the value of the location counter (`.`) at the start and at the end of each section definition.

The code is below:

        .text :
        {
            KEEP(*(.vectors .vectors.*))
            *(.text.*)
            *(.rodata.*)
            _etext = .;
        } > rom
    
        .bss (NOLOAD) :
        {
            _sbss = . ;
            *(.bss .bss.*)
            *(COMMON)
            _ebss = . ;
        } > ram
    
        .data :
        {
            _sdata = .;
            *(.data*);
            _edata = .;
        } > ram AT >rom
    

One quirk of these linker-provided symbols: you must use a reference to them, never the variable themselves. For example, the following gets us a pointer to the start of the `.data` section:

    uint8_t *data_byte = &_sdata;
    

You can read more details about this in the [binutils docs](https://sourceware.org/binutils/docs/ld/Source-Code-Reference.html).

## Conclusão
------------

I hope this post gave you confidence in writing your own **script de ligação**s.

In my next post, we’ll talk about writing a bootloader to assist with loading and starting your application.

_EDIT: Post written!_ - [Writing a Bootloader from Scratch](/blog/how-to-write-a-bootloader-from-scratch)

As with previous posts, code examples are available on Github in the [zero to main repository](https://github.com/memfault/zero-to-main)

See anything you'd like to change? Submit a pull request or open an issue at [GitHub](https://github.com/memfault/interrupt)

Like Interrupt? [Subscribe](https://go.memfault.com/interrupt-subscribe) to get our latest posts straight to your mailbox.

![](/img/author/francois.jpg) [François Baldassari](/authors/francois) has worked on the embedded software teams at Sun, Pebble, and Oculus. He is currently the CEO of [Memfault](https://memfault.com).  
[](https://twitter.com/baldassarifr)[](https://www.linkedin.com/in/francois-baldassari)[](https://github.com/franc0is)
