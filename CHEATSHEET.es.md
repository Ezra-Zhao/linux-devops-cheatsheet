# Linux / DevOps — Hoja de referencia en una página 🇪🇸

Guía rápida de comandos para sysadmins, SREs y DevOps. Cada comando lleva una explicación de una línea y ejemplos reales de diagnóstico.

> Convención: `#` = se ejecuta como root (sudo). Rutas y nombres entre `<>` hay que reemplazarlos.

---

## 1. Archivos y directorios

| Comando | Para qué sirve |
|---|---|
| `ls -lah` | Lista con tamaños legibles, ocultos y permisos |
| `tree -L 2 -h` | Árbol del directorio, 2 niveles, con tamaños |
| `find /var/log -name "*.log" -mtime -7` | Archivos `.log` modificados en los últimos 7 días |
| `find / -xdev -size +500M -exec ls -lh {} \;` | Archivos de más de 500 MB (sin cruzar filesystems) |
| `stat <archivo>` | Metadatos completos: inodo, permisos, fechas de acceso/modificación |
| `cp -a origen destino` | Copia conservando permisos, enlaces y fechas (modo archivo) |
| `mv -i origen destino` | Mueve/renombra pidiendo confirmación antes de sobrescribir |
| `rm -i <archivo>` | Borra pidiendo confirmación (úsalo siempre con comodines) |
| `touch -d "2024-01-01" <archivo>` | Cambia la fecha de un archivo (útil en pruebas) |
| `realpath <archivo>` | Resuelve la ruta absoluta siguiendo enlaces simbólicos |
| `file <archivo>` | Dice qué tipo de archivo es realmente (no se fía de la extensión) |

🔧 **Diagnóstico 1 — El disco se llena y no sabes con qué:**
```bash
du -x --max-depth=2 / 2>/dev/null | sort -rn | head -20
# Baja al directorio culpable y repite hasta encontrar el archivo gordo.
```

🔧 **Diagnóstico 2 — Un archivo borrado sigue ocupando espacio:**
```bash
lsof +L1 | grep -i deleted
# El proceso aún tiene el descriptor abierto. Reinicia el servicio o haz
# : > /proc/<PID>/fd/<FD> para truncarlo sin reiniciar.
```

---

## 2. Procesamiento de texto: grep, sed, awk, jq

| Comando | Para qué sirve |
|---|---|
| `grep -RniE "error\|timeout" /var/log/app/` | Busca (recursivo, sin mayús/minús, con nº de línea, regex extendida) |
| `grep -v "^#" nginx.conf` | Muestra el fichero sin líneas comentadas |
| `grep -A5 -B2 "OOM" syslog` | 5 líneas después y 2 antes de cada coincidencia |
| `sed -n '100,200p' app.log` | Imprime solo las líneas 100–200 |
| `sed -i 's/foo/bar/g' config.yaml` | Reemplaza en el fichero (¡haz backup antes!) |
| `awk '{print $1, $9}' access.log` | Imprime las columnas 1 y 9 (IPs y códigos HTTP) |
| `awk '$9 >= 500' access.log \| wc -l` | Cuenta las peticiones con error 5xx |
| `cut -d: -f1 /etc/passwd` | Primera columna separada por `:` (lista de usuarios) |
| `sort access.log \| uniq -c \| sort -rn \| head` | Top de líneas repetidas (IPs más activas) |
| `jq '.status, .latency' respuesta.json` | Extrae campos de un JSON |
| `jq -r '.[] \| "\(.ip) \(.code)"' datos.json` | Salida en texto plano, un registro por línea |
| `tr 'A-Z' 'a-z' < datos.txt` | Pasa todo a minúsculas |
| `wc -l <archivo>` | Cuenta líneas (más rápido que `cat \| wc`) |

🔧 **Diagnóstico 1 — ¿Qué IPs están pegando más fuerte al servidor?**
```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
```

