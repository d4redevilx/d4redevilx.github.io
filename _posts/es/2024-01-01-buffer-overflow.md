---
title: "Buffer Overflow"
date: 2024-01-01T16:28:51-03:00
categories: ["Fundamentos"]
tags: ["Buffer Overflow", "Linux", "Windows"]
---

En este post, estaremos explicando algunos conceptos claves que podemos encontrarnos cuando tratamos con un Buffer Overflow.

## ¿Qué es un Buffer?

Un buffer, también conocido como búfer, es un área temporal de memoria física utilizada para almacenar información mientras se transfiere de un lugar a otro. Está diseñado para optimizar el proceso de transferencia de datos al proporcionar una respuesta rápida. Estos búferes suelen residir en la memoria RAM del sistema. Un aspecto crucial de los búferes es su capacidad limitada para contener una cantidad específica de datos.

Cuando un programa utiliza un búfer, es esencial incorporar instrucciones que gestionen adecuadamente la cantidad de datos almacenados. En ausencia de tales instrucciones, existe el riesgo de que los datos se sobrescriban en la memoria adyacente al búfer, lo que podría llevar a problemas de seguridad y errores en la ejecución del programa. La gestión cuidadosa de los búferes es fundamental para garantizar un flujo de datos eficiente y seguro en sistemas informáticos.

## ¿Qué es un registro?

Un registro puede entenderse de manera sencilla como una entidad similar a una variable que reside en la memoria de la CPU. Es una región de almacenamiento de datos con la capacidad de almacenar y recuperar información de manera eficiente. A diferencia de las variables que definimos en un programa, los registros están intrínsecamente vinculados a la arquitectura de la CPU y cumplen funciones específicas durante la ejecución del código.

La singularidad de los registros radica en su propósito especializado y su limitación en cantidad. Estos registros desempeñan roles cruciales en el procesamiento de instrucciones y la gestión de datos en la CPU. Dependiendo de la arquitectura de la CPU, los registros pueden tener tamaños distintos, siendo comunes las variantes de 32 bits o 64 bits. La elección del tamaño del registro influye en la capacidad de la CPU para manejar y procesar datos de manera eficiente. En resumen, los registros son componentes esenciales para el rendimiento y la ejecución eficaz de las instrucciones en una arquitectura de computadora.

Según la arquitectura hay nomenclaturas diferentes para cada registro:

- Registros de 64 bits: RAX, RBC, RCX, RDX, RSI, RDI, RBP, RSP
- Registros de 16 bits: AX, BX, CX, DX, SI, DI, BP, SP, IP
- Registros de 8bits: AH, AL, BH, BL, CH, CL, DH, DL

En este caso nos centraremos en los registros de 32 bits.

| Nomenclatura | Nombre Completo               | Uso                                                           |
| ------------ | ----------------------------- | ------------------------------------------------------------- |
| EIP          | Extended Instruction Pointer  | Es la siguiente dirección de memoria que se debería ejecutar. |
| EAX          | Extended Accumulator Register | Almacena de forma temporal cualquier dirección de retorno.    |
| ESI          | Extended Source Index         | Contiene la dirección de memoria de los datos de entrada.     |
| EBX          | Extended Base Register        | Almacena datos y direcciones de memoria.                      |
| ESP          | Extended Stack Pointer        | Almacena un puntero a la parte superior de la pila.           |
| EBP          | Extended Base Pointer         | Indica la dirección de memoria del final de un hilo.          |

## Little Endian - Big Endian

En la arquitectura x86 los datos se almacenan en el formato **little endian**, es decir, el byte menos significativo se almacena en una posición de memoria menor, y así hasta el byte más significativo.

Por lo tanto, en *little endian* el dato `0x12345678` se almacena en memoria:

```
  0x78 | 0x56 | 0x34 | 0x12
  n    | n+1  | n+2  | n+3
```

El dato `ABCD`, es decir `\x41\x42\x43\x44` se almacena en memoria como `DCBA`:

```
  0x44 | 0x43 | 0x42 | 0x41
  n    | n+1  | n+2  | n+3
```

![Little Endian - Big Endian](/assets/img/bof/LittleEndian-BigEndian.png)
_Little Endian - Big Endian_

## ¿Qué es un Buffer Overflow?

Un Buffer overflow también conocido como desborde de memoria, se produce cuando un programa excede el uso de cantidad de memoria asignada, lo cual provoca que los datos de entrada ocupen zonas de memoria adyacentes, donde se podria ejecutar código.

La explotación de esta vulnerabilidad se centra en sobrescribir la dirección de retorno (EIP). De esta forma, se puede redirigir el flujo de ejecucción del programa haciendo que pueda ir a un código desarrollado por el atacante.

![BOF](/assets/img/bof/BOF.png)
_Buffer Overflow_

## Tipos de Buffer Overflow

