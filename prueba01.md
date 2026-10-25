**1. Ficha**
Eventos de Windows (Security y System) exportados con separador `|` · DC01 (dominio EJEMPLO.LOCAL) · 2026-10-20 07:58:00 a 08:56:50 (-06:00) · 106 de 106 líneas · conteos hechos con código · supuesto: no se dio contexto del entorno, así que se trata como un DC de producción.

**2. Resumen**
Estado al cierre del log: ESTABLE.
Solo se ve actividad normal: inicios de sesión Kerberos, GPO aplicada sin cambios y el alta de un usuario por helpdesk. No hay impacto y no hace falta ninguna acción urgente.

**3. Hallazgos**
Sin hallazgos. Hay 105 eventos correctos y 1 solo fallo de preautenticación, seguido de un login correcto 16 s después. Lo más probable es una contraseña mal escrita (confianza Alta):

```
L31 | ... | 4771 | Audit Failure | ... Account=crodriguez Client=10.10.20.61 FailureCode=0x18 ...
L32 | ... | 4768 | Audit Success | ... Account=crodriguez Client=10.10.20.61 Result=0x0 ...
```

El alta de nruiz (L101 a L105: creación, habilitación, reset con cambio obligatorio y alta en GG-Ventas) sigue el procedimiento habitual de onboarding.

**5. Para confirmar**

- Que exista un ticket que autorice el alta de nruiz por hd.lmartinez (L101).
- `Get-WinEvent -FilterHashtable @{LogName='Security';Id=4771,4625}` en DC02, para descartar fallos repartidos entre los dos DC.
