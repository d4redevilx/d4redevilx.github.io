---
title: "Cheatsheet"
icon: fas fa-info-circle
order: 4
---

Esta cheatsheet agrupa una variedad de comandos y scripts útiles para realizar Pentesting. Diseñada como una herramienta dinámica, esta recopilación se mantendrá en constante evolución, actualizándose con nuevos conocimientos y conceptos que adquiera con el tiempo.

# 1 Reconocimiento

## 1.1 whois
```bash
whois <URL>
```

## 1.2 host
```bash
host <URL>
```

## 1.3 dnsrecon
```bash
dnsrecon -d <URL>
```

## 1.4 wafw00f
```bash
wafw00f <URL>
```

## 1.5 sublist3r
```bash
sublist3r -d <URL> -e <engines>
```

## 1.6 theHarvester
```bash
theHarvester -d <URL> -b <engines>
```

# 2 Descubrimiento de Hosts / Enumeración

## 2.1 fping

```bash
fping -a -g <IP-RANGE> 2>/dev/null
```

- `-a` solo muestra hosts activos
- `-g` envía una traza icmp a un rango de direcciones IP

Ejemplo:

```bash
fping -a -g 10.10.10.0/24 2>/dev/null > hosts.txt
```

Combinación de `fping` con `nmap`

```bash
fping -a -g 10.10.10.0/24 2>/dev/null
nmap -sn -iL hosts.txt
```

## 2.2 Nmap

### 2.2.1 Descubrimiento de host - Ping Scan

```bash
sudo nmap -sn 10.10.10.0/24
```

- `-sn` Esta opción le dice a Nmap que no haga un escaneo de puertos después del descubrimiento de hosts y que sólo imprima los hosts disponibles que respondieron a la traza icmp.
### 2.2.2 Escaneo de puertos

```bash
sudo nmap -p- --open -Pn -n 10.10.10.5 -vvv -oG scanPorts
```

Parámetros utilizados:

- `-sS`: Realiza un TCP SYN Scan para escanear de manera sigilosa, es decir, que no completa las conexiones TCP con los puertos de la máquina víctima.
- `-p-`: Indica que debe escanear todos los puertos (es igual a `-p 1-65535`).
- `--min-rate 5000`: Establece el número mínimo de paquetes que nmap enviará por segundo.
- `-Pn`: Desactiva el descubrimiento de host por medio de ping.
- `-vvv`: Activa el modo _verbose_ para que nos muestre resultados a medida que los encuentra.
- `-oG`: Determina el formato del archivo en el cual se guardan los resultados obtenidos. En este caso, es un formato _grepeable_, el cual almacena todo en una sola línea. De esta forma, es más sencillo procesar y obtener los puertos abiertos por medio de expresiones regulares, en conjunto con otras utilidades como pueden ser grep, awk, sed, entre otras.
### 2.2.3 Versión y Servicio

```bash
sudo nmap -sCV -p<PORTS> 10.10.10.5 -oN targeted -vvv 
```

- `-sCV` Es la combinación de los parámetros `-sC`  y `-sV`. El primero determina que se utilizarán una serie de scripts básiscos de enumeración propios de nmap, para conocer el servicio que esta corriendo en dichos puertos. Por su parte, segundo parámetro permite conocer más acerca de la versión de ese servicio.
- `-p-`: Indica que debe escanear todos los puertos (es igual a `-p 1-65535`).
- `-oN`: Determina el formato del archivo en el cual se guardan los resultados obtenidos. En este caso, es el formato por defecto de nmap.
- `-vvv`: Activa el modo _verbose_ para que nos muestre resultados a medida que los encuentra.

### 2.2.4 UDP (top 100)

```bash
sudo nmap -n -v -sU -F -T4 --reason --open -T4 -oA nmap/udp-fast 10.10.10.5
```

### 2.2.5 UDP (top 20)
```bash
sudo nmap -n -v -sU -T4 --top-ports=20 --reason --open -oA nmap/udp-top20 10.10.10.5
```

### 2.2.5 Obtener ayuda sobre scripts

```bash
nmap --script-help="http-*"
```

## 2.3 Enrutamiento

### 2.3.1 Mostrar la tabla de enrutamiento

En Windows y Linux podemos usar:

```bash
arp -a
```

En Linux, podemos usar:

```bash
ip route
```
#### 2.3.2 Configurar una ruta con `ip route`

```bash
ip route add <Network To Access> via <Gateway Address>
```

Ejemplo:

```bash
ip route add 10.10.10.0/24 via 10.10.10.1
```

Esto añade una ruta a la red 10.10.10.0/24 a través del router 10.10.10.1.

# 3 Servicios Comunes

En esta sección, se presentan técnicas de enumeración, explotación e interacción para servicios comunes que podrían ser descubiertos a través del escaneo.

## 3.1 TCP

|**Puerto**|**Servicio**|
|---|---|
|21|FTP|
|22|SSH|
|23|Telnet|
|25|SMTP|
|53|DNS|
|80|HTTP|
|110|POP3|
|139|SMB - NetBios|
|445|SMB|
|143|IMAP|
|443|HTTPS|


## 3.2 UDP

|**Puerto**|**Servicio**|
|---|---|
|53|DNS|
|67|DHCP|
|68|DHCP|
|69|TFTP|
|161|SNMP|

## 3.3 FTP (21)

El Protocolo de transferencia de archivos es un protocolo de red para la transferencia de ficheros entre sistemas conectados a una red TCP, basado en la arquitectura cliente-servidor.

### 3.3.1 Nmap

Cuando lanzamos una enumeración usando Nmap, se utilizan por defecto una serie de scripts que comprueban si se permite el acceso de forma anonima.

- `anonymous:anonymous`
- `anonymous`
- `ftp:ftp`

### 3.3.2 Conexión al servidor FTP

```bash
# -A: Esta opción es específica del cliente FTP y suele utilizarse para activar 
# el modo ASCII  de transferencia de archivos. En este modo, los archivos se 
# transfieren en formato de texto, lo que significa que se pueden realizar 
# conversiones de formato (por ejemplo, de CRLF a LF en sistemas Unix).
ftp -A 192.168.1.10

nc -nvc 192.168.1.10 21

telnet 192.168.1.10 21
```

### 3.3.3 Interactuar con el cliente FTP

```bash
ftp> anonymous # usuario
ftp> anonymous # contraseña
ftp> help # mostrar la ayuda
ftp> help CMD # mostrar la ayuda de un comando especifico
ftp> binary # establecer la transmisión en binario en lugar de ascii
ftp> ascii # establecer la transmisión a ascii en lugar de binario
ftp> ls -a # lista todos los archivos incluyendo los ocultos
ftp> cd DIR # cambia el directorio remoto
ftp> lcd DIR # cambia el directorio local
ftp> pwd # mostrar el directorio actual de trabajo
ftp> cdup  # mover al directorio anterior de trabajo
ftp> mkdir DIR # crea un directorio
ftp> get FILE [NEWNAME] # descarga un archivo con el nombre indicado NEWNAME
ftp> mget FILE1 FILE2 ... # descarga multiples archivos
ftp> put FILE [NEWNAME] # sube un fichero local a el servidor ftp con el nuevo nombre indicado NEWNANE
ftp> mput FILE1 FILE2 ... # sube multiples archivos
ftp> rename OLD NEW # renombra un archivo remoto
ftp> delete FILE # borra un fichero
ftp> mdelete FILE1 FILE2 ... # borra multiples archivos
ftp> mdelete *.txt # borra multiples archivos que cumplan con el patrón
ftp> exit # abandona la conexión ftp
```

### 3.3.4 Fuerza bruta de credenciales

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://192.168.1.10
```

### 3.3.5 Arhivos de configuración

- `/etc/ftpusers`
- `/etc/vsftpd.conf`
- `/etc/ftp.conf`
- `/etc/proftpd.conf`

### 3.3.6 Descargar archivos

```bash
wget -m ftp://anonymous:anonymous@192.168.1.10
```

## 3.4 SMB (445)

SMB (Server Message Block) es un protocolo diseñado para la compartición de archivos en red, facilitando la interconexión de archivos y periféricos como impresoras y puertos serie entre ordenadores dentro de una red local (LAN).
- SMB utiliza el puerto 445 (TCP). Sin embargo, originalmente, SMB se ejecutaba sobre NetBIOS utilizando puerto 139.
- SAMBA es la implementación Linux de código abierto de SMB, y permite a los sistemas Windows acceder a recursos compartidos y dispositivos Linux acceder a recursos compartidos y dispositivos Linux.

El protocolo SMB utiliza dos niveles de autenticación, a saber:
- **Autenticación de usuario**: los usuarios deben proporcionar un nombre de usuario y una contraseña para autenticarse con el servidor SMB para acceder a un recurso compartido.
- **Autenticación de recurso compartido**: los usuarios deben proporcionar una contraseña para acceder a un recurso compartido restringido.

### 3.4.1 Nmap

Scripts de `nmap` utiles para este servicio:

- smb-ls
- smb-protocols
- smb-security-mode
- smb-enum-sessions
- smb-enum-shares
- smb-enum-users
- smb-enum-groups
- smb-enum-domains
- smb-enum-services

Sintaxis: 

```bash
nmap -p445 --script <script> 192.168.1.10
```

### 3.4.2 smbclient

Es un cliente que nos permite acceder a recursos compartidos en servidores SMB.

```bash
# Realiza una conexión con el usuario elliot
smbclient //192.168.1.10/Public -U elliot

