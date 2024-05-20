---
title:  "Creando una bóveda para datos sensibles"
header:
  teaser:
categories: 
  - Blog
tags:
  - Tutorial
---

En esta publicación, vamos a crear una bóveda de archivos para guardar datos sensibles en Linux. Tener un volumen cifrado nos puede ser muy útil para proteger información personal, como copia de seguridad e incluso para el cumplimiento normativo.

Durante el proceso, compararemos un disco normal con otro cifrado. Primero creamos dos archivos de 200MB cada uno. Podemos hacerlo con `dd` y leyendo de /dev/zero, de esta forma no tendrán basura.

```shell
dd if=/dev/zero of=file1 bs=200M count=1
dd if=/dev/zero of=file2 bs=200M count=1
```

Ahora, con `fdisk` crearemos la tabla de particiones y una partición Linux.

```shell
fdisk file1
```

Tras ejecutar el comando, nos llevará en un menú de selección. Con `m` podemos ver todas las opciones. 

```shell
Orden (m para obtener ayuda): m

Ayuda:

  DOS (MBR)
   a   conmuta el indicador de iniciable
   b   modifica la etiqueta de disco BSD anidada
   c   conmuta el indicador de compatibilidad con DOS

  General
   d   borra una partición
   F   lista el espacio libre no particionado
   l   lista los tipos de particiones conocidos
   n   añade una nueva partición
   p   muestra la tabla de particiones
   t   cambia el tipo de una partición
   v   verifica la tabla de particiones
   i   imprime información sobre una partición

  Miscelánea
   m   muestra este menú
   u   cambia las unidades de visualización/entrada
   x   funciones adicionales (sólo para usuarios avanzados)

  Script
   I   carga la estructura del disco de un fichero de script sfdisk
   O   vuelca la estructura del disco a un fichero de script sfdisk

  Guardar y Salir
   w   escribe la tabla en el disco y sale
   q   sale sin guardar los cambios

  Crea una nueva etiqueta
   g   crea una nueva tabla de particiones GPT vacía
   G   crea una nueva tabla de particiones SGI (IRIX) vacía
   o   crea una nueva tabla de particiones DOS vacía
   s   crea una nueva tabla de particiones Sun vacía
```

Para crear una nueva tabla de particiones GPT deberemos pulsar `g`.

```shell
Orden (m para obtener ayuda): g
Se ha creado una nueva etiqueta de disco GPT (GUID: AA8B2A9C-7C67-5A47-9F86-1BA4A5EC36C3).
```

Si usamos la opción `l` podremos ver una lista con los tipos de particiones. Seleccionaremos uno con `n`, en este caso será el número 20 (Linux filesystem).

```shell
Orden (m para obtener ayuda): n
Número de partición (1-128, valor predeterminado 1): 20
Primer sector (2048-409566, valor predeterminado 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-409566, valor predeterminado 409566): 

Crea una nueva partición 20 de tipo 'Linux filesystem' y de tamaño 199 MiB.
```

Para guardar los cambios y salir de este menú pulsaremos `w`. 

***NOTA: Realizamos los pasos anteriores con los dos archivos que hemos creado (file1 y file2).***

Ahora vamos a formatear el primer archivo a EXT4 usando `mkfs.ext4`

```shell
mkfs.ext4 file1
```

Después lo montaremos con el comando `mount`.

```shell
sudo mount -t ext4 file1 miboveda/
```

Podemos comprobar si se ha montado correctamente con `df`.

```shell
df -hT
S.ficheros     Tipo     Tamaño Usados  Disp Uso% Montado en
/dev/loop28    ext4       172M    28K  158M   1% /home/tarde/miboveda
```

Una vez tenemos montado el archivo, accedemos a él y dentro guardamos una imagen. Después nos salimos del directorio que acabamos de montar y procedemos a formatear el segundo archivo.

El otro fichero lo cifraremos con LUKS, el sistema de cifrado de disco en Linux. Podemos hacer esto con `cryptsetup`.

```shell
sudo cryptsetup luksFormat file2
WARNING: Device file2 already contains a 'ext4' superblock signature.

WARNING!
========
Sobrescribirá los datos en file2 de forma irrevocable.

Are you sure? (Type 'yes' in capital letters): YES
Introduzca una contraseña para file2: 
Verificar frase de paso:
```

