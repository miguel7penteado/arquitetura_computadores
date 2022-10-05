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

Para alocar espaço de programa, o vinculador precisa saber quanta memória está disponível e em quais endereços essa memória existe. É para isso que serve a definição `MEMORY` no **script de ligação**.

A sintaxe para MEMORY é definida nos [documentos do binutils](https://sourceware.org/binutils/docs/ld/MEMORY.html#MEMORY) e é a seguinte:

```
    MEMORY
      {
        name [(attr)] : ORIGIN = origin, LENGTH = len
        …
      }
```    

Onde

* `name` é um nome que você deseja usar para esta região. Os nomes não têm significado, então você pode usar o que quiser. Você encontrará frequentemente “flash” e “ram” como nomes de região.
* `(attr)` são atributos opcionais para a região, como se é gravável (`w`), legível (`r`) ou executável (`x`). A memória flash geralmente é `(rx)`, enquanto a ram é `rwx`. Marcar uma região como não gravável não a torna magicamente protegida contra gravação: esses atributos destinam-se a descrever as propriedades da memória, não a configurá-la.
* `origin` é o endereço inicial da região de memória.
* `len` é o tamanho da região de memória, em bytes.

O mapa de memória para o chip SAMD21G18 que temos em nossa placa pode ser encontrado em seu [datasheet](http://ww1.microchip.com/downloads/en/DeviceDoc/SAMD21-Family-DataSheet-DS40001882D.pdf) em tabela 10-1, reproduzida abaixo.

**SAMD21G18 Memory Map**

| Memória       | Endereço Inicial | Tamanho    |
|---------------|------------------|------------|
| Flash Interna | 0x00000000       | 256 Kbytes |
| SRAM Interna  | 0x20000000       | 32 Kbytes  |

Transcrito em uma definição `MEMORY`, isso nos dá:

```
    MEMORY
    {
      rom      (rx)  : ORIGIN = 0x00000000, LENGTH = 0x00040000
      ram      (rwx) : ORIGIN = 0x20000000, LENGTH = 0x00008000
    }
```

## Definições de seção
----------------------

Código e dados são agrupados em seções, que são áreas contíguas da memória. Não há regras rígidas sobre quantas seções você deve ter ou quais devem ser, mas normalmente você deseja colocar símbolos na mesma seção se:

1. Eles devem estar na mesma região de memória, ou
2. Eles precisam ser inicializados juntos.

Em nosso post anterior, aprendemos sobre dois tipos de símbolos que são inicializados em massa:

1. Variáveis estáticas inicializadas que devem ser copiadas do flash
2. Variáveis estáticas não inicializadas que devem ser zeradas.

Nosso **script de ligação** se preocupa com mais duas coisas:

1. Código e dados constantes, que podem viver na memória somente leitura (por exemplo, flash)
2. Seções reservadas de RAM, como uma pilha ou heap

Por convenção, nomeamos essas seções da seguinte forma:

1. `.text` para código e constantes
2. `.bss` para dados não inicializados
3. `.stack` para nossa pilha
4. `.data` para dados inicializados

A [especificação elf](http://refspecs.linuxbase.org/elf/elf.pdf) contém uma lista completa. Seu firmware funcionará bem se você chamá-los de qualquer outra coisa, mas seus colegas podem ficar confusos e algumas ferramentas podem falhar de maneiras estranhas. A única restrição é que você não pode chamar sua seção `/DISCARD/`, que é uma palavra-chave reservada.

Primeiro, vamos ver o que acontece com nossos símbolos se não definirmos nenhuma dessas seções no **script de ligação**.

```
    MEMORY
    {
      rom      (rx)  : ORIGIN = 0x00000000, LENGTH = 0x00040000
      ram      (rwx) : ORIGIN = 0x20000000, LENGTH = 0x00008000
    }
    
    SECTIONS
    {
        /* empty! */
    }
```

O vinculador está perfeitamente feliz em vincular nosso programa a isso. Sondando o arquivo elf resultante com objdump, vemos o seguinte:

```bash
$ arm-none-eabi-objdump -h build/minimal.elf
build/minimal.elf:     file format elf32-littlearm
    
SYMBOL TABLE:
no symbols
```

Sem símbolos! Embora o vinculador seja capaz de fazer suposições que permitirão vincular símbolos com pouca informação, ele precisa pelo menos saber qual deve ser o ponto de entrada ou quais símbolos colocar na seção de texto.

### `.text` Section[](#text-section)

Vamos começar adicionando nossa seção `.text`. Queremos essa seção em ROM. A sintaxe é simples:

    SECTIONS
    {
        .text :
        {
    
        } > rom
    }
    

Isso define uma seção chamada `.text` e a adiciona à ROM. Agora precisamos dizer ao vinculador o que colocar nessa seção. Isso é feito listando todas as seções de nossos arquivos de objeto de entrada que queremos em `.text`.

Para descobrir quais seções estão em nosso arquivo objeto, podemos mais uma vez usar `objdump`:

```bash
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
```

Vemos que cada um dos nossos símbolos tem uma seção. Isso se deve ao fato de que compilamos nosso firmware com os sinalizadores `-ffunction-sections` e `-fdata-sections`. Se não as tivéssemos incluído, o compilador estaria livre para mesclar várias funções em uma única seção `text.<algum identificador>`.

Para colocar todas as nossas funções na seção `.text` em nosso **script de ligação**, usamos a seguinte sintaxe: `<filename>(<section>)`, onde `filename` é o nome da entrada arquivos cujos símbolos queremos incluir, e `section` é o nome das seções de entrada. Como queremos todas as seções `.text...` em todos os arquivos, usamos o curinga `*`:

    .text :
    {
        KEEP(*(.vector*))
        *(.text*)
    } > rom
    

Observe a seção de entrada `.vector`, que contém funções que queremos manter no início da nossa seção `.text`. É assim que o `Reset_Handler` está onde o MCU espera que esteja. Falaremos mais sobre a tabela de vetores em um post futuro.

Despejando nosso arquivo elf, agora vemos todas as nossas funções (mas nenhum dado)!

```bash
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
```

### `.bss` Section[](#bss-section)

Agora, vamos cuidar do nosso `.bss`. Lembre-se, esta é a seção onde colocamos a memória estática não inicializada. `.bss` deve ser reservado no mapa de memória, mas não há nada para carregar, pois todas as variáveis são inicializadas em zero. Dessa forma, é assim que deve ficar:

```
    SECTION {
        ...
        .bss (NOLOAD) :
        {
            *(.bss*)
            *(COMMON)
        } > ram
    }
``` 

Você notará que a seção `.bss` também inclui `*(COMMON)`. Esta é uma seção de entrada especial onde o compilador coloca variáveis globais não inicializadas que vão além do escopo do arquivo. `int foo;` vai lá, enquanto `static int foo;` não. Isso permite que o vinculador mescle várias definições em um símbolo se elas tiverem o mesmo nome.

Indicamos que esta seção não está carregada com a propriedade `NOLOAD`. Esta é a única propriedade de seção usada em **scripts de ligação** modernos.

### `.stack` Section[](#stack-section)

Fazemos a mesma coisa para nossa memória `.stack`, já que ela está na RAM e não está carregada. Como a pilha não contém símbolos, devemos reservar espaço explicitamente para ela indicando seu tamanho. Também devemos alinhar a pilha em um limite de 8 bytes por Padrões de Chamada de Procedimento ARM ([AAPCS](https://static.docs.arm.com/ddi0403/ec/DDI0403E_c_armv7m_arm.pdf)).

Para atingir esses objetivos, recorremos a uma variável especial `.`, também conhecida como “contador de localização”. O contador de localização rastreia o deslocamento atual em uma determinada região de memória. À medida que as seções são adicionadas, o contador de local é incrementado de acordo. Você pode forçar o alinhamento ou as lacunas definindo o contador de localização para frente. Você não pode configurá-lo para trás e o vinculador lançará um erro se você tentar.

Definimos o contador de localização com a função `ALIGN`, para alinhar a seção, e usamos atribuição simples e aritmética para definir o tamanho da seção:

```
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
``` 

Falta apenas mais uma seção!

### `.data` Section

A seção `.data` contém variáveis estáticas que possuem um valor inicial na inicialização. Você se lembrará de nosso artigo anterior que, como a RAM não persiste enquanto a energia está desligada, essas seções precisam ser carregadas do flash. Na inicialização, o `Reset_Handler` copia os dados do flash para a RAM antes que a função `main` seja chamada.

Para tornar isso possível, cada seção em nosso **script de ligação** tem dois endereços, seu endereço _load_ (LMA) e seu endereço _virtual_ (VMA). Em um contexto de firmware, o LMA é onde seu carregador JTAG precisa colocar a seção e o VMA é onde a seção é encontrada durante a execução.

Você pode pensar no LMA como o endereço “em repouso” e no VMA o endereço durante a execução, ou seja, quando o dispositivo está ligado e o programa está em execução.

A sintaxe para especificar o LMA e o VMA é relativamente simples: cada endereço tem duas partes: AT . No nosso caso fica assim:

```
    .data :
    {
        *(.data*);
    } > ram AT > rom  /* "> ram" is the VMA, "> rom" is the LMA */
```

Observe que, em vez de anexar uma seção a uma região de memória, você também pode especificar explicitamente um endereço da seguinte forma:

```
    .data ORIGIN(ram) /* VMA */ : AT(ORIGIN(rom)) /* LMA */
    {
        . = ALIGN(4);
        _sdata = .;
        *(.data*);
        . = ALIGN(4);
        _edata = .;
    }
``` 

Onde `ORIGIN(<region>)` é uma maneira simples de especificar o início de uma região. Você também pode inserir um endereço em hexadecimal.

E terminamos! Aqui está nosso **script de ligação** completo com cada seção:

### **script de ligação completo!**

```C
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
```

Você pode encontrar todos os detalhes sobre a sintaxe das seções **script de ligação** no [ld manual](https://sourceware.org/binutils/docs/ld/SECTIONS.html#SECTIONS).

## Variáveis
-----------------------
No primeiro post, nosso `ResetHandler` dependia de variáveis aparentemente mágicas para saber o endereço de cada uma de nossas seções de memória. Acontece que essas variáveis vieram

Para disponibilizar endereços de seção para código, o vinculador é capaz de gerar símbolos e adicioná-los ao programa.

Você pode encontrar a sintaxe na [documentação do linker](https://sourceware.org/binutils/docs/ld/Simple-Assignments.html#Simple-Assignments), ela se parece exatamente com uma atribuição C: `symbol = expression; `

Aqui, precisamos:

1. `_etext` o final do código na seção `.text` em flash.
2. `_sdata` o início da seção `.data` na RAM
3. `_edata` o final da seção `.data` na RAM
4. `_sbss` o início da seção `.bss` na RAM
5. `_ebss` o final da seção `.bss` na RAM

Eles são todos relativamente simples: podemos atribuir nossos símbolos ao valor do contador de localização (`.`) no início e no final de cada definição de seção.

O código está abaixo:

```C
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
```    

Uma peculiaridade desses símbolos fornecidos pelo vinculador: você deve usar uma referência a eles, nunca a própria variável. Por exemplo, o seguinte nos dá um ponteiro para o início da seção `.data`:

```C
    uint8_t *data_byte = &_sdata;
``` 

Você pode ler mais detalhes sobre isso nos [documentos do binutils](https://sourceware.org/binutils/docs/ld/Source-Code-Reference.html).

## Conclusão
------------
Espero que este post tenha lhe dado confiança para escrever seu próprio **script de ligação**s.

No meu próximo post, falaremos sobre como escrever um bootloader para ajudar a carregar e iniciar seu aplicativo.

_EDIT: Post escrito!_ - [Escrevendo um bootloader do zero](/blog/how-to-write-a-bootloader-from-scratch)

Assim como nas postagens anteriores, os exemplos de código estão disponíveis no Github no [repositório zero to main](https://github.com/memfault/zero-to-main)

Vê algo que você gostaria de mudar? Envie um pull request ou abra um problema no [GitHub](https://github.com/memfault/interrupt)

Like Interrupt? [Subscribe](https://go.memfault.com/interrupt-subscribe) to get our latest posts straight to your mailbox.

![](/img/author/francois.jpg) [François Baldassari](/authors/francois) has worked on the embedded software teams at Sun, Pebble, and Oculus. He is currently the CEO of [Memfault](https://memfault.com).  
[](https://twitter.com/baldassarifr)[](https://www.linkedin.com/in/francois-baldassari)[](https://github.com/franc0is)
