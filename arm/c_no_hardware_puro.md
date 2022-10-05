# Do nada ao main(): C no Hardware Puro
--------------------------------------------------------------------

#### Original de *François Baldassari*

Trabalhando em software embarcado, desenvolve-se rapidamente um respeito quase religioso pelos axiomas da programação C embarcada:

1.  O ponto de entrada do teu programa deve ser nomeado “main”.
2.  Você deve inicializar suas variáveis estáticas, caso contrário, a Máquina deve defini-las para zero.
3.  Tu implementarás As Interrupções. HardFault\_Handler principal entre eles, mas também SysTick\_Handler.

Pergunte a um engenheiro de onde vêm essas regras e eles acenarão para arquivos de inicialização enigmáticos implementados na montagem. Muitas vezes esses arquivos são copiados e colados de projeto para projeto. Raramente eles são lidos, muito menos modificados.

Ao longo da série de posts *do nada ao main()*, desmistificamos o que acontece entre quando a energia é aplicada e sua função principal é chamada. No processo, aprenderemos como inicializar um ambiente C, implementar um carregador de inicialização, realocar código e muito mais!

## Montando o palco
-------------------

Embora a maioria dos conceitos e códigos apresentados nesta série funcionem para todos os MCUs da série Cortex-M, nossos exemplos visam o processador SAMD21G18 da Atmel. Este é um chip Cortex-M0+ encontrado em várias placas de desenvolvimento acessíveis.


Especificamente, estamos usando:

