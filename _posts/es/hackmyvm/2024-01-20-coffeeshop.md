---
title: 'CoffeeShop'
date: 2024-01-20T16:30:45-03:00
categories: ["Writeups", "HackMyVm"]
permalink: /posts/hackmyvm/:title/
tags: ['Linux', 'CTF', 'sudo', 'cron', 'crontab', 'ruby']
---

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop.png)
_CoffeeShop_

|Nombre|VM Info|
|---|---|
|**Sistema operativo**|Linux|
|**Dificultad**|Fácil|
|**Fecha de lanzamiento**|2024-01-15|
|**Máquina**|[CoffeShop](https://hackmyvm.eu/machines/machine.php?vm=CoffeShop)|
|**Creador**|[MrMidnight](https://www.youtube.com/@mrmidnight7331)|
|**Plataforma**|[HackMyVM](https://hackmyvm.eu/)|

# Introducción

En este artículo, llevaremos a cabo la resolución de la máquina [CoffeeShop](https://hackmyvm.eu/machines/machine.php?vm=CoffeeShop) de la plataforma HackMyVM.
La máquina presenta un sitio web en el puerto 80 con un formulario de inicio de sesión. Sin embargo, carecemos de credenciales y no podemos registrarnos de manera convencional. Realizando una enumeración con web fuzzing, descubrimos un subdominio que revela credenciales, las cuales aprovechamos para iniciar sesión en el sitio web. Una vez dentro, obtenemos nuevas credenciales que nos permiten acceder al servidor a través de SSH. Posteriormente, debemos realizar un movimiento lateral hacia otro usuario, aprovechando una tarea programada mediante cron. Finalmente, explotamos una mala configuración de `sudo` para elevar nuestros privilegios y completar la escalada de privilegios con éxito.

# Reconocimiento

Comenzamos lanzando una traza ICMP a la máquina objetivo para comprobar que tengamos conectividad.

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-1.png)
_Reconocimento usando el comando ping_

Vemos que responde al envio de nuestro paquete, verificando de esta manera que tenemos conectividad. Por otra parte, confirmamos que estamos frente a una máquina Linux basandonos en el TTL (Time To Live).

## Enumeración

### Descubrimiento de puertos abiertos
Realizamos un escaneo con nmap para descubrir que puertos TCP se encuentran abiertos en la máquina víctima.

```bash
nmap -sS -p- --open --min-rate 5000 -Pn -n 10.10.10.12 -oG scanPorts -vvv
```

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-2.png)
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
nmap -sCV -p22,80 10.10.10.12 -oN targeted -vvv
```

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-3.png)
_Descubrimiento de versión y servicio con nmap_

Parámetros utilizados:

- `-sCV` Es la combinación de los parámetros `-sC` y `-sV`. El primero determina que se utilizarán una serie de scripts básiscos de enumeración propios de nmap, para conocer el servicio que esta corriendo en dichos puertos. Por su parte, el segundo parámetro permite conocer más acerca de la versión de ese servicio.
- `-p-`: Indica que debe escanear todos los puertos (es igual a `-p 1-65535`)
- `-oN`: Determina el formato del archivo en el cual se guardan los resultados obtenidos, junto con el nombre del archivo `targeted`. En este caso, utiliza el formato por defecto de nmap.
- `-vvv`: Activa el modo _verbose_ para que nos muestre resultados a medida que los encuentra.
`

## Explotación

### Usuario `developer`

Si ingresamos al la dirección `10.10.10.12` desde nuestro navegador, nos encontramos con el siguiente sitio web:

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-4.png)
_Web CoffeeShop_

Vemos que nos indica que el sitio web se encuentra en construcción, haciendo mención también a un dominio **midnight.coffee**.

Agregemos este a nuestro archivo `/etc/hosts`.

```bash
echo '10.10.10.12 midnight.coffee' >> /etc/hosts
```

Hagamos un poco de fuzzing para descubrir alguna ruta o archivo interesante.

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-5.png)
_Fuzzing Web CoffeeShop_

