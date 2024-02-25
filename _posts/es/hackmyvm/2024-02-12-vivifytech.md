---
title: 'VivifyTech'
date: 2024-02-12T18:36:32-03:00
permalink: /posts/hackmyvm/:title/
categories: ["Writeups", "HackMyVm"]
tags: ['Linux', 'Information disclosure', 'CMS', 'Wordpress', 'sudo', 'git']
---

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech.png)

|Nombre|VM Info|
|---|---|
|**Sistema operativo**|Linux|
|**Dificultad**|Fácil|
|**Fecha de lanzamiento**|2023-12-28|
|**Máquina**|[VivifyTech](https://hackmyvm.eu/machines/machine.php?vm=VivifyTech)|
|**Creador**|[Sancelisso](https://sancelisso.github.io/)|
|**Plataforma**|[HackMyVM](https://hackmyvm.eu/)|

# Introducción

¡Hola, hacker! ¡Bienvenido a un nuevo post!

En este artículo, nos adentraremos en la resolución de la máquina [VivifyTech](https://hackmyvm.eu/machines/machine.php?vm=VivifyTech) de la plataforma HackMyVM. Comenzaremos aprovechando una filtración de contraseñas y una lista de usuarios para llevar a cabo un ataque de fuerza bruta al protocolo SSH y obtener acceso a la máquina como el usuario `sarah`. A continuación, realizamos un movimiento lateral para obtener acceso como el usuario `gbodja` al encontrar su contraseña en un archivo de texto plano. Finalmente, aprovechamos una configuración inadecuada de `sudo` para elevar nuestros privilegios, logrando así una exitosa escalada de privilegios.

# Reconocimiento

Comenzamos lanzando una traza ICMP a la máquina objetivo para comprobar que tengamos conectividad.

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-1.png)
_Reconocimento usando el comando ping_

Podemos observar, que responde al envio de nuestro paquete, verificando de esta manera que tenemos conectividad. Por otra parte, confirmamos que estamos frente a una máquina Linux basandonos en el TTL (Time To Live).

## Enumeración

### Descubrimiento de puertos abiertos
Realizamos un escaneo con nmap para descubrir que puertos TCP se encuentran abiertos en la máquina víctima.

```bash
nmap -sS -p- --open --min-rate 5000 -Pn -n 192.168.1.14 -vvv -oG scanPorts
```

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-2.png)
_Descubrimiento de puertos abiertos_

> ver la [Cheatsheet](/cheatsheet) para más detalle sobre los parámetros utilizados.
{: .prompt-info }

### Enumeración de versión y servicio

Lanzamos una serie de script básicos de enumeración propios de nmap, para conocer la versión y servicio que esta corriendo bajo los puertos abiertos.

```bash
nmap -sCV -p22,80 -vvv -oN targeted 192.168.1.14
```

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-2.png)
_Descubrimiento de versión y servicio con nmap_

> ver la [Cheatsheet](/cheatsheet) para más detalle sobre los parámetros utilizados.
{: .prompt-info }

## Explotación

### Usuario `sarah`

Accedemos al sitio web que se encuentra en ejecución en el puerto 80, donde nos encontramos con la tipica web por defecto de una instanacia de Apache2:

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-5.png){: width="700" height="400" }
_Web Apache2_

Al realizar un poco de web fuzzing, nos encontramos con una instancia de Wordpress.

```bash
gobuster dir -u http://192.168.1.14 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 50
```

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-4.png)
_Gobuster_

Al ingresar a esta ruta, podemos observar la siguiente web:

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-6.png){: width="700" height="400" }
_Wordpress_

Realizamos una enumeración de Wordpress en busca de plugins o temas vulnerables, en este caso, usando el script `http-wordpress-enum` de nmap.

```bash
nmap -p80 --script http-wordpress-enum --script-args http-wordpress-enum.root='/wordpress',search-limit=1000 192.168.1.14
```

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-19.png)
_Enumeración de Wordpress_

Otras herramientas como `wpscan`, `nuclei` o la realización de fuzzing con `ffuf`, `gobuster`, entre otras, podrían ser utilizadas. Sin embargo, en este contexto, el resultado sería el mismo y no encontraríamos ninguna vulnerabilidad.

Si continuamos haciendo fuzzing, en este caso, bajo el directorio `wordpress` encontramos las carpetas tipicas de una instalación de este CMS.

Al intentar acceder desde nuestro navegador a `wordpress/wp-content`, descubrimos que no tenemos los permisos necesarios para listar el contenido del directorio. Sin embargo, al ingresar a `wordpress/wp-includes`, pudimos listar el contenido del directorio y encontramos un archivo llamado `secrets.txt`, el cual parece contener posibles contraseñas.

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-9.png)
_wp-includes - secrets.txt_

Nos descargamos el archivo a nuestra máquina atacante.

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-10.png)
_secrets.txt_

Podríamos optar por realizar un ataque de fuerza bruta utilizando `hydra`, sin embargo, esto implicaría contar únicamente con las contraseñas. Aunque factible, este enfoque consumiría considerablemente más tiempo de ejecución en comparación con el uso de una lista más específica de usuarios.

Recorriendo el sitio web, podemos encontrar el siguiente artículo en el cual se mencionan varios nombres (posibles usuarios).

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-11.png){: width="500" height="250" }
_Web VivifyTech_

Armamos una lista con todos los nombres mencionados.

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-12.png)
_users.txt_

Realizamos un ataca de fuerza bruta con `hydra` al puerto 22 (SSH) utilizando nuestra lista de usuarios y las posibles contraseñas del fichero `secrets.txt`.

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-13.png)
_hydra_

Vemos que tenemos éxito y encontramos la contraseña del usuario `sarah`.

De esta forma, ya podemos conectarnos a la máquina víctima.

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-14.png)
_sarah_

Leemos el flag de `user.txt`.

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-15.png)
_user.txt_

### Usuario `gbodja`

Si miramos dentro del directorio `/home/sarah/.private`, vemos que existe un archivo llamado `Tasks.txt` en el cual se revelan en texto plano las credenciales del usuario `gbodja`.

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-16.png)
_Tasks.txt_

Usamos estas credenciales para loguearnos como este.

## Escalación de privilegios

Haciendo una enumeración básica del sistema, podemos ver que el usuario `gbodja` puede ejecutar con `sudo` el binario de `git` sin solicitarse contraseña. 

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-17.png)
_sudo -l_

Efectuando una búsqueda rápida en [GTFObins - git](https://gtfobins.github.io/gtfobins/git/#sudo), podemos ver que existen varias formas de escalar nuestros privilegios en este contexto.

En este caso, utilizaré la siguiente:

```bash
sudo git -p help config
!/bin/sh
```

De esta manera, ya podemos leer el flag del `root.txt`.

![VivifyTech](/assets/img/hmvm/vivifytech/vivifytech-18.png)
_root.txt_

Con esto, concluimos el Writeup de la máquina VivifyTech.

Espero que los conceptos hayan quedado claros. En caso de tener alguna duda, te invito a realizar nuevamente la máquina para afianzar tus conocimientos.

¡Gracias por leer!

Happy Hacking!