# Conexión utilizando una sesión nula
smbclient //192.168.1.10/Public -N

# Lista recursos compartidos
smbclient -L 192.168.1.10 -N
```

### 3.4.3 smbmap

SMBMap permite a los usuarios enumerar las unidades compartidas samba en todo un dominio. Enumera las unidades compartidas, los permisos de las unidades, el contenido compartido, la funcionalidad de carga/descarga, la coincidencia de patrones de descarga automática de nombres de archivo e incluso la ejecución de comandos remotos.

```bash
# Utiliza un usuario de invitado (guest) con una contraseña en blanco para 
# autenticarse en el objetivo especificado por 192.168.1.10.
# -d indica el dominio actual.
smbmap -u guest -p "" -d . -H 192.168.1.10

# Autentica con un usuario y contraseña específicos (<USER> y <PASSWORD>) 
# en el objetivo (192.168.1.10).
# -L lista los recursos compartidos disponibles en la máquina.
smbmap -u <USER> -p <PASSWORD> -H 192.168.1.10 -L

# Autentica en el objetivo y muestra la lista de archivos y 
# carpetas en la unidad C$ de forma recursiva.
smbmap -u <USER> -p <PASSWORD> -H 192.168.1.10 -r 'C$'

# Autentica y sube un archivo desde la ubicación local /root/file
# al recurso compartido C$ en el objetivo.
smbmap -H 192.168.1.10 -u <USER> -p <PASSWORD> --upload '/root/file' 'tmp/file'

# Autentica y descarga el archivo 'file' desde el recurso compartido tmp 
# en el objetivo.
smbmap -H 192.168.1.10 -u <USER> -p <PASSWORD> --download 'tmp/file'

# Autentica en el objetivo y ejecuta el comando ipconfig en el sistema remoto 
# usando SMB.
smbmap -u <USER> -p <PASSWORD> -H 192.168.1.10 -x 'ipconfig'
```

### 3.4.4 enum4linux

Enum4linux es una herramienta utilizada para extraer información de hosts de Windows y Samba. La herramienta está escrita en Perl y envuelta en herramientas de samba `smbclient`, `rpcclient`, `net` y `nslookup`.

```bash
# -o indica que se realizará una enumeración básica.
enum4linux -o 192.168.1.10

# -U indica que se realizará una enumeración de usuarios
enum4linux -U 192.168.1.10

# -G indica que se realizará una enumeración de grupos 
enum4linux -G 192.168.1.10

# -S indica que se realizará una enumeración de los recursos compartidos
enum4linux -S 192.168.1.10

# -i Comprueba si el servidor smb esta configurado para imprimir
enum4linux -i 192.168.1.10

# -r Intentará enumerar usuarios utilizando RID cycling en el sistema remoto.
# -u Especifica el nombre de usuario que se utilizará para la autenticación. 
# -p Especifica la contraseña asociada al usuario proporcionado con la opción -u.
enum4linux -r -u <user> -p <password> 192.168.1.10
```

#### 3.4.5 NetExec

```bash
# Enumerar hosts
nxc smb 192.168.1.10/24

# comprobar null sessions
nxc smb 192.168.1.10 -u '' -p ''

# comprobar guest login
nxc smb 192.168.1.10 -u 'guest' -p ''

# enumerar hosts con firma SMB no requerida
nxc smb 192.168.1.10/24 --gen-relay-list relay_list.txt

# enumerar recursos compartidos usando una null session
crackmapexec smb VICTIM_IPS -u '' -p '' --shares