🔧 **Diagnóstico 2 — Errores 5xx por minuto en la última hora:**
```bash
awk '$9 >= 500 {print substr($4, 2, 17)}' access.log | uniq -c
# $4 es [10/Oct/2026:08:15:33 ; substr recorta a minuto. Picos = incidentes.
```

---

## 3. Permisos y usuarios

| Comando | Para qué sirve |
|---|---|
| `chmod 640 <archivo>` | rw para dueño, r para grupo, nada para otros |
| `chmod -R u+rwX,g+rX,o-rwx <dir>` | Permisos sanos en un árbol: X solo en dirs/ejecutables |
| `chown -R app:app /opt/app` | Cambia dueño y grupo recursivamente |
| `umask` / `umask 027` | Muestra/cambia la máscara de permisos por defecto |
| `id <usuario>` | UID, GID y grupos del usuario |
| `getent passwd <usuario>` | Info del usuario (funciona también con LDAP/SSSD) |
| `sudo -l` | Qué puede ejecutar tu usuario con sudo (auditoría rápida) |
| `useradd -m -s /bin/bash <usuario>` | Crea usuario con home y shell |
| `passwd -S <usuario>` / `passwd -l <usuario>` | Estado de la contraseña / bloquear cuenta |
| `last -n 20` | Últimos 20 inicios de sesión (detecta accesos raros) |
| `ls -lZ <archivo>` | Contexto SELinux (si `Permission denied` sin motivo aparente) |

🔧 **Diagnóstico 1 — "Permission denied" aunque el permiso parece correcto:**
```bash
namei -l /opt/app/data/archivo.log
# Muestra los permisos de CADA componente de la ruta. El 90% de las veces
# el problema está en un directorio padre sin permiso x.
```

🔧 **Diagnóstico 2 — La app no puede escribir su log tras un despliegue:**
```bash
ls -ld /var/log/app && id app
# Compara dueño del directorio vs usuario del proceso. Corrige con
# chown app:app /var/log/app o ajusta el User= del servicio systemd.
```

---

## 4. Procesos

| Comando | Para qué sirve |
|---|---|
| `ps aux --sort=-%mem \| head -15` | Top 15 procesos por consumo de memoria |
| `ps -ef | grep <nombre>` | Busca un proceso por nombre (mejor: `pgrep -af`) |
| `pgrep -af python` | PIDs y línea completa de los procesos python |
| `top` / `htop` | Monitor interactivo (htop: más visual, permite matar con F9) |
| `kill <PID>` | SIGTERM: pide al proceso que termine limpiamente |
| `kill -9 <PID>` | SIGKILL: mata sin avisar (último recurso, puede dejar locks) |
| `pkill -f "gunicorn.*worker"` | Mata por patrón de línea de comando |
| `lsof -i :8080` | Qué proceso está escuchando en el puerto 8080 |
| `lsof -p <PID>` | Archivos y sockets abiertos por un proceso |
| `nice -n 10 <cmd>` / `renice -n 5 -p <PID>` | Baja/sube la prioridad de CPU |
| `nohup <cmd> &` | Ejecuta inmune al cierre de la terminal |
| `strace -p <PID> -e trace=open,read` | Ve las syscalls de un proceso en vivo (diagnóstico profundo) |

🔧 **Diagnóstico 1 — Un puerto "ya está en uso" al arrancar el servicio:**
```bash
ss -tlnp | grep 8080
# Te dice el PID exacto. Decide: ¿es un proceso zombie de un despliegue
# anterior? Mátalo con kill <PID>, no con -9 si puedes evitarlo.
```

🔧 **Diagnóstico 2 — La máquina va lenta y no sabes qué proceso es:**
```bash
ps -eo pid,ppid,%cpu,%mem,cmd --sort=-%cpu | head -10
dmesg -T | grep -i "oom\|killed process" | tail -5
# Si el OOM killer actuó, verás qué proceso sacrificó el kernel.
```

---

## 5. systemd: servicios

