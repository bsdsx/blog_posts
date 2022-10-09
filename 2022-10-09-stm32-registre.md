###### 202210090800 stm32 bluepill
# FreeBSD et bluepill

A force de suivre les vidéos de [Sieur Rancune](https://www.twitch.tv/rancune_) j'ai fini par craquer pour une poignée de [bluepill](https://stm32-base.org/boards/STM32F103C8T6-Blue-Pill.html).
Parce qu'il est conseillé sur tous les interweb d'utiliser **arm-none-eabi-gcc** pour programmer ces charmantes bestioles j'ai décidé d'utiliser clang/llvm livré de base.

Si allumer une led constitue le 'Hello World!' de l'électronique, j'ai découvert le niveau -1 en lisant [cette série de billets](https://vivonomicon.com/2018/04/02/bare-metal-stm32-programming-part-1-hello-arm/): placer une valeur particulière dans un registre.

## Les sources

Le fichier [core.S](https://raw.githubusercontent.com/WRansohoff/STM32F0_minimal/master/core.S) requiert 3 modifications mineures:

- adapter le type de cpu
- modifier la casse de la fonction **reset_handler**
- renommer une variable

    $ fetch -o - https://raw.githubusercontent.com/WRansohoff/STM32F0_minimal/master/core.S | sed -e 's/cortex-m0/cortex-m3/' -e 's/reset_handler/Reset_Handler/g' -e 's/_estack/__end_stack/' > core.S
    $ clang --target=arm-none-eabi -mcpu=cortex-m3 -mfloat-abi=soft -c core.S -o core.o

## Le binaire

Pour que mon binaire final soit adapté à ma carte, je dois récupérer un 'linker script':

    $ fetch https://github.com/STM32-base/STM32-base/archive/refs/heads/master.zip
    $ tar xf master.zip STM32-base-master/linker
    $ clang --target=arm-none-eabi -mcpu=cortex-m3 -mfloat-abi=soft -nostdlib -L STM32-base-master/linker -T STM32-base-master/linker/STM32F1xx/STM32F103xB.ld core.o -o main.elf

C'est pour se conformer à ce script qu'il faut modifier le fichier **core.S**.

##  Les soucis commencent

Avant d'envoyer le fichier sur la carte, il faut le convertir:

    $ objcopy -O binary main.elf main.bin
    $ du -Ah main.bin 
    384M  main.bin

Là je crois qu'on dépasse les bornes des limites: ça fait beaucoup de méga pour placer une valeur dans un registre. Si je regarde mon fichier elf:

    $ readelf -l main.elf | grep 0x
    Entry point 0x8000009
      LOAD           0x010000 0x08000000 0x08000000 0x0001c 0x0001c R E 0x10000
      LOAD           0x020000 0x20000000 0x20000000 0x00600 0x00600 RW  0x10000
      GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0
      ARM_EXIDX      0x01001c 0x0800001c 0x0800001c 0x00000 0x00000 R   0x4

Je peux me passer de **GNU_STACK** avec l'option **-z nognustack**:

    $ clang --target=arm-none-eabi -mcpu=cortex-m3 -mfloat-abi=soft -z nognustack -nostdlib -L STM32-base-master/linker -T STM32-base-master/linker/STM32F1xx/STM32F103xB.ld core.o -o main.elf
    $ readelf -l main.elf | grep 0x
    Entry point 0x8000009
      LOAD           0x010000 0x08000000 0x08000000 0x0001c 0x0001c R E 0x10000
      LOAD           0x020000 0x20000000 0x20000000 0x00600 0x00600 RW  0x10000
      ARM_EXIDX      0x01001c 0x0800001c 0x0800001c 0x00000 0x00000 R   0x4

Mais cela ne corrige pas le problème. J'ai fini par comprendre qu'une section n'avait pas le bon type:

    $ readelf -S main.elf.OK | grep ._user_heap_stack
    [ 6] ._user_heap_stack NOBITS          20000000 020000 000600 00  WA  0   0  1
    $ readelf -S main.elf.KO | grep ._user_heap_stack
    [ 7] ._user_heap_stack PROGBITS        20000000 020000 000600 00  WA  0   0  1

La correction la plus simple que j'ai trouvé est de passer cette section en 'NOLOAD':

    $ grep ._user_heap_stack STM32-base-master/linker/common.ld
    ._user_heap_stack (NOLOAD) : {
    $ clang --target=arm-none-eabi -mcpu=cortex-m3 -mfloat-abi=soft -z nognustack -nostdlib -L STM32-base-master/linker -T STM32-base-master/linker/STM32F1xx/STM32F103xB.ld core.o -o main.elf
    $ readelf -S main.elf | grep ._user_heap_stack
    [ 8] ._user_heap_stack NOBITS          20000000 020000 000600 00  WA  0   0  1
    $ objcopy -O binary main.elf main.bin
    $ du -Ah main.bin
    512B  main.bin

C'est déjà plus raisonnable.

## Les soucis continuent

Il est temps de programmer la carte. J'utilise donc un adaptateur usb st link v2 et devel/openocd avec la configuration suivante:

    $ cat openocd.cfg
    source [find interface/stlink.cfg]
    transport select hla_swd
    source [ find target/stm32f1x.cfg]
    reset_config none separate

Je connecte la bluepill à l'adaptateur, je branche l'adaptateur et paf:

    $ openocd -f openocd.cfg -c "init"
    Open On-Chip Debugger 0.11.0
    Licensed under GNU GPL v2
    For bug reports, read
    	http://openocd.org/doc/doxygen/bugs.html
    Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
    none separate
    
    Info : clock speed 1000 kHz
    Info : STLINK V2J17S4 (API v2) VID:PID 0483:3748
    Info : Target voltage: 3.137032
    Warn : UNEXPECTED idcode: 0x2ba01477
    Error: expected 1 of 1: 0x1ba01477

Pour qu'openocd daigne utiliser mon adaptateur je dois lui dire qu'il est "de confiance" (kofkof):

    $ cat openocd.cfg
    set CPUTAPID 0x2ba01477
    source [find interface/stlink.cfg]
    transport select hla_swd
    source [ find target/stm32f1x.cfg]
    reset_config none separate
    
    $ openocd -f openocd.cfg -c "init"
    Open On-Chip Debugger 0.11.0
    Licensed under GNU GPL v2
    For bug reports, read
    	http://openocd.org/doc/doxygen/bugs.html
    Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
    none separate
    
    Info : clock speed 1000 kHz
    Info : STLINK V2J17S4 (API v2) VID:PID 0483:3748
    Info : Target voltage: 3.138581
    Info : stm32f1x.cpu: hardware has 6 breakpoints, 4 watchpoints
    Info : starting gdb server for stm32f1x.cpu on 3333
    Info : Listening on port 3333 for gdb connections
    Info : Listening on port 6666 for tcl connections
    Info : Listening on port 4444 for telnet connections

## Et quand y'en a plus y'en a encore

On va facilement trouver des exemples d'utilisation conjointe d'openocd et de gdb. Pas de bol pour moi, le debugger llvm n'est pas compatible, d'après https://stackoverflow.com/questions/36287351/how-to-setup-lldb-with-openocd-and-jtag-board:

    "from what we were researching, it is not possible to debug remote (bare-metal!) targets with lldb without writing extra code."

Et autant dire que l'extra code en python, je passe.

## Le bout du tunnel

Je vais donc utiliser l'interface "telnet" d'openocd:

    $ echo 'reset halt' | nc -N 127.0.0.1 4444
    Open On-Chip Debugger
    > reset halt
    target halted due to debug-request, current mode: Thread 
    xPSR: 0x01000000 pc: 0x1fffe61c msp: 0x20000180
    
    $ echo 'stm32f1x mass_erase 0' | nc -N 127.0.0.1 4444
    Open On-Chip Debugger
    > stm32f1x mass_erase 0
    device id = 0x20036410
    flash size = 25616kbytes
    stm32x mass erase complete
    
    $ md5 main.bin 
    MD5 (main.bin) = 09c6d874e631d40370b70a7bcc120f41
    
    $ echo 'flash write_image erase main.bin 0x8000000' | nc -N 127.0.0.1 4444
    Open On-Chip Debugger
    > flash write_image erase main.bin 0x8000000
    auto erase enabled
    wrote 1024 bytes from file main.bin in 0.087373s (11.445 KiB/s)
    
    $ echo 'reset run' | nc -N 127.0.0.1 4444
    Open On-Chip Debugger
    > reset run
    
    $ echo 'halt' | nc -N 127.0.0.1 4444
    Open On-Chip Debugger
    > halt
    target halted due to debug-request, current mode: Handler HardFault
    xPSR: 0x00000003 pc: 0x012fff1e msp: 0xe59efff8
    
    $ echo 'reg' | nc -N 127.0.0.1 4444
    Open On-Chip Debugger
    > reg
    ===== arm v7m registers
    (0) r0 (/32): 0x008fec14
    (1) r1 (/32): 0x00000000
    (2) r2 (/32): 0x00000000
    (3) r3 (/32): 0x00000000
    (4) r4 (/32): 0x00000000
    (5) r5 (/32): 0x00000000
    (6) r6 (/32): 0x00000000
    (7) r7 (/32): 0xdeadbeef
    (8) r8 (/32): 0x00000000
    (9) r9 (/32): 0x00000000
    (10) r10 (/32): 0x00000000
    (11) r11 (/32): 0x00000000
    (12) r12 (/32): 0x00000000
    (13) sp (/32): 0x20005000
    (14) lr (/32): 0xffffffff
    (15) pc (/32): 0x08000012
    (16) xPSR (/32): 0x01000000
    (17) msp (/32): 0x20005000
    (18) psp (/32): 0x00000000
    (20) primask (/1): 0x00
    (21) basepri (/8): 0x00
    (22) faultmask (/1): 0x00
    (23) control (/3): 0x00
    ===== Cortex-M DWT registers

Pour être sûr de mon coup, j'essaie avec les valeurs suivantes:

- OxCAFECAFE: MD5 (main.bin) = 45e3b36afd8ebd902675c26e804c21b5
- OxBABEBABE: MD5 (main.bin) = 4f03cd6b4a29ca80dc28fc49401cb1c5
- Ox12345678: MD5 (main.bin) = 908fc5196f85ade8321d1d951d8035f7

A moi les <blink>leds</blink> qui clignottent !

Commentaires: [https://github.com/bsdsx/blog_posts/issues/15](https://github.com/bsdsx/blog_posts/issues/15)
