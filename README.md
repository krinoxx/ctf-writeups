# CTF Writeups ⚔️

Writeups de las máquinas que voy resolviendo en distintas plataformas de hacking ético.

Cada writeup documenta el proceso completo: reconocimiento, enumeración, explotación, escalada de privilegios y lecciones aprendidas.  
El objetivo no es solo llegar a root — es entender el *por qué* de cada paso.

---

## 📊 Progreso

| Plataforma | Resueltas |
|------------|-----------|
| DockerLabs | 3 |
| The Hacker Labs | 0 |
| HackTheBox | 0 |
| Otras | 5 |
| **Total** | **8** |

---

## 📁 Índice de máquinas

| Máquina | Plataforma | Dificultad | Técnicas | Writeup |
|---------|------------|------------|----------|---------|
| Firsthacking | DockerLabs | 🟢 Muy fácil | FTP · CVE-2011-2523 · Backdoor vsftpd 2.3.4 · netcat | [writeup](./dockerlabs/muy-facil/firsthacking/writeup.md) |
| BreakMySSH | DockerLabs | 🟢 Muy fácil | Fuerza bruta SSH · Hydra · Metasploit ssh_login | [writeup](./dockerlabs/muy-facil/breakmyssh/writeup.md) |
| Trust | DockerLabs | 🟢 Muy fácil | Gobuster · Fuerza bruta SSH · sudo misconfiguration · GTFOBins (vim) | [writeup](./dockerlabs/muy-facil/trust/writeup.md) |
| crAPI | OWASP crAPI | 🟡 Medio | OTP brute force · Mass Assignment · Business Logic Flaw · NoSQL Injection · BOLA/IDOR | [writeup](./owasp-crapi/writeup.md) |
| File Upload Abuse | File Upload Vulnerability Scenarios | 🟡 Medio | Blacklist de extensión (.php5/.pht) · Bypass Content-Type · MAX_FILE_SIZE · Magic bytes (GIF/JPEG) · Subida de .htaccess · Doble extensión · Gobuster | [writeup](./file-upload-abuse/writeup.md) |
| Prototype Pollution | skf-labs | 🟢 Fácil | Prototype Pollution · Burp Suite · Node.js `_.merge` inseguro | [writeup](./prototype-pollution/writeup.md) |
| DNS Zone Transfer | Vulhub | 🟢 Muy fácil | Transferencia de zona DNS (AXFR) · dig · Enumeración DNS · BIND sin allow-transfer | [writeup](./transferencia-zona-dns/writeup.md) |
| Mass Assignment | OWASP Juice Shop | 🟢 Fácil | Mass Assignment · Burp Repeater · API abuse (campo `role` no filtrado) | [writeup](./mass-assignment-juice-shop/writeup.md) |

---
 
## 📝 Estructura de writeups
 
Todos los writeups siguen esta plantilla:
 
```
## Reconocimiento
## Enumeración web      ← solo si aplica
## Acceso inicial
## Escalada de privilegios
## Lecciones aprendidas
### Desde el punto de vista del atacante
### Desde el punto de vista del defensor
```
 
---
 
## 🛠️ Herramientas habituales
 
`nmap` `gobuster` `hydra` `netcat` `Metasploit` `Burp Suite` `searchsploit`  
`sudo -l` `linpeas` `GTFOBins` `John the Ripper` `Impacket`
 
---
 
## 📚 Recursos útiles
 
- [GTFOBins](https://gtfobins.github.io) — escalada de privilegios Linux
- [HackTricks](https://book.hacktricks.xyz) — referencia general de pentesting
- [RevShells](https://www.revshells.com) — generador de reverse shells
- [ExploitDB](https://www.exploit-db.com) — base de datos de exploits