| Comando | Para qué sirve |
|---|---|
| `systemctl status <servicio>` | Estado, últimos logs y PID del servicio |
| `systemctl restart <servicio>` | Reinicia el servicio |
| `systemctl enable --now <servicio>` | Activa en el arranque Y lo arranca ya |
| `systemctl is-enabled <servicio>` | ¿Arranca solo al boot? |
| `systemctl daemon-reload` | Recarga la configuración tras editar un unit |
| `systemctl list-units --failed` | Servicios que fallaron (tu primer vistazo tras un reboot) |
| `systemctl list-timers --all` | Timers programados (el "cron" de systemd) |
| `systemd-analyze blame | head` | Qué ralentiza el arranque |
| `journalctl -u <servicio> -n 100 --no-pager` | Últimas 100 líneas del log del servicio |
| `journalctl -u <servicio> -f` | Sigue el log en vivo |
| `journalctl --since "30 min ago" -p err` | Errores de todo el sistema en la última media hora |

🔧 **Diagnóstico 1 — El servicio arranca y muere al instante:**
```bash
systemctl status mi-app -l --no-pager
journalctl -u mi-app -n 50 --no-pager
# El 80% de las veces: ruta mala en ExecStart, permiso denegado en
# WorkingDirectory, o falta una variable en EnvironmentFile.
```

🔧 **Diagnóstico 2 — Tras editar el .service no coge los cambios:**
```bash
systemctl daemon-reload && systemctl restart mi-app
# Sin daemon-reload, systemd sigue usando la definición vieja en memoria.
```

---

## 6. Red: diagnóstico

| Comando | Para qué sirve |
|---|---|
| `ss -tlnp` | Puertos TCP en escucha con proceso (el `netstat` moderno) |
| `ss -tupn` | TCP + UDP con procesos |
| `ip -brief addr` | IPs de cada interfaz, en una línea por interfaz |
| `ip route` | Tabla de rutas (¿por dónde sale el tráfico?) |
| `ping -c 4 <host>` | Conectividad básica (4 paquetes) |
| `mtr --report <host>` | Traceroute + pérdida por salto (ideal para tickets con el ISP) |
| `curl -sSI https://<host>` | Solo cabeceras HTTP (¿responde? ¿qué código?) |
| `curl -w "dns:%{time_namelookup} conn:%{time_connect} tls:%{time_appconnect} total:%{time_total}\n" -o /dev/null -s https://<host>` | Desglose de tiempos: DNS, conexión, TLS, total |
| `dig +short <dominio>` | Resolución DNS rápida |
| `dig +trace <dominio>` | Traza completa desde la raíz (detecta delegaciones rotas) |
| `host <dominio> 8.8.8.8` | Pregunta a un DNS concreto (¿es mi DNS o es global?) |
| `nc -zv <host> <puerto>` | ¿Está abierto ese puerto TCP? |
| `tcpdump -i eth0 -n port 443 -c 20` | Captura 20 paquetes del puerto 443 (sin resolver nombres) |
| `getent hosts <dominio>` | Cómo resuelve el sistema (respeta /etc/hosts y nsswitch) |

🔧 **Diagnóstico 1 — "La app no llega a la base de datos":**
```bash
nc -zv db-interna 5432        # ¿puerto abierto?
getent hosts db-interna       # ¿resuelve el nombre?
ip route get <ip-db>          # ¿por qué interfaz/gateway sale?
# Si el puerto está cerrado pero el host responde: firewall o servicio caído.
```

🔧 **Diagnóstico 2 — HTTPS lento, ¿dónde se pierde el tiempo?**
```bash
curl -w "dns:%{time_namelookup} tcp:%{time_connect} tls:%{time_appconnect} srv:%{time_starttransfer}\n" -o /dev/null -s https://api.ejemplo.com
# tls alto = handshake pesado o red con latencia; srv alto = backend lento.
```

---

## 7. Disco y filesystems

