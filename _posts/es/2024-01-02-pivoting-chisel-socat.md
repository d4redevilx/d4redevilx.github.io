---
title: Pivoting utilizando Chisel, Socat y Proxychains
date: 2024-01-02T22:30:35-03:00
categories: ["Fundamentos"]
tags: ["Pivoting", "Linux", "chisel", "socat", "proxychains"]
---

El proposito de este laboratorio, es aprender a realizar pivoting de manera manual utilizando las herramientas `chisel`, `socat` y `proxychains`.

El escenario que se nos presenta es el siguiente:

![Laboratirio de Pivoting](/assets/img/pivoting/Pivitong-Manual-Linux-1.png "Laboratorio de Pivoting")

Podemos observar que tenemos tres subredes la `10.10.10.0/24`, otra subred que es la `20.20.20.0/24` y por ultimo la `30.30.30.0/24`.

El objetivo, es poder acceder desde nuestra máquina atacante a la máquina Linux que tiene la ip `30.30.30.5`. En principio no tenemos acceso a esta máquina, ya que no compartimos la misma subred, para lo cual deberemos hacer pivoting a través de las otras máquinas.

> Con el fin de solamente centrarnos en lo que es el pivoting, dejaremos de lado la reconocimiento, enumeración, explotación y el resto de fases de un pentesting. Para lo cual, contamos con credenciales para acceder por ssh a cada una de las máquinas.

## Chisel

Antes de seguir avanzando en nuestro laboratorio, tenemos que conocer las herramientas a utilizar. Para ello, debemos conocer que es Chisel.