***NOTA: Importante no perder la contraseña con la que ciframos el archivo.***

Para descrifar este archivo volvemos a usar `cryptsetup`.

```shell
sudo cryptsetup luksOpen file2 bovedacifrada
```

Cuando descifremos el archivo se guardará en /dev/mapper como un volumen lógico. Mientras lo tengamos abierto con `luksOpen`, LUKS cifrara todos los cambios que realicemos en él y lo descifrara cuando lo leamos.

```shell
ls /dev/mapper/
bovedacifrada  control
```

Antes de montarlo debemos formatearlo a EXT4.

```shell
mkfs.ext4 /dev/mapper/bovedacifrada
```

Igual que como hicimos con el primer fichero, debemos crear un directorio donde montarlo. Cuando lo tengamos solo nos queda por ejecutar `mount`.

```shell
sudo mount -t ext4 /dev/mapper/bovedacifrada mibovedacifrada/
``` 

Podemos comprobar si se ha hecho correctamente con `df`.

```shell
S.ficheros                Tipo     Tamaño Usados  Disp Uso% Montado en
/dev/loop28               ext4       172M    28K  158M   1% /home/tarde/miboveda
/dev/mapper/bovedacifrada ext4       157M    24K  144M   1% /home/tarde/mibovedacifrada
```

Ahora, nos metemos al directorio y volvemos a guardar una imagen dentro.

Estas dos imágenes nos servirán para comprobar como no vamos a poder recuperar datos de un archivo cifrado. 

## Recuperar los datos con Photorec 

PhotoRec es un programa de recuperación de archivos. Nosotros lo utilizaremos para intentar rescatar las imágenes de los volúmenes que hemos montado. Si no lo tenemos instalado en el sistema, podemos instalarlo con `sudo apt install testdisk`.

El primer paso será crear un directorio donde almacenar todos los datos que extraiga la herramienta. Antes de usar Photorec, vamos a comprobar que ruta tienen los volúmenes con `df`.

```shell
df -hT
S.ficheros                Tipo     Tamaño Usados  Disp Uso% Montado en
/dev/loop28               ext4       172M    52K  158M   1% /home/tarde/miboveda
/dev/mapper/bovedacifrada ext4       157M    92K  144M   1% /home/tarde/mibovedacifrada
```

En mi caso, el volumen no cifrado es "/dev/loop28" y el volumen del fichero cifrado es "/dev/mapper/bovedacifrada" (No confundir con el fichero real que está abierto con luksOpen).  

Ahora que conocemos las rutas, abrimos photorec.

```shell
photorec
```

![PhotoRec 1](/assets/images/2024-05-19-boveda_datos_sensibles/img_photorec_1.png)

Observamos cuáles son los volúmenes que hemos montado. El /dev/loop35 es el fichero cifrado que está abierto con luksOpen, podemos saberlo por su tamaño de 200MB.

Para recuperar los archivos debemos seleccionar un volumen. Escoger la partición "Unknown".

![PhotoRec 2](/assets/images/2024-05-19-boveda_datos_sensibles/img_photorec_5.png)

Marcamos la opción "ext2/ext3". 

![PhotoRec 3](/assets/images/2024-05-19-boveda_datos_sensibles/img_photorec_6.png)

Seleccionamos donde vamos a guardar los datos que extraiga la herramienta y pulsamos `C`.

![PhotoRec 4](/assets/images/2024-05-19-boveda_datos_sensibles/img_photorec_7.png)

Con esto guardaremos los archivos recuperados en el directorio que hemos seleccionado.

***Volumen cifrado***

![PhotoRec 5](/assets/images/2024-05-19-boveda_datos_sensibles/img_photorec_2.png)

***Volumen sin cifrar***

![PhotoRec 6](/assets/images/2024-05-19-boveda_datos_sensibles/img_photorec_3.png)

***Fichero cifrado abierto con luksOpen***

![PhotoRec 7](/assets/images/2024-05-19-boveda_datos_sensibles/img_photorec_4.png)

Como podemos ver, PhotoRec no ha podido recuperar información de un fichero cifrado, por lo que hemos comprobado que es útil para almacenar datos sensibles.

Cuando terminemos de trabajador en nuestra bóveda, podemos cerrar el archivo cifrado con `cryptsetup`.

```shell
sudo cryptsetup luksClose file2 bovedacifrada
```

<!--
[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/ 
-->