| Comando | Para qué sirve |
|---|---|
| `df -hT` | Espacio por filesystem, con tipo (ext4, xfs…) |
| `df -i` | Inodos libres (disco con espacio pero "lleno" = sin inodos) |
| `du -sh <dir>` | Tamaño total de un directorio |
| `lsblk -f` | Discos, particiones y filesystems |
| `mount \| column -t` | Qué hay montado y con qué opciones |
| `findmnt` | Árbol de montajes (más legible que mount) |
| `dumpe2fs -h /dev/sda1 \| grep -i "mount count\|state"` | Estado del filesystem ext4 |
| `tune2fs -l /dev/sda1 \| grep -i reserved` | Bloques reservados para root (5% por defecto) |
| `fstrim -av` | Libera bloques no usados en SSD (discard manual) |

🔧 **Diagnóstico 1 — `df` dice 100% pero borraste archivos y sigue lleno:**
```bash
lsof +L1 | grep deleted
# Archivos borrados con descriptores abiertos. Trúncalos vía /proc o
# reinicia el servicio dueño. df solo baja cuando se cierran.
```

🔧 **Diagnóstico 2 — "No space left on device" con espacio libre:**
```bash
df -i /
# Sin inodos libres: millones de ficheros pequeños (cachés, sesiones, colas).
# Busca el culpable: find /var/spool -xdev | head
```

---

## 8. Logs

| Comando | Para qué sirve |
|---|---|
| `tail -f /var/log/app.log` | Sigue el log en vivo |
| `tail -n 200 /var/log/app.log` | Últimas 200 líneas |
| `journalctl -f` | Sigue el journal del sistema en vivo |
| `journalctl --since today -p warning` | Avisos y errores de hoy |
| `journalctl -b -1 -p err` | Errores del arranque anterior (tras un cuelgue) |
| `grep -c "ERROR" app.log` | Cuenta errores (tendencia rápida) |
| `awk '/ERROR/ {print $1, $2}' app.log | uniq -c` | Errores agrupados por hora |
| `logrotate --debug /etc/logrotate.conf` | Simula la rotación sin ejecutarla |
| `dmesg -T | tail -30` | Últimos mensajes del kernel con fecha legible |

🔧 **Diagnóstico 1 — El servidor se reinició solo de madrugada:**
```bash
journalctl -b -1 -p err --no-pager | tail -30
last -x reboot shutdown | head
# Cruza: ¿hubo OOM? ¿kernel panic? ¿lo reinició alguien (last muestra el usuario)?
```

🔧 **Diagnóstico 2 — Necesitas los logs de ayer de un servicio:**
```bash
journalctl -u nginx --since yesterday --until today --no-pager | grep " 5[0-9][0-9] "
# --since/--until aceptan "yesterday", "-2h", "2026-10-04 08:00".
```

---

## 9. SSH, SCP y rsync

| Comando | Para qué sirve |
|---|---|
| `ssh -i ~/.ssh/clave.pem usuario@host` | Conecta con clave específica |
| `ssh -J bastion usuario@interna` | Salta por el bastión (ProxyJump, sin túneles manuales) |
| `ssh -L 8080:localhost:8080 usuario@host` | Túnel: tu :8080 → :8080 remoto |
| `ssh -N -f -L 5432:db:5432 usuario@host` | Túnel en segundo plano (base de datos local) |
| `ssh-copy-id -i ~/.ssh/id_rsa.pub usuario@host` | Instala tu clave pública en el servidor |
| `ssh-keygen -t ed25519 -C "mi-laptop"` | Genera clave moderna (ed25519, no RSA-1024) |
| `scp -r ./app usuario@host:/opt/` | Copia recursiva por SSH |
| `rsync -avz --delete origen/ usuario@host:/destino/` | Sincroniza (comprime, verboso, borra lo que sobra en destino) |
| `rsync -avz --progress archivo.iso usuario@host:/tmp/` | Copia grande con barra de progreso |
| `rsync -avzn origen/ destino/` | Dry-run: muestra qué haría sin hacerlo |

