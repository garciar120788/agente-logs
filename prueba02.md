**1. Ficha:** syslog de Linux (`/var/log/messages`, estilo RHEL/Fedora) · host `combo` · del 14 jun 15:16:01 al 27 jul 14:42:00 (el año 2005 sale de las líneas de ftpd) · 2000 de 2000 líneas · analizado con código · supuestos: el log no indica zona horaria; no se dio contexto del entorno.

**2. Resumen:** Estado al cierre del log: **DEGRADADO**.
Hubo fuerza bruta SSH sostenida contra root, a la vez que la cuenta `test` abría sesiones SSH en ráfagas desde una IP que el log no registra. Además, logrotate falla todos los días. Lo primero es revisar la cuenta `test`: de dónde entra y si su uso es legítimo.

**3. Hallazgos**

**H1 · Posible uso no autorizado de la cuenta `test` por SSH · ALTO · indeterminado**
- Evidencia (36 aperturas y 36 cierres, de L92 a L1279):
```
L585 | Jun 30 22:16:32 combo sshd(pam_unix)[19432]: session opened for user test by (uid=509)
L604 | Jun 30 22:16:33 combo sshd(pam_unix)[19440]: session closed for user test
L953 | Jul  8 20:14:55 ... rhost=212.0.132.20  user=test
```
- Qué pasa: se abren y cierran hasta 10 sesiones en 1 s, y cada una dura un segundo o menos. Es un ritmo automático. El log no muestra la IP de origen.
- Causa probable: un script legítimo (prueba o monitorización) que usa `test` (Media). Una cuenta débil comprometida que usa un bot, en un host bajo fuerza bruta (Media).
- Verificar: `grep -E "Accepted .* for test" /var/log/secure*; last -i test`
- Resolver: `passwd -l test` `[IMPACTO: bloquea accesos]` hasta confirmar el origen.
- Prevenir: `PasswordAuthentication no` y `AllowUsers` en sshd_config `[IMPACTO: cambia configuración]`.

**H2 · Posible ataque de fuerza bruta SSH · MEDIO · activo (último en L1901)**
- Evidencia (489 fallos desde 47 orígenes: 351 con user=root y 116 de "user unknown"):
```
L1136 | Jul 10 16:01:43 ... rhost=150.183.249.110  user=root
L1215 | Jul 10 16:03:18 ... rhost=150.183.249.110  user=root
L1901 | Jul 26 07:04:12 ... rhost=207.243.167.114  user=root
```
- Qué pasa: desde 150.183.249.110 llegan 80 intentos en 95 s. Ningún fallo de root va seguido de una sesión de root por SSH.
- Causa probable: escaneo automatizado de Internet (Alta). La alternativa benigna, alguien que olvidó su contraseña, no explica 47 orígenes.
- Verificar: `grep "Failed password" /var/log/secure | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn`
- Resolver: instalar fail2ban o bloquear las IPs principales `[IMPACTO: bloquea accesos]`.
- Prevenir: `PermitRootLogin no` `[IMPACTO: cambia configuración]`.

**H3 · logrotate falla a diario · MEDIO · activo**
- Evidencia (43 veces, de L16 a L1904):
```
L1904 | Jul 27 04:16:09 combo logrotate: ALERT exited abnormally with [1]
```
- Qué pasa: la rotación diaria sale con código 1 todos los días.
- Causa probable: un error en algún fichero de `/etc/logrotate.d` o un postrotate que falla (Media). Riesgo: que se llene `/var/log` (referencia general).
- Verificar: `logrotate -d /etc/logrotate.conf; df -h /var/log`
- Resolver: corregir la entrada que falla y relanzar `logrotate -f` `[IMPACTO: cambia configuración]`.
- Prevenir: alertar cuando el código de salida de cron.daily no sea 0.

**H4 · Posible escaneo masivo de FTP · BAJO · activo**
- Evidencia (909 conexiones desde 38 IPs, en ráfagas de 23 en el mismo segundo):
```
L1358 | Jul 17 04:06:32 combo ftpd[19675]: connection from 83.116.207.11 ...
```
- Qué pasa: solo aparecen conexiones. No hay ningún resultado de login, ni bueno ni malo.
- Causa probable: escáneres o bots (Media). Clientes FTP con muchas conexiones en paralelo (Baja).
- Verificar: `grep -i "ftpd.*\(login\|fail\)" /var/log/xferlog /var/log/secure`
- Resolver: limitar FTP a IPs conocidas `[IMPACTO: corta tráfico]`.
- Prevenir: sustituir FTP por SFTP.

**4. Otras observaciones**

- Hay 46 fallos de Kerberos en klogind desde 163.27.187.39 en unos 2 s (L539–L584).
- El reinicio del 27 jul (L1906 `syslogd 1.4.1: restart.`) no tiene ningún mensaje de apagado antes. El último evento anterior es L1905, a las 10:59. Además, en L1983–L1991 los timestamps retroceden, lo que es normal en el arranque (referencia general).
- named registra 16 avisos de `notify question section contains no SOA` el 25 jul (L1810–L1825).

**5. Para confirmar**

- `/var/log/secure`: líneas "Accepted" con la IP de origen de `test`.
- Qué hizo `test` en el sistema: `~test/.bash_history`, procesos y crontab de uid 509.
- `last -x` para saber si el reinicio del 27 jul fue limpio.