Encontramos una ruta llamada `shop`.

Si accedemos a la misma, nos encontramos con la siguiente sección del sitio.

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-6.png){: width="500" height="250" }
_Web CoffeeShop_

Vemos que presenta una sección login, en la cual debemos ingresar un `Username` y `Password`.

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-7.png){: width="500" height="250" }
_Web CoffeeShop - Login_

Al intentar acceder utilizando credenciales comunes como `admin:admin`, `user:user`, o `admin:password123`, observamos que no logramos obtener acceso. Del mismo modo, al explorar la posibilidad de realizar una inyección SQL, parece que el sistema no presenta vulnerabilidades aparentes.

Sigamos realizando fuzzing, pero ahora en lugar de buscar archivos o rutas, busquemos si existe algún subdominio.

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-8.png)
_Fuzzing - We CoffeeShop_

Encontramos el subdominio `dev.midnight.coffee`.

Agregemos este a nuestro archivo `/etc/hosts`.

Si accedemos desde nuestro navegador podemos observar el siguiente sitio web.

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-9.png)
_Web Dev CoffeeShop_

Encontramos unas credenciales pertenecientes al usuario `developer`.

### Usuario `tuna`

Probemos iniciar sesión en el sistema principal.

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-10.png)
_Web Dev CoffeeShop_

Encontramos otras credenciales, en este caso, del usuario `tuna`.

Si recordamos nuestra enumeración de puertos, el puerto `22 (SSH)` estaba abierto.

Usemos estas credenciales para iniciar sesión a través de esta utilidad.

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-11.png){: width="500" height="250" }
_tuna login Server CoffeeShop_

Genail! logramos entrar en el servidor con el usuario `tuna`.

### Usuario `shopadmin`

Después de llevar a cabo una enumeración básica en el sistema, identificamos una tarea cron en ejecución asociada al usuario `shopadmin`. Esta tarea está programada para ejecutar un script ubicado en el directorio del usuario `shopadmin`, el cual, a su vez, ejecuta todos los archivos con extensión `.sh` encontrados bajo el directorio `/tmp`.

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-12.png){: width="700" height="400" }
_Archivo crontab_

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-13.png)
_execute.sh_

Probemos crear un pequeño script en el directorio `/tmp`, el cual nos otorge una shell como el usuario `shopadmin`.

```bash
#!/bin/bash
cp /bin/bash /tmp # copia el binario bash a el directorio tmp
chmod u+s /tmp/bash # asigna privilegios SUID
```

Si ejecutamos el binario, vemos que logramos obtener una shell como el usuario `shopadmin`.

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-14.png)
_execute.sh_

De esta forma ya podemos leer la flag del `user.txt`.

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-15.png)
_user.txt_

### Escalación de privilegios

Durante la enumeración básica, observamos la capacidad de ejecutar un script en Ruby utilizando el comando `sudo`.

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-16.png)
_sudo /usr/bin/ruby_

Haciendo una búsqueda en [GTFOBins](https://gtfobins.github.io/gtfobins/ruby/#sudo), vemos que existe una forma de escalar nuestros privilegios en este contexto. La cual consiste en pasar el argumento `-e 'exec "/bin/sh"'` al momento de ejecutar el binario.

Si probamos ejecutar el script, vemos que logramos elevar nuestros privilegios.

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-17.png)
_Escalación de privilegios_

De esta manera, ya podemos leer el flag de `root.txt`.

![CoffeeShop](/assets/img/hmvm/coffeeshop/coffeeshop-18.png)
_root.txt_

Con esto, concluimos la resolución de la máquina CoffeeShop.

Espero que los conceptos hayan quedado claros. Si has llegado hasta este punto y aún tienes dudas, te recomiendo volver a realizar la máquina.

Gracias por tu lectura!

Happy Hacking!