🔧 **Diagnóstico 1 — SSH pide contraseña aunque tienes la clave:**
```bash
ssh -v usuario@host   # mira qué métodos ofrece el servidor
# Causas típicas: ~/.ssh con permiso 777 (debe ser 700, authorized_keys 600),
# o el home del usuario escribible por otros (StrictModes lo rechaza).
```

🔧 **Diagnóstico 2 — rsync entre servidores va lentísimo:**
```bash
rsync -av --progress origen/ usuario@host:/destino/
# Prueba SIN -z: si la red es rápida y los datos no comprimen (videos, zips),
# la compresión solo añade CPU. Y usa --partial para reanudar cortes.
```

---

## 10. Gestión de paquetes

| Comando (Debian/Ubuntu) | Comando (RHEL/Rocky) | Para qué sirve |
|---|---|---|
| `apt update && apt upgrade -y` | `dnf upgrade -y` | Actualiza el sistema |
| `apt list --upgradable` | `dnf check-update` | Qué se puede actualizar |
| `apt install -y <paq>` | `dnf install -y <paq>` | Instala |
| `apt remove <paq>` / `apt purge <paq>` | `dnf remove <paq>` | Desinstala (purge borra configs) |
| `dpkg -l \| grep <paq>` | `rpm -qa \| grep <paq>` | ¿Está instalado? ¿qué versión? |
| `dpkg -S /usr/bin/<bin>` | `rpm -qf /usr/bin/<bin>` | ¿A qué paquete pertenece este binario? |
| `dpkg -L <paq>` | `rpm -ql <paq>` | Qué archivos instaló el paquete |
| `apt-cache policy <paq>` | `dnf info <paq>` | Versión instalada vs disponible, de qué repo viene |

🔧 **Diagnóstico 1 — Un binario se comporta raro tras actualizar:**
```bash
dpkg -S $(which nginx) && dpkg -l | grep nginx
# Confirma qué paquete lo provee y su versión. Si hay dos binarios
# (ej. /usr/bin y /usr/local/bin), which te dice cuál se ejecuta.
```

🔧 **Diagnóstico 2 — `apt update` falla con errores de GPG:**
```bash
apt update 2>&1 | grep -i "expired\|NO_PUBKEY"
# Clave expirada o repo sin firmar. No uses --allow-unauthenticated
# en producción: reimporta la clave oficial del vendor.
```

---

## 11. Docker: lo esencial

| Comando | Para qué sirve |
|---|---|
| `docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"` | Contenedores en tabla legible |
| `docker logs -f --tail 100 <cont>` | Logs en vivo, últimas 100 líneas |
| `docker exec -it <cont> bash` | Entra al contenedor (sh si no hay bash) |
| `docker stats --no-stream` | CPU/RAM por contenedor, una foto |
| `docker inspect <cont> \| jq '.[0].NetworkSettings.IPAddress'` | IP del contenedor |
| `docker compose up -d` / `docker compose down` | Levanta/baja el stack |
| `docker compose logs -f <svc>` | Logs de un servicio del compose |
| `docker system df` | Cuánto disco usan imágenes, contenedores y volúmenes |
| `docker system prune -a --volumes` | Limpieza agresiva (¡borra volúmenes no usados!) |
| `docker cp <cont>:/app/log.txt ./` | Saca un archivo del contenedor |

🔧 **Diagnóstico 1 — El contenedor arranca y se apaga (Restarting):**
```bash
docker logs <cont> --tail 50
docker inspect <cont> --format '{{.State.ExitCode}} {{.Config.Cmd}}'
# ExitCode 1 + log de "connection refused" = arrancó antes que la DB.
# Solución real: depends_on con healthcheck, no restart: always a ciegas.
```

🔧 **Diagnóstico 2 — "No space left" y el culpable es Docker:**
```bash
docker system df
# Casi siempre: capas de builds viejos. Limpia con
# docker builder prune y docker image prune -a (revisa antes qué borras).
```