>Como se indica en el repositorio oficial de [Chisel](https://github.com/jpillora/chisel), Chisel es un túnel rápido TCP/UDP, transportado sobre HTTP, asegurado vía SSH. Un único ejecutable que incluye cliente y servidor. Escrito en Go (golang). Chisel es principalmente útil para pasar a través de firewalls, aunque también se puede utilizar para proporcionar un punto final seguro en su red.
{: .prompt-info }

Ahora si, iniciemos con el laboratorio.

En primer lugar, accedemos via ssh a la máquina 1 `10.10.10.4` y descargamos el binario de `chisel` y `socat`, el cual compartimos desde la nuestra máquina atacante creando un simple servidor http con python.

![Servidor HTTP con Python](/assets/img/pivoting/1.png "Servidr HTTP con Python")

Descargamos los binarios de `chisel` y `socat` desde la máquina 1 y le asignamos permisos de ejecución.

![Descarga del los binarios chisel y socat en la Máquina 1](/assets/img/pivoting/2.png "Descarga del los binarios chisel y socat en la Máquina 1")

## Servidor de `chisel`

Creamos el servidor de chisel desde nuetra máquina atacante.

![Creamos el servidor de chisel](/assets/img/pivoting/3.png "Servidor de chisel")

- `server` Corre chisel en modo servidor
- `--reverse` Este parámetro habilita la funcionalidad de redirección reversa de puertos en el servidor de Chisel. Esto implica que un cliente de Chisel puede exponer los puertos de la máquina en la que se está ejecutando hacia los puertos locales de la máquina donde se encuentra en funcionamiento el servidor de Chisel.
- `--port, -p` Define el puerto HTTP en el cual escuchará el servidor.

## Cliente de `chisel` - Máquina 1

Nos conectamos con el cliente de chisel al servidor creado anteriormente haciendo un portforwarding de todos los puertos

![Conexión del cliente Máquina 1](/assets/img/pivoting/4.png "Conexión del cliente Máquina 1")

![Primer tunel Proxy](/assets/img/pivoting/5.png "Primer tunel Proxy")

La máquina 2 `20.20.20.5` tiene un servidor HTTP Apache corriendo en el puerto 80. Pero todavía no tenemos acceso, ya que debemos realizar una configuración en nuestro navegador.

![Settings Firefox](/assets/img/pivoting/6.png "Settings Firefox")

![Network Settings Firefox](/assets/img/pivoting/7.png "Network Settings Firefox")

![Connection Settings Firefox](/assets/img/pivoting/8.png "Connection Settings Firefox")

Una vez realizada la configuración, ya podemos acceder al sitio web.

![Web Máquina 2](/assets/img/pivoting/9.png "Web Máquina 2")

## Proxychains

Por otra parte, si queremos realizar escaneos con nmap, conectarnos por ssh, utilizar herramientas de web fuzzing, entre otros, deberemos agregar la siguiente linea `socks5 127.0.0.1 4444` a nuestro archivo `/etc/proxychains4.conf` (en el caso de Parrot el archivo es `/etc/proxychains.conf`):

> Proxychains es una herramienta de Linux que actua como un "encadenador" o "cadena" entre otras herramientas o aplicaciones que deseamos ejecutar y una serie de servidores proxy que queremos utilizar. En este post, lo utilizaremos junto con chisel, pero también podemos usarlo con herramientas como Ligolo-NG o SSH.
{: .prompt-info }

![Archivo proxychains4.conf](/assets/img/pivoting/10.png "Archivo proxychains4.conf")

Es importante aclarar, que al igual que configuramos el proxy en nuestro navegador, debemos ejecutar nuestros comandos anteponiendo el comando `proxychains`.

Ejemplo:

```bash
proxychains nmap -sT -p- --open -Pn -n 20.20.20.5 -vvv -oG scanPorts
proxychains nmap -sT -sCV --open -p22,80 20.20.20.5 -n -Pn -vvv -oN targeted
```

> Es importante mencionar, que proxychains soporta unicamente conexiones TCP, por esta razón especificamos el argumente `-sT`, donde la técnica de escaneo implica que Nmap intentará establecer una conexión TCP con los puertos del objetivo para determinar su estado. Si la conexión se establece correctamente, se concluye que el puerto está abierto. Si la conexión es rechazada, se interpreta como un puerto cerrado. Además de lo mencionado, proxychains soporta unicamente los protocolos HTTP y SOCKS en su versión 4 y 5. Por ende, el uso del comando `ping` queda descartado ya que utiliza el protocolo ICMP.
{: .prompt-tip }

## Reverse Shell - Máquina 2

Hasta el momento, hemos podido acceder al sitio web alojado en la máquina `20.20.20.5`. Pero que pasa si queremos acceder a la máquina 2, es decir, obtener una reverse shell. No podremos, ya que lo que hacemos con `chisel` es hacer un portforwarding de los puertos de la máquina víctima, pero la máquina 2 no tiene conexión con nuestra máquina atacante. Es en este punto donde entra en juego `socat`.

![Pivitong Manual Linux](/assets/img/pivoting/Pivitong-Manual-Linux-2.png "Pivitong Manual Linux")

## Socat

Antes de continuar, debemos explicar ¿Qué es SOCAT?

`socat` es una herramienta de línea de comandos en sistemas operativos basados en Unix, incluidos Linux, que proporciona funcionalidades avanzadas para la manipulación y transferencia de datos entre dos puntos. El nombre "socat" proviene de "SOcket CAT", y puede considerarse como una versión avanzada y extendida de la utilidad `cat`.

`socat` es versátil y puede utilizarse para crear conexiones de red, realizar redirecciones, y manipular varios tipos de comunicación de red y sistemas de archivos. Puede funcionar como un "conector" entre diferentes dispositivos o procesos, y es conocido por ser extremadamente flexible.

Algunos ejemplos de uso de `socat` incluyen:

1. **Túneles y reenvíos de puertos:** Puede utilizar `socat` para crear túneles de red o redirigir puertos, permitiendo la comunicación entre dos sistemas a través de un firewall o una red.

```bash
# Ejemplo de reenvío de puerto
socat tcp-listen:1111,fork,reuseaddr tcp:10.10.10.5:4444
```

- `tcp-listen:1111,fork,reuseaddr`: Esta parte del comando indica a `socat` que debe escuchar en el puerto TCP 1111 en el sistema local. La opción `fork` significa que se debe crear un nuevo proceso para manejar cada conexión entrante, y `reuseaddr` permite que se vuelva a utilizar la dirección y el puerto incluso si la conexión anterior aún está en curso.
- `tcp:10.10.10.5:4444`: Esto especifica que las conexiones entrantes al puerto TCP 1111 deben ser redirigidas a la dirección IP `10.10.10.5` en el puerto TCP 4444. En otras palabras, este comando establece un reenvío de puerto desde el puerto 1111 en el sistema local hasta el puerto 4444 en el sistema remoto con la dirección IP 10.10.10.5.

2. **Manipulación de archivos y dispositivos:** Podemos utilizar `socat` para redirigir la salida de un comando a un archivo o manipular datos entre procesos.

```bash
# Ejemplo de redirección de salida
echo "Hola Mundo" | socat - FILE:/tmp/salida.txt
```

3. **Conexiones SSL/TLS:** `socat` también es capaz de manejar conexiones seguras utilizando SSL/TLS.

```bash
# Ejemplo de conexión SSL
socat TCP-LISTEN:443,reuseaddr,fork OPENSSL:localhost:80,verify=0
```

Continuando con nuestro laboratorio, lo que debemos hacer en este punto es acceder nuevamente a la máquina 1 por ssh y ejecutar socat de la siguiente manera:

![Socat Máquina 1](/assets/img/pivoting/12.png "Socat Máquina 1")

Nos ponemos en escucha con `nc` en el puerto 111 para recibir la reverse shell desde la máquina 2, la cual es reenviada por la conexión socat creada anteriormente.

![Reverse Shell Máquina 2](/assets/img/pivoting/13.png "Reverse Shell Máquina 2")

Accedemos por ssh a la máquina 2 y enviamos la reverse shell a la máquina 1, donde socat se encargara de realizar el envio.

![Acceso por ssh a la Máquina 2](/assets/img/pivoting/14.png "Acceso por ssh a la Máquina 2")

Enviamos la reverse shell

![Envio de la Reverse Shell Máquina 2](/assets/img/pivoting/15.png "Envio de la Reverse Shell Máquina 2")

Máquina atacante

![Reverse Shell Máquina Kali](/assets/img/pivoting/16.png "Reverse Shell Máquina Kali")

Hasta el momento tenemos lo siguiente:

![Pivitong Manual Linux](/assets/img/pivoting/Pivitong-Manual-Linux-3.png "Pivitong Manual Linux")

El proximo paso que debemos realizar, es compartir desde la máquina 1 los binario de `chisel` y `socat`, para lo cual creamos un servidor con python por el puerto `8000`.

![Servidor HTTP con Python Máquina 1](/assets/img/pivoting/17.png "Servidor HTTP con Python Máquina 1")

Máquina 2 - Descargamos los binarios de `chisel` y `socat` y le asignamos permisos de ejecución.

![Descargamos los binarios de chisel y socat](/assets/img/pivoting/18.png "Descargamos los binarios de chisel y socat")

Ahora lo que debemos hacer es crear una nueva conexión de socat en la **Máquina 1**

![Conexión con socat Máquina 1](/assets/img/pivoting/19.png "Conexión con socat Máquina 1")

Luego, en la Máquina 2, creamos otro tunel proxy con `chisel`.

![Tunel proxy con chisel Máquina 2](/assets/img/pivoting/20.png "Tunel proxy con chisel Máquina 2")

Y si miramos nuestro chisel server, deberíamos ver un nuevo tunel proxy creado en el puerto 5555

![Servidor Chisel Máquina atacante](/assets/img/pivoting/21.png "Servidor Chisel Máquina atacante")

En este punto, debemos agregar este nuevo proxy al archivo `proxychains4.conf`

![Archivo proxychains](/assets/img/pivoting/22.png "Archivo proxychains")

> Es importante aclarar, que los nuevos proxys deben ingresarse arriba de los que estan presentes.
{: .prompt-warning }

También debemos descomentar la opción `dynamic_chain` y comentar la opción `strict_chain`.

![Archivo proxychains](/assets/img/pivoting/22b.png "Archivo proxychains")

![Connection Firefox](/assets/img/pivoting/23.png "Connection Firefox")

![Host 3 Web](/assets/img/pivoting/24.png "Host 3 Web")

Hasta el momento, nuestro diagrama de red luce de la siguiente forma:

![Pivitong Manual Linux](/assets/img/pivoting/Pivitong-Manual-Linux-4.png "Pivitong Manual Linux")

## Reverse Shell - Máquina 3

Por ultimo, resta ganar acceso a través de una reverse shell a la máquina 3. Para lo cual debemos hacer lo siguiente:

Desde la máquina 1, debemos crear una nueva conexión con `socat`.

![Conexión con socat Máquina 1](/assets/img/pivoting/25.png "Conexión con socat Máquina 1")

Desde la máquina 2, también debemos crear un conexión con `socat` que redirija las conexiones a la máquina 1.

![Conexión con socat Máquina 2](/assets/img/pivoting/26.png "Conexión con socat Máquina 2")

Luego, desde nuestra máquina atacante ejecutamos `nc` para que escuche por el puerto 222 esperando recibir la reverse shell desde la máquina 3.

![Reverse Shell Máquina Kali](/assets/img/pivoting/27.png "Reverse Shell Máquina Kali")

Enviamos la reverse shell desde la máquina 3

![Reverse Shell Máquina 3](/assets/img/pivoting/28.png "Reverse Shell Máquina 3")

Recibimos la reverse shell en nuestra máquina atacante

![Reverse Shell Máquina Kali](/assets/img/pivoting/29.png "Reverse Shell Máquina Kali")

![Reverse Shell Máquina Kali](/assets/img/pivoting/30.png "Reverse Shell Máquina Kali")

El diagrama final de nuestra red quedaría de la siguiente manera:

![Pivitong Manual Linux](/assets/img/pivoting/Pivitong-Manual-Linux-5.png "Pivitong Manual Linux")

De esta forma, llegamos al final de nuestro laboratorio.

Espero que los conceptos y la metodología para realizar pivoting con estas herramientas hayan quedado claros. Si has llegado hasta este punto y aún tienes dudas, te recomiendo volver a realizar el laboratorio.

¡Gracias por tu lectura!
