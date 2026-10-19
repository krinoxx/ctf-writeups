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
| VulnHub | 3 |
| Otras | 12 |
| **Total** | **18** |
 
---
 
## 📁 Índice de máquinas
 
| Máquina | Plataforma | Dificultad | Técnicas | Writeup |
|---------|------------|------------|----------|---------|
| Firsthacking | DockerLabs | 🟢 Muy fácil | FTP · CVE-2011-2523 · Backdoor vsftpd 2.3.4 · netcat | [writeup](./dockerlabs/muy-facil/firsthacking/writeup.md) |
| BreakMySSH | DockerLabs | 🟢 Muy fácil | Fuerza bruta SSH · Hydra · Metasploit ssh_login | [writeup](./dockerlabs/muy-facil/breakmyssh/writeup.md) |
| Trust | DockerLabs | 🟢 Muy fácil | Gobuster · Fuerza bruta SSH · sudo misconfiguration · GTFOBins (vim) | [writeup](./dockerlabs/muy-facil/trust/writeup.md) |
| crAPI | OWASP crAPI | 🟡 Medio | OTP brute force · Mass Assignment · Business Logic Flaw · NoSQL Injection · BOLA/IDOR | [writeup](./owasp-crapi/writeup.md) |
| File Upload Abuse | File Upload Vulnerability Scenarios | 🟡 Medio | Blacklist de extensión (.php5/.pht) · Bypass Content-Type · MAX_FILE_SIZE · Magic bytes (GIF/JPEG) · Subida de .htaccess · Doble extensión · Gobuster | [writeup](./file-upload-abuse/writeup.md) |
| Prototype Pollution | Skf-labs | 🟢 Fácil | Prototype Pollution · Burp Suite · Node.js `_.merge` inseguro | [writeup](./prototype-pollution/writeup.md) |
| DNS Zone Transfer | Vulhub | 🟢 Muy fácil | Transferencia de zona DNS (AXFR) · dig · Enumeración DNS · BIND sin allow-transfer | [writeup](./transferencia-zona-dns/writeup.md) |
| Mass Assignment | OWASP Juice Shop | 🟢 Fácil | Mass Assignment · Burp Repeater · API abuse (campo `role` no filtrado) | [writeup](./mass-assignment-juice-shop/writeup.md) |
| Open Redirect | Skf-labs | 🟡 Media | Open Redirect (CWE-601) · Bypass de filtros "." y "/" · WHATWG URL parsing · Burp Repeater | [writeup](./open-redirect-skf-labs/writeup.md) |
| WebDAV Auth Bypass | bytemark/webdav (BytemarkHosting) | 🟢 Fácil | WebDAV · Digest Auth · Fuerza bruta (davtest + rockyou) · cadaver · Escritura arbitraria de ficheros | [writeup](./webdav-enum-explotacion/writeup.md) |
| SickOs 1.1 | VulnHub | 🟡 Media | Pivoting vía Squid Proxy · Gobuster · Descubrimiento de puertos internos vía proxy · ShellShock (CVE-2014-6271) · Reverse shell | [writeup](./vulnhub/sickos-1.1/writeup.md) |
| XVWA — XPATH Injection | VulnHub | 🟢 Fácil | XPath Injection · Burp Suite · Extracción ciega de datos (blind) · Automatización en Python (requests) | [writeup](./vulnhub/xvwa-xpath-injection/writeup.md) |
| CORS Hijacking | Skf-labs | 🟢 Fácil | CORS misconfiguration · Access-Control-Allow-Origin: * + Credentials: true · Burp Repeater · Exfiltración vía XHR withCredentials | [writeup](./cors-hijacking/writeup.md) |
| IA: Tornado | VulnHub | 🟡 Media | Gobuster · Info disclosure (listado de directorio Apache + comentario HTML) · SQL Truncation Attack · OS Command Injection · Reverse shell | [writeup](./vulnhub/tornado/writeup.md) |
| Session Puzzling | Skf-labs | 🟡 Media | Session Puzzling · JWT (decodificación de claims) · Burp Suite · Reutilización de cookie de sesión entre flujos (forgot-password → dashboard) | [writeup](./session-puzzling-skf-labs/writeup.md) |
| JWT — SKF Labs | Skf-labs | 🟢 Fácil | JWT · alg NONE/NULL cipher bypass · Forja manual de token sin firma · Fuerza bruta de secreto débil HS256 · jwt.io | [writeup](./jwt-skf-labs/writeup.md) |
| Race Condition | Skf-labs | 🟡 Media | TOCTOU race condition · OS Command Injection · Reverse shell · Condición de carrera en escritura/lectura de fichero compartido · Burp Repeater | [writeup](./race-condition-skf-labs/writeup.md) |
| CSS Injection (CSSI) | Skf-labs | 🟢 Fácil | CSS Injection · Escape de bloque `<style>` (`}</style>`) · XSS reflejado · view-source | [writeup](./css-injection-skf-labs/writeup.md) |
 
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