---

## 12. Rendimiento: CPU, memoria, I/O

| Comando | Para qué sirve |
|---|---|
| `uptime` | Carga media 1/5/15 min (compárala con nº de CPUs) |
| `nproc` | Nº de CPUs disponibles |
| `vmstat 1 5` | Foto de CPU, memoria, swap e I/O cada segundo |
| `free -h` | Memoria legible (mira "available", no "free") |
| `iostat -x 1 3` | Por dispositivo: %util, await, r/s w/s |
| `iotop -o` | Qué procesos están haciendo I/O ahora mismo |
| `sar -u 1 3` / `sar -r 1 3` | Histórico de CPU / memoria (si sysstat guarda datos) |
| `dmesg -T \| grep -i "error\|fail" \| tail` | Errores de hardware/kernel recientes |
| `cat /proc/loadavg` | Carga + procesos en ejecución/cola |
| `perf top` | Qué funciones consumen CPU (cuando top no basta) |

🔧 **Diagnóstico 1 — Load average por las nubes pero CPU libre:**
```bash
vmstat 1 5
# Si la columna 'b' (bloqueados) es alta y 'wa' (iowait) también: el cuello
# es DISCO, no CPU. Sigue con iostat -x para ver qué dispositivo satura.
```

🔧 **Diagnóstico 2 — ¿Es falta de RAM o es normal?**
```bash
free -h
# En Linux, "buff/cache" alto es SANO (caché de disco). Preocúpate si
# "available" baja de ~10% o si "Swap used" crece sin parar.
```

---

## 13. Seguridad: lo que nunca debes hacer

| Regla | Por qué |
|---|---|
| ❌ `rm -rf /` o `rm -rf $VAR/` con VAR vacía | Borra el sistema. Usa rutas absolutas y `-i` |
| ❌ `chmod -R 777` | Regala escritura+ejecución a todo el mundo |
| ❌ `curl http://... \| sudo bash` | Ejecutas código sin leerlo. Descarga, lee, luego ejecuta |
| ❌ Contraseñas en la línea de comandos | Quedan en `history` y visibles en `ps`. Usa variables de entorno o ficheros con permiso 600 |
| ❌ SSH como root con contraseña | Solo clave + `PermitRootLogin prohibit-password` |
| ✅ `sudo -i` y revisa `/var/log/auth.log` | Auditoría de quién hizo sudo y cuándo |
| ✅ `ufw allow 22/tcp && ufw enable` | Firewall mínimo viable en un VPS nuevo |
| ✅ `fail2ban-client status sshd` | ¿Está baneando IPs que fuerzan SSH? |
| ✅ Actualiza: `dnf upgrade -y` / `apt upgrade -y` | La mayoría de brechas entran por software sin parchear |

🔧 **Diagnóstico 1 — Sospechas de intrusión:**
```bash
last -n 30                  # ¿logins a horas raras o desde IPs extrañas?
ss -tlnp | grep -v -E ":(22|80|443) "   # ¿puertos escuchando que no reconoces?
grep -i "failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head
# Top de IPs con fallos de SSH. Si una tiene miles: fail2ban debería haberla baneado.
```

---

## Mini-chuleta de emergencia 🚨

```bash
# La máquina no responde: ¿qué pasa?
uptime; free -h; df -h /; ss -tlnp | head; systemctl list-units --failed

# Un servicio caído: el ritual
systemctl status <svc>; journalctl -u <svc> -n 50 --no-pager; ss -tlnp | grep <puerto>

# Red rota: el ritual
ping -c2 8.8.8.8; ip route; getent hosts <dominio>; curl -sSI https://<dominio> | head -3

# Disco lleno: el ritual
df -h; df -i; du -x --max-depth=2 / 2>/dev/null | sort -rn | head; lsof +L1 | grep deleted
```

---

*Contenido 100% original, escrito desde cero por el autor. Sin copias de otras guías.*