*   Adafruit’s [Metro M0 Express](https://www.adafruit.com/product/3505) como nossa placa de desenvolvimento,
*   Um simples [CMSIS-DAP Adapter](https://www.adafruit.com/product/2764)
*   OpenOCD (the [Arduino fork](https://github.com/arduino/OpenOCD)) para programação

Em cada caso, implementaremos um aplicativo simples de LED piscando. Não é particularmente interessante por si só, mas por uma questão de completude, você pode encontrar o código reproduzido abaixo.

```c
    #include <samd21g18a.h>
    
    #include <port.h>
    #include <stdbool.h>
    #include <stdint.h>
    
    #define LED_0_PIN PIN_PA17
    
    static void set_output(const uint8_t pin) {
      struct port_config config_port_pin;
      port_get_config_defaults(&config_port_pin);
      config_port_pin.direction = PORT_PIN_DIR_OUTPUT;
      port_pin_set_config(pin, &config_port_pin);
      port_pin_set_output_level(pin, false);
    }
    
    int main() {
      set_output(LED_0_PIN);
      while (true) {
        port_pin_toggle_output_level(LED_0_PIN);
        for (volatile int i = 0; i < 100000; ++i) {}
      }
    }
    
```

## Ligando o botão de Power
---------------------------

Então, como chegamos ao principal? Tudo o que podemos dizer pela observação é que aplicamos energia à placa e nosso código começou a ser executado. Deve haver um comportamento intrínseco ao chip que define como o código é executado.

E de fato, existe! Examinando o [ARMv6-M Technical Reference Manual](https://static.docs.arm.com/ddi0419/d/DDI0419D_armv6m_arm.pdf), que é o manual de arquitetura subjacente para o Cortex-M0+, podemos encontrar alguns pseudo -código que descreve o comportamento de redefinição:

```c
    // B1.5.5 TakeReset()
    // ============
    TakeReset()
        VTOR = Zeros(32);
        for i = 0 to 12
            R[i] = bits(32) UNKNOWN;
        bits(32) vectortable = VTOR;
        CurrentMode = Mode_Thread;
        LR = bits(32) UNKNOWN; // Value must be initialised by software
        APSR = bits(32) UNKNOWN; // Flags UNPREDICTABLE from reset
        IPSR<5:0> = Zeros(6); // Exception number cleared at reset
        PRIMASK.PM = '0'; // Priority mask cleared at reset
        CONTROL.SPSEL = '0'; // Current stack is Main
        CONTROL.nPRIV = '0'; // Thread is privileged
        ResetSCSRegs(); // Catch-all function for System Control Space reset
        for i = 0 to 511 // All exceptions Inactive
            ExceptionActive[i] = '0';
        ClearEventRegister(); // See WFE instruction for more information
        SP_main = MemA[vectortable,4] AND 0xFFFFFFFC<31:0>;
        SP_process = ((bits(30) UNKNOWN):'00');
        start = MemA[vectortable+4,4]; // Load address of reset routine
        BLXWritePC(start); // Start execution of reset routine
```

Em resumo, o chip faz o seguinte:

*   Redefina o endereço da tabela de vetores para `0x00000000`
*   Desabilitar todas as interrupções
*   Carregar o SP do endereço `0x00000000`
*   Carregar o PC do endereço `0x00000004`

“Mistério resolvido!”, você dirá. Nossa função `main` deve estar no endereço `0x00000004`!

Vamos verificar.

Primeiro, despejamos nosso arquivo `bin` para ver qual endereço `0x0000000` e `0x00000004` contêm:

```bash
    usuario@maquina$:\xxd build/minimal/minimal.bin  | head
    00000000: 0020 0020 c100 0000 b500 0000 bb00 0000  . . ............
    00000010: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    00000060: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    00000070: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    00000080: 0000 0000 0000 0000 0000 0000 0000 0000  ................
    00000090: 0000 0000 0000 0000 0000 0000 0000 0000  ................
```

Se estou lendo isso corretamente, nosso SP inicial é 0x20002000 e nosso ponteiro de endereço inicial é 0x000000c1.

Vamos despejar nossos símbolos para ver qual deles está em `0x000000c1`.

```bash
    usuario@maquina$:\arm-none-eabi-objdump -t build/minimal.elf | sort
    ...
    000000b4 g     F .text  00000006 NMI_Handler
    000000ba g     F .text  00000006 HardFault_Handler
    000000c0 g     F .text  00000088 Reset_Handler
    00000148 l     F .text  0000005c system_pinmux_get_group_from_gpio_pin
    000001a4 l     F .text  00000020 port_get_group_from_gpio_pin
    000001c4 l     F .text  00000022 port_get_config_defaults
    000001e6 l     F .text  0000004e port_pin_set_output_level
    00000234 l     F .text  00000038 port_pin_toggle_output_level
    0000026c l     F .text  00000040 set_output
    000002ac g     F .text  0000002c main
    ...
```

Isso é estranho! Nossa função principal é encontrada em `0x000002ac`. Nenhum símbolo em `0x000000c1`, mas um símbolo `Reset_Handler` em `0x000000c0`.

Acontece que o bit mais baixo do PC é usado para indicar instruções thumb2, que é um dos dois conjuntos de instruções suportados por processadores ARM, então `Reset_Handler` é o que estamos procurando (para mais detalhes, confira a seção A4. 1.1 no manual ARMv6-M).

## Escrevendo um Reset_Handler
----------------------------------------------------

Infelizmente, o Reset\_Handler geralmente é uma bagunça indecifrável de código Assembly. Veja o arquivo [nRF52 SDK startup](https://github.com/NordicSemiconductor/nrfx/blob/293f553ed9551c1fdfd05eac48e75bbdeb4e7290/mdk/gcc_startup_nrf52.S#L217) por exemplo.

Em vez de passar por este arquivo linha por linha, vamos ver se podemos escrever um Reset\_Handler mínimo a partir dos primeiros princípios.

Aqui, novamente, os Manuais de Referência Técnica da ARM são úteis. [Seção 5.9.2 do Cortex-M3 TRM](https://developer.arm.com/docs/ddi0337/e/exceptions/resets/intended-boot-up-sequence) contém a seguinte tabela:

**Reset boot-up behavior**

| Ação                                    | Descrição                                                                                                                                                                                                         |   |   |   |
|-----------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---|---|---|
| Inicializar Variáveis                   | Quaisquer variáveis globais/estáticas devem ser configuradas. Isso inclui inicializar a variável BSS como 0 e copiar valores iniciais de ROM para RAM para variáveis não constantes.                              |   |   |   |
| [Configurar a pilha]                    | Se mais de uma pilha for usada, os outros SPs em banco devem ser inicializados. O SP atual também pode ser alterado para Process from Main.                                                                       |   |   |   |
| [Inicialize qualquer tempo de execução] | Opcionalmente, faça chamadas para o código de inicialização do tempo de execução C/C++ para habilitar o uso de heap, ponto flutuante ou outros recursos. Isso normalmente é feito por __main da biblioteca C/C++. |   |   |   |

  
Assim, nosso ResetHandler é responsável por inicializar variáveis estáticas e globais, e iniciar nosso programa. Isso reflete o que os padrões C nos dizem:

> Todos os objetos com duração de armazenamento estático devem ser inicializados (definidos para seus valores iniciais) antes da inicialização do programa. A maneira e o tempo de tal inicialização não são especificados.

(Seção 5.1.2, Ambiente de execução)

Na prática, isso significa que, dado o seguinte trecho:

```c
    static uint32_t foo;
    static uint32_t bar = 2;
```

Nosso `Reset_Handler` precisa garantir que a memória em `&foo` seja 0x00000000, e a memória em `&bar` seja 0x00000002.

Não podemos simplesmente inicializar cada variável uma por uma. Em vez disso, contamos com o compilador (tecnicamente, o vinculador) para colocar todas essas variáveis no mesmo lugar para que possamos inicializá-las de uma só vez.

Para variáveis estáticas que devem ser zeradas, o linker nos dá `_sbss` e `_ebss` como endereços inicial e final. Podemos, portanto, fazer:

```c
    /* Clear the zero segment */
    for (uint32_t *bss_ptr = &_sbss; bss_ptr < &_ebss;) {
        *bss_ptr++ = 0;
    }
```

Para variáveis estáticas com um valor de inicialização, o vinculador nos fornece:

* `_etext` como o endereço em que os valores de inicialização são armazenados
* `_sdata` como o endereço em que as variáveis estáticas vivem
* `_edata` como o fim da memória de variáveis estáticas

Podemos então fazer:

```c
    uint32_t *init_values_ptr = &_etext;
    uint32_t *data_ptr = &_sdata;
    if (init_values_ptr != data_ptr) {
        for (; data_ptr < &_edata;) {
            *data_ptr++ = *init_values_ptr++;
        }
    }
```

Juntando, podemos escrever nosso `Reset_Handler`

```c
    void Reset_Handler(void)
    {
        /* Copy init values from text to data */
        uint32_t *init_values_ptr = &_etext;
        uint32_t *data_ptr = &_sdata;
    
        if (init_values_ptr != data_ptr) {
            for (; data_ptr < &_edata;) {
                *data_ptr++ = *init_values_ptr++;
            }
        }
    
        /* Clear the zero segment */
        for (uint32_t *bss_ptr = &_sbss; bss_ptr < &_ebss;) {
            *bss_ptr++ = 0;
        }
    }
```

Ainda precisamos começar nosso programa! Isso é conseguido com uma simples chamada para `main()`.

```c
    void Reset_Handler(void)
    {
        /* Copy init values from text to data */
        uint32_t *init_values_ptr = &_etext;
        uint32_t *data_ptr = &_sdata;
    
        if (init_values_ptr != data_ptr) {
            for (; data_ptr < &_edata;) {
                *data_ptr++ = *init_values_ptr++;
            }
        }
    
        /* Clear the zero segment */
        for (uint32_t *bss_ptr = &_sbss; bss_ptr < &_ebss;) {
            *bss_ptr++ = 0;
        }
    
        /* Overwriting the default value of the NVMCTRL.CTRLB.MANW bit (errata reference 13134) */
        NVMCTRL->CTRLB.bit.MANW = 1;
    
        /* Branch to main function */
        main();
    
        /* Infinite loop */
        while (1);
    }
```

Você notará que adicionamos duas coisas:

1. Um loop infinito depois de `main()`, para não corrermos para as ervas daninhas se a função principal retornar
2. Solução alternativa para bugs de chip que devem ser resolvidos antes do início do nosso programa. Algumas vezes estes são encapsulados em uma função `SystemInit` chamada pelo `Reset_Handler` antes de `main`. Esta é a abordagem [tomada pela Nordic](https://github.com/NordicSemiconductor/nrfx/blob/6f54f689e9555ea18f9aca87caf44a3419e5dd7a/mdk/system_nrf52811.c#L60).

# Conclusão
-----------

Todo o código usado nesta postagem do blog está disponível no [Github](https://github.com/memfault/zero-to-main/tree/master/minimal).

Vê algo que você gostaria de mudar? Envie um pull request ou abra um problema no [GitHub](https://github.com/memfault/interrupt)

Programas mais complexos geralmente requerem um `Reset_Handler` mais complicado. Por exemplo:

1. O código realocável deve ser copiado
2. Se nosso programa depende de libc, devemos inicializá-lo
     _EDIT: Post escrito!_ - [From Zero to main(): Bootstrapping libc with Newlib](/blog/boostrapping-libc-with-newlib)
3. Layouts de memória mais complexos podem adicionar alguns loops de cópia/zero

Abordaremos todos eles em posts futuros. Mas antes disso, falaremos sobre como surgem as variáveis mágicas da região de memória, como o endereço do nosso `Reset_Handler` termina em `0x00000004` e como escrever um script de linker em nosso próximo post!

![](/img/author/francois.jpg) [François Baldassari](/authors/francois) has worked on the embedded software teams at Sun, Pebble, and Oculus. He is currently the CEO of [Memfault](https://memfault.com).  
[](https://twitter.com/baldassarifr)[](https://www.linkedin.com/in/francois-baldassari)[](https://github.com/franc0is)
