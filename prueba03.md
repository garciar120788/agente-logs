**1. Ficha:** Cisco IOS (syslog de switch Catalyst) · un switch sin hostname en el log · May 16 14:01:22.123 → 14:01:35.500 (13,4 s) · 11 de 11 líneas · por lectura (11 líneas, con intervalos calculados) · Supuestos: separé las líneas por timestamp porque el log llegó en una sola línea; no hay año ni zona horaria; no se indicó cuál es el root bridge esperado.

**2. Resumen:** Estado al cierre del log: **DEGRADADO**.
Gi1/0/1 cayó y STP reconvergió la VLAN 10 por Gi1/0/4 en 5,883 s, con un nuevo root bridge. Gi1/0/1 sigue caído, hay un duplex mismatch hacia SW-Backup y una violación de port-security. Lo primero es confirmar que el nuevo root es legítimo y revisar la capa física de Gi1/0/1.

**3. Hallazgos**

**H1 · Caída de Gi1/0/1 con cambio de root bridge en VLAN 10 · ALTO · activo**
- Evidencia:
```
L1 | May 16 14:01:22.123: %LINK-3-UPDOWN: Interface GigabitEthernet1/0/1, changed state to down
L4 | ...%SPANTREE-6-PORTSTATE: Port Gi1/0/3 state changed from FORWARDING to BLOCKING
L8 | ...%SPANTREE-5-ROOTCHANGE: Root changed for VLAN0010 New root is 00e0.b6ff.ee11, cost is 19, port Gi1/0/4
```
- Qué pasa: el enlace cae a nivel físico (L1–L2). Llega un TCN (L3), Gi1/0/3 pasa a bloqueo y Gi1/0/4 pasa por listening, learning y forwarding (L5–L7). El cambio de root ocurre 5,883 s después de la caída. L3–L7 son síntomas de esta caída.
- Causa probable: (1) Fallo físico de cable, SFP o equipo vecino en Gi1/0/1, que llevaba hacia el root anterior. Confianza Media. L1 es LINK-3 y no "administratively down", lo que apunta a algo físico y no a un `shutdown` (referencia general). (2) Un switch no previsto que se anuncia como root. La explicación benigna es una conmutación por falla al root de respaldo; la maliciosa, un ataque de root bridge. Confianza Baja.
- Verificar: `show spanning-tree vlan 10 detail` y `show interfaces Gi1/0/1`
- Resolver: revisar o reemplazar el cable o SFP de Gi1/0/1 `[IMPACTO: corta tráfico]` (STP vuelve a reconverger). Si 00e0.b6ff.ee11 no es el root esperado, activar `spanning-tree guard root` en Gi1/0/4 `[IMPACTO: cambia configuración | corta tráfico]`.
- Prevenir: fijar la prioridad del root primario y del secundario en el core, aplicar root guard en los downlinks y valorar rapid-pvst.

**H2 · Duplex mismatch con SW-Backup · MEDIO · activo**
- Evidencia:
```
L9 | May 16 14:01:30.000: %CDP-4-DUPLEX_MISMATCH: duplex mismatch discovered on GigabitEthernet1/0/2 (full), with SW-Backup GigabitEthernet0/1 (half).
```
- Qué pasa: Gi1/0/2 trabaja en full duplex y el otro extremo en half.
- Causa probable: autonegociación desactivada o forzada en un solo extremo. Confianza Alta. Suele provocar colisiones tardías y CRC (referencia general).
- Verificar: `show interfaces Gi1/0/2 | include duplex|collision|CRC`
- Resolver: igualar speed y duplex en ambos extremos, ya sea auto/auto o forzado en los dos `[IMPACTO: cambia configuración | corta tráfico]`.
- Prevenir: una plantilla estándar de configuración de interfaces y una revisión periódica de CDP.

**H3 · Posible dispositivo no autorizado en Gi1/0/5 · MEDIO · indeterminado**
- Evidencia:
```
L10 | May 16 14:01:32.500: %PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred on interface GigabitEthernet1/0/5
```
- Qué pasa: hubo una violación de port-security. El log no muestra la MAC ni un err-disable.
- Causa probable: la explicación benigna es un equipo legítimo cambiado o reconectado. La maliciosa es un dispositivo ajeno conectado. Confianza Baja para ambas.
- Verificar: `show port-security interface Gi1/0/5` y `show interfaces status err-disabled`
- Resolver: identificar la MAC. Si el equipo es legítimo, ajustar la MAC permitida y recuperar el puerto `[IMPACTO: cambia configuración]`. Si no lo es, dejar el puerto bloqueado y localizar físicamente el equipo.
- Prevenir: 802.1X o MACs sticky documentadas, más `errdisable recovery` controlado.

**H4 · Cambio de configuración por VTY sin detalle · BAJO · indeterminado**
- Evidencia:
```
L11 | May 16 14:01:35.500: %SYS-5-CONFIG_I: Configured from console by vty0 (192.168.1.5)
```
- Qué pasa: alguien configuró el switch desde 192.168.1.5, 13,377 s después de L1. El mensaje no incluye usuario.
- Causa probable: lo más probable es un administrador actuando ante la incidencia. Un acceso no autorizado no puede descartarse. Confianza Baja. CONFIG_I se registra al salir del modo de configuración, así que el cambio pudo empezar antes de L1 (referencia general).
- Verificar: `show archive log config all` y `show users`
- Resolver: confirmar el cambio con el responsable de 192.168.1.5. Si no está autorizado, restringir las VTY con `access-class` `[IMPACTO: bloquea accesos]`.
- Prevenir: AAA con usuarios individuales, `archive log config` y ACL en las líneas VTY.

**4. Otras observaciones**
- Gi1/0/4 pasa 2,002 s en listening y 2,002 s en learning. Eso queda por debajo del forward-delay mínimo de 802.1D, que es 4 s (referencia general). Puede ser un timer no estándar, un entorno de laboratorio o un problema de timestamps.
- El log no incluye hostname, año ni números de secuencia, lo que complica correlacionarlo con otras fuentes.

**5. Para confirmar**
- El log anterior a 14:01:22 y el root bridge esperado en la VLAN 10.
- La MAC que causó la violación en Gi1/0/5 (`show port-security address`).
- A quién pertenece 192.168.1.5 y qué se cambió (archive o AAA accounting).