# enumerar recursos compartidos de lectura/escritura en múltiples IPs con/sin credenciales
crackmapexec smb VICTIM_IPS -u USERNAME -p 'PASSWORD' --shares --filter-shares READ WRITE
```

### 3.4.6 Rpcclient

rpcclient es una utilidad que forma parte del conjunto de herramientas Samba. Se utiliza para interactuar con el protocolo Remote Procedure Call (RPC) de Microsoft, que se utiliza para la comunicación entre los sistemas basados en Windows y otros dispositivos. rpcclient se utiliza principalmente para fines de depuración y pruebas, y se puede utilizar para consultar y manipular sistemas remotos.

El protocolo SMB se utiliza principalmente para compartir archivos, impresoras y otros recursos en una red, pero también puede aprovechar RPC para ciertas funcionalidades y operaciones específicas.

Por ejemplo, cuando accedes a recursos compartidos en una red Windows, como carpetas compartidas o impresoras, estás utilizando el protocolo SMB. Sin embargo, para algunas operaciones administrativas y de gestión, como enumerar usuarios y grupos, modificar permisos de archivos o impresoras, o acceder a la configuración del sistema remoto, SMB puede utilizar RPC para realizar estas tareas.

Cuando utilizas herramientas como `rpcclient` para interactuar con un sistema remoto que ejecuta servicios SMB, estás esencialmente aprovechando el protocolo RPC subyacente que forma parte de la implementación de SMB en ese sistema. De esta manera, `rpcclient` puede actuar como una interfaz para realizar consultas y ejecutar comandos a través del protocolo RPC en el contexto de un servidor SMB.

```bash
# obtener información sobre el sistema remoto, como el nombre del servidor, 
# la versión del sistema operativo, el dominio de trabajo, la fecha y la hora del sistema, 
# entre otros detalles.
rpcclient -U "" -N 192.168.1.10 -c "srvinfo"

# enumera los usuarios del dominio
rpcclient -U "" -N 192.168.1.10 -c "enumdomusers"

# enumera los grupos del dominio
rpcclient -U "" -N 192.168.1.10 -c "enumdomgroups"

# obtiene el SID del usuario en base a su nombre
rpcclient -U "" -N 192.168.1.10 -c "lookupnames root"

# Se utiliza para enumerar los SID (Security Identifiers) asignados a los grupos 
# en un servidor remoto. Es útil para obtener información sobre los grupos de 
# seguridad disponibles en un sistema y sus respectivos SID.
rpcclient -U "" -N 192.168.1.10 -c "lsaenumsid"
```

El parámetro `-c` en `rpcclient` se utiliza para especificar un comando o una secuencia de comandos que se ejecutarán en el servidor remoto una vez que se haya establecido la conexión. Esto permite realizar operaciones específicas de forma automática sin necesidad de interactuar manualmente con `rpcclient` después de establecer la conexión.

La sintaxis básica del parámetro `-c` es la siguiente:

```bash
rpcclient -U username //192.168.1.10 -c "command1; command2; command3"
```

### 3.4.7 RID Cycling Attack

```bash
seq 1 5000 | xargs -P 50 -I{} rpcclient -U "" 30.30.30.4 -N -c "lookupsids S-1-22-1-{}" 2>&1
```

### 3.4.8 SMB desde Windows
```powershell
# listar recursos compartidos
net share

# borrar el recurso compartido
net use * \delete

# montar el recurso compartido
net use z: \\192.168.1.10\c$ <password> /user:<username>

# /all nos permite ver los recursos compartidos administrativos (que terminan en '$').
# Puede usar IP o nombre de host para especificar el host.
net view \\192.168.1.10 /all
```

Recursos compartidos comunes en Windows:

- `C$` corresponde a C:/
- `ADMIN$` se asigna a C:/Windows
- `IPC$` se utiliza para RPC
- `Print$` aloja controladores para impresoras compartidas
- `SYSVOL` sólo en DCs
- `NETLOGON` sólo en los DC

### 3.4.9 Interactuar con el cliente SMB

```bash
smb: \> help # muestra la ayuda
smb: \> ls # listar archivos
smb: \> put file.txt # subir un archivo
smb: \> get file.txt # descargar un archivo
```

### 3.4.10 Montar una carpeta compartida

```bash
mount -t cifs -o "username=user,password=password" //192.168.1.10/shared_folder /mnt/shared_folder
```

### 3.4.11 Fuerza bruta de credenciales

```bash
nmap --script smb-brute -p 445 192.168.1.10
hydra -l admin -P /usr/share/wordlist/rockyou.txt 192.168.1.10 smb
```

### 3.4.12 Metasploit

Modulos utiles:

- auxiliary/scanner/smb/smb2
- auxiliary/scanner/smb/smb_login
- auxiliary/scanner/smb/smb_enumusers
