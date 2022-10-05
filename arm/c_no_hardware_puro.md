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

Power on![](#power-on)
----------------------

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

If I’m reading this correctly, our inital SP is `0x20002000`, and our start address pointer is `0x000000c1`.

Let’s dump our symbols to see which one is at `0x000000c1`.

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

That’s odd! Our main function is found at `0x000002ac`. No symbol at `0x000000c1`, but a `Reset_Handler` symbol at `0x000000c0`.

It turns out that the lowest bit of the PC is used to indicate thumb2 instructions, which is one of the two instruction sets supported by ARM processors, so `Reset_Handler` is what we’re looking for (for more details check out section A4.1.1 in the ARMv6-M manual).

Writing a Reset\_Handler[](#writing-a-reset_handler)
----------------------------------------------------

Unfortunately, the Reset\_Handler is often an inscrutable mess of Assembly code. See the [nRF52 SDK startup](https://github.com/NordicSemiconductor/nrfx/blob/293f553ed9551c1fdfd05eac48e75bbdeb4e7290/mdk/gcc_startup_nrf52.S#L217) file for example.

Instead of going through this file line-by-line, let’s see if we can write a minimal Reset\_Handler from first principles.

Here again, ARM’s Technical Reference Manuals are useful. [Section 5.9.2 of the Cortex-M3 TRM](https://developer.arm.com/docs/ddi0337/e/exceptions/resets/intended-boot-up-sequence) contains the following table:

Reset boot-up behavior

Action

Description

Initialize variables

Any global/static variables must be setup. This includes initializing the BSS variable to 0, and copying initial values from ROM to RAM for non-constant variables.

\[Setup stacks\]

If more than one stack is be used, the other banked SPs must be initialized. The current SP can also be changed to Process from Main.

\[Initialize any runtime\]

Optionally make calls to C/C++ runtime init code to enable use of heap, floating point, or other features. This is normally done by `__main` from the C/C++ library.

  
So, our ResetHandler is responsible for initializing static and global variables, and starting our program. This mirrors what the C standards tells us:

> All objects with static storage duration shall be initialized (set to their initial values) before program startup. The manner and timing of such initialization are otherwise unspecified.

(Section 5.1.2, Execution environment)

In practice this means that given the following snippet:

    static uint32_t foo;
    static uint32_t bar = 2;
    

Our `Reset_Handler` needs to make sure that the memory at `&foo` is 0x00000000, and the memory at `&bar` is 0x00000002.

We cannot just go and initialize each variable one by one. Instead, we rely on the compiler (technically, the linker) to put all those variables in the same place so we can initialize them in one fell swoop.

For static variables that must be zeroed, the linker gives us `_sbss` and `_ebss` as start and end addresses. We can therefore do:

    /* Clear the zero segment */
    for (uint32_t *bss_ptr = &_sbss; bss_ptr < &_ebss;) {
        *bss_ptr++ = 0;
    }
    

For static variables with an init value, the linker gives us:

*   `_etext` as the address the init values are stored at
*   `_sdata` as the address the static variables live at
*   `_edata` as the end of the static variables memory

We then can do:

    uint32_t *init_values_ptr = &_etext;
    uint32_t *data_ptr = &_sdata;
    if (init_values_ptr != data_ptr) {
        for (; data_ptr < &_edata;) {
            *data_ptr++ = *init_values_ptr++;
        }
    }
    

Putting it together, we can write our `Reset_Handler`

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
    

We still need to start our program! That’s achieved with a simple call to `main()`.

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
    

You will note that we added two things:

1.  An infinite loop after `main()`, so we do not run off into the weeds if the main function returns
2.  Workaround for chip bugs which are best taken care of before our program starts. Sometimes these are wrapped in a `SystemInit` function called by the `Reset_Handler` before `main`. This is the approach [taken by Nordic](https://github.com/NordicSemiconductor/nrfx/blob/6f54f689e9555ea18f9aca87caf44a3419e5dd7a/mdk/system_nrf52811.c#L60).

Closing[](#closing)
-------------------

All the code used in this blog post is available on [Github](https://github.com/memfault/zero-to-main/tree/master/minimal).

See anything you'd like to change? Submit a pull request or open an issue at [GitHub](https://github.com/memfault/interrupt)

More complex programs often require a more complicated `Reset_Handler`. For example:

1.  Relocatable code must be copied over
2.  If our program relies on libc, we must initialize it  
    _EDIT: Post written!_ - [From Zero to main(): Bootstrapping libc with Newlib](/blog/boostrapping-libc-with-newlib)
3.  More complex memory layouts can add a few copy / zero loops

We’ll cover all of them in future posts. But before that, we’ll talk about how the magical memory region variables come about, how our `Reset_Handler`’s address ends up at `0x00000004`, and how to write a linker script in our next post!

_EDIT: Post written!_ - [From Zero to main(): Demystifying Firmware Linker Scripts](/blog/how-to-write-linker-scripts-for-firmware)

![](/img/author/francois.jpg) [François Baldassari](/authors/francois) has worked on the embedded software teams at Sun, Pebble, and Oculus. He is currently the CEO of [Memfault](https://memfault.com).  
[](https://twitter.com/baldassarifr)[](https://www.linkedin.com/in/francois-baldassari)[](https://github.com/franc0is)
