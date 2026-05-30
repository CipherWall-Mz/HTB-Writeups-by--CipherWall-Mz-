 # 🛡️ HTB Writeups by CipherWall-Mz
 
![HackTheBox](https://img.shields.io/badge/Platform-HackTheBox-9fef00?style=for-the-badge&logo=hackthebox&logoColor=black)
![Writeups](https://img.shields.io/badge/Writeups-En%20progreso-blue?style=for-the-badge)
![Enfoque](https://img.shields.io/badge/Enfoque-Pentesting%20Ético-red?style=for-the-badge)
 
Repositorio dedicado a máquinas de **Hack The Box (HTB)**, donde documento writeups con fines educativos. El objetivo es registrar metodologías, técnicas de enumeración, explotación y escalada de privilegios para el aprendizaje práctico de ciberseguridad ofensiva.
 
> ⚠️ **Aviso legal:** Todo el contenido de este repositorio fue realizado en entornos controlados y autorizados de HackTheBox. Las técnicas documentadas aquí son únicamente con fines educativos. Aplicarlas fuera de entornos autorizados es ilegal.
 
---
 
## 👤 Sobre mí
 
Soy **CipherWall-Mz**, entusiasta de la ciberseguridad enfocado en pentesting ético y CTFs. Este repositorio es mi diario de aprendizaje — cada writeup representa no solo la solución a una máquina, sino los conceptos y técnicas que fui consolidando en el proceso.
 
---
 
## 🗂️ Índice de máquinas
 
| Máquina | OS | Dificultad | Técnicas principales | Writeup |
|---------|----|------------|----------------------|---------|
| Reactor | Linux | Easy | CVE-2025-55182, Node.js --inspect, SUID | [📄 Ver](./HTB/Reactor/README.md) |
 
> El índice se irá actualizando a medida que se agreguen nuevas máquinas.
 
---
 
## 🧠 Metodología general
 
La metodología que sigo en cada máquina está basada en las fases estándar de pentesting:
 
### 1. 🔍 Reconocimiento
Identificación de puertos, servicios y versiones expuestas.
```bash
nmap -p- --open -sS --min-rate 5000 -Pn -n <IP> -oG allPorts
nmap -sCV <IP> -oN targeted
```
 
### 2. 🌐 Enumeración
Análisis profundo de los servicios identificados: directorios web, usuarios, tecnologías, versiones vulnerables.
 
### 3. 💥 Explotación
Uso de exploits públicos (CVEs), vulnerabilidades de configuración o lógica de aplicación para obtener acceso inicial.
 
### 4. 🔄 Post-explotación
Enumeración interna del sistema comprometido: puertos locales, procesos, archivos sensibles, credenciales.
 
### 5. ⬆️ Escalada de privilegios
Identificación y abuso de vectores para obtener acceso como root o Administrator.
 
---
 
## 🛠️ Herramientas frecuentes
 
| Categoría | Herramientas |
|-----------|-------------|
| Reconocimiento | `nmap`, `rustscan`, `masscan` |
| Web | `gobuster`, `ffuf`, `feroxbuster`, `burpsuite`, `nikto`, `whatweb`, `wappalyzer` |
| Explotación | `metasploit`, `searchsploit`, `sqlmap`, `hydra`, `medusa` |
| Shells | `nc`, `pwncat-cs`, `socat`, `rlwrap` |
| Cracking | `hashcat`, `john`, `crackstation`, `haiti` |
| Escalada Linux | `linpeas`, `linenum`, `pspy`, `GTFOBins`, `sudo -l`, `find / -perm -4000` |
| Escalada Windows | `winpeas`, `PowerUp`, `BloodHound`, `SharpHound`, `mimikatz` |
| Active Directory | `bloodhound`, `crackmapexec`, `evil-winrm`, `impacket`, `kerbrute`, `rubeus` |
| Forense / Análisis | `strings`, `binwalk`, `exiftool`, `volatility`, `stegseek` |
| Red / MITM | `wireshark`, `tcpdump`, `responder`, `bettercap` |
| Transferencia | `wget`, `curl`, `python3 -m http.server`, `scp`, `certutil` |
| Tunneling | `ssh -L`, `chisel`, `ligolo-ng`, `proxychains`, `socat` |
| Misc | `sqlite3`, `cyberchef`, `haiti`, `ghidra`, `gdb` |
 
---

 
## 📚 Técnicas documentadas hasta ahora
 
- **CVE-2025-55182** — RCE sin autenticación en Next.js 15 vía React Server Components
- **Node.js `--inspect` privesc** — Abuso del debugger de Node corriendo como root
- **SUID bash** — Escalada via binario con bit SUID
- **Hash cracking MD5** — Extracción y crackeo de credenciales desde SQLite
- **Port forwarding SSH** — Tunneling para acceder a servicios internos
- **Reverse shell estabilization** — PTY con Python, stty raw
---
 
## 📁 Estructura del repositorio
 
```
HTB-Writeups/
├── README.md               ← Este archivo
└── HTB/
    └── Reactor/
        └── README.md       ← Writeup completo
```
 
---
 
## 🤝 Contribuciones
 
Este es un repositorio personal de aprendizaje. Si encuentras algún error o tienes sugerencias, abre un **Issue** o manda un **Pull Request**.
 
---

*Hecho con dedicación y mucho esfuerzo — CipherWall-Mz* 🧱
 