- **Stack overflow**: El desbordamiento de pila es un tipo de error que se produce cuando un programa informático intenta usar más espacio de memoria en la pila del que está asignado. La pila de llamadas, denominada “segmento de pila” es un búfer de tamaño fijo que almacena variables y funciones locales y datos de dirección de retorno durante la ejecucción. La pila de llamadas se adhiere a un tipo de arquitectura de último en entrar, primero en salir (LIFO). Cada función tiene su marco de pila, esto se añade a la parte superior de la pila de llamadas. Este marco de pila permanece en la memoria hasta que la función termine de ejecutarse, liberando así memoria para otros marcos de pila. El tamaño de una pila se suele definir a la hora del inicio del programa, su tamaño depende de determinados factores, como la arquitectura del ordenador, el lenguaje de programación, cantidad de memoria utilizada. Si un programa demanda más memoria de la que hay disponible se produce un desbordamiento de pila, lo que puede provocar que el programa se bloqueé.

- **Heap overflow**: A diferencia de la pila de llamadas, existe lo que se denomina como segmento, es un espacio de memoria que se asigna dinámicamente y que se almacena en variables globales. Este segmento también es igual de vulnerable que el de la “pila de llamadas (Stack)". Con los heaps, los que desarrollan los programas son responsables de reasignar la memoria, si no lo hacen de la forma correcta puede producirse un desbordamiento de pila. Este desbordamiento también puede ocurrir cuando las variables almacenadas contienen más datos que la cantidad de memoria asignada.

## El contador de programa (registro `eip`).

El *instruction pointer register* apunta a la siguiente instrucción a ser ejecutada por el programa. Cada vez que una instrucción se procesa, el procesador actualiza automáticamente este registro para que apunte a la siguiente instrucción a ser ejecutada. Para ello, su valor se incrementa de acuerdo al tamaño de la instrucción (por ejemplo, la instrucción `add eax, 0x10` que se almacena en memoria como `83 c0 10`, ocupa 3 bytes).

Además de los registros para el almacenamiento de datos, en un programa se utilizan áreas de memoria que pueden ser accedidas con instrucciones de accesos a memoria del tipo `load/store` o con las operaciones de la pila `push/pop`.

## La pila

### PUSH & POP

Las operaciones `push/pop` manipulan un área de la memoria de un proceso denominada pila o stack, que responde a una estructura de datos *LIFO* (_last in, first out_: último en entrar, primero en salir) donde los elementos se almacenan con `push` y se desapilan con `pop`.

La pila se utiliza para almacenar: valores de registros de manera temporaria, variables locales, parámetros de funciones y direcciones de retorno.

Uno de los registros especiales vinculados a la pila es el puntero de pila.

El **ESP** apunta al tope de la pila, es decir, la dirección de memoria donde se almacenará el próximo valor en la pila o donde se recuperará el último valor almacenado.

#### PUSH

![Stack Push](/assets/img/bof/stack-push.png)
_Operación Push_

#### POP

![Stack Pop](/assets/img/bof/stack-pop.png)
_Operación Pop_

## Ret2Libc

Ret2Libc hace referencia a "Return to libc" o en español "retorno a libc". Es una técnia que se utiliza principalmente en ataques de desbordamiento de búfer (buffer overflow) en programas escritos en lenguaje C, donde un atacante intenta ejecutar código malicioso al manipular el flujo de ejecución del programa.
Este tipo de ataques, se produce comunmente cuando existe un Buffer Overflow en una aplicación y no podemos inyectar nuestro shellcode debido a que esta habilitado el DEP (Data Execution Prevention), representado en los binarios como `NX` (No-eXecute).

> La tecnología NX (No-eXecute) o DEP (Data Execution Prevention) evita que áreas de la memoria marcadas como "solo de ejecución" se utilicen para almacenar datos.

En otras palabras, cuando un programa se ve afectado por un desbordamiento de búfer, el atacante puede intentar sobrescribir la dirección de retorno de una función (EIP (Instruction Pointer) o RIP (en arquitecturas más recientes de 64 bits, como x86-64)) con una dirección que apunte a una función de la biblioteca estándar del lenguaje C (libc), como `system()` o `execve()`. Esto se hace para ejecutar comandos o cargar programas sin necesidad de inyectar código malicioso directamente.

## Libc

La libc (biblioteca estándar de C) es una biblioteca de funciones en lenguaje de programación C que proporciona implementaciones estandarizadas de operaciones comunes. Esta biblioteca es parte integral del entorno de ejecución de programas escritos en C y se utiliza para realizar diversas tareas, como entrada/salida (E/S), manipulación de cadenas, gestión de memoria, operaciones matemáticas y muchas otras funciones que son esenciales para la programación en C.

Algunas de las funciones comunes que se encuentran en la libc incluyen `printf()`, `scanf()`, `malloc()`, `free()`, `strcpy()`, `strcmp()`, `system()`, y más. Estas funciones proporcionan una interfaz estándar y portátil para realizar tareas específicas en diferentes sistemas operativos y plataformas.

## ASLR

ASLR (Address Space Layout Randomization) es una técnica de seguridad diseñada para dificultar la explotación de vulnerabilidades de seguridad, como los ataques de desbordamiento de búfer. La idea principal detrás de ASLR es aleatorizar la ubicación en la memoria de ciertas áreas críticas del espacio de direcciones de un programa, lo que hace que sea más difícil para un atacante predecir la ubicación exacta de componentes clave, como el código ejecutable, las pilas y los montones.

Para habilitarlo en nuestra máquina de 32 bits, simplemente debemos ejecutar el siguiente comando como usuario **root**.

```bash
echo 2 > /proc/sys/kernel/randomize_va_space
```
