# Oracle DBA Administration — Backup & Recovery con RMAN

Proyecto de portafolio enfocado en tareas de **administración de bases de datos Oracle** (no de análisis de datos): instalación de Oracle Database XE en un entorno Linux "real" (VM, no contenedor), configuración de modo ARCHIVELOG, backups con RMAN, simulación de pérdida de datos y recuperación (restore + recover).

## Objetivo

Practicar y documentar el ciclo completo de **backup y recovery** que un DBA de Oracle ejecuta en un incidente real de pérdida de datos, usando RMAN (Recovery Manager), la herramienta oficial de Oracle para este propósito.

## Entorno

| Componente | Detalle |
|---|---|
| Virtualización | VMware Workstation 17 Player |
| Sistema operativo | Oracle Linux 8.10 (64-bit) |
| Motor de base de datos | Oracle Database Express Edition (XE) 21c |
| Arquitectura | Multitenant — CDB (`XE`) + PDB (`XEPDB1`) |
| Recursos VM | 2 vCPU, 3 GB RAM, 40 GB disco |
| Modo de log | ARCHIVELOG |

Se eligió una instalación de Oracle XE "tradicional" sobre una VM Linux, en lugar de Docker o Oracle Autonomous Database, precisamente porque este proyecto requiere acceso completo al sistema operativo para ejecutar tareas de DBA reales (RMAN, gestión de archivos físicos, backups a nivel de sistema operativo) — algo que Autonomous Database no permite, ya que es un servicio totalmente gestionado sin acceso al SO.

## Paso 1 — Preparación del entorno de prueba

Se creó un tablespace, usuario y tabla dedicados exclusivamente a esta simulación, para no arriesgar ningún dato del sistema:

```sql
ALTER SESSION SET CONTAINER = XEPDB1;

CREATE TABLESPACE test_tbs
DATAFILE '/opt/oracle/oradata/XE/XEPDB1/test_tbs01.dbf' SIZE 50M
AUTOEXTEND ON NEXT 10M MAXSIZE 200M;

CREATE USER test_user IDENTIFIED BY test123
DEFAULT TABLESPACE test_tbs
QUOTA UNLIMITED ON test_tbs;

GRANT CONNECT, RESOURCE TO test_user;
```

Conectado como `test_user`, se creó una tabla con datos de prueba:

```sql
CREATE TABLE empleados (
  id NUMBER PRIMARY KEY,
  nombre VARCHAR2(50)
);

INSERT INTO empleados VALUES (1, 'Ricardo');
INSERT INTO empleados VALUES (2, 'Prueba RMAN');
COMMIT;
```

<img width="798" height="339" alt="paso1_tablespace_y_tabla_creados" src="https://github.com/user-attachments/assets/5e3da98b-aae8-408a-9137-825da9797b90" />


## Paso 2 — Backup completo con RMAN

Con la base ya en modo ARCHIVELOG, se ejecutó un backup completo (datafiles + archivelogs) usando RMAN:

```bash
rman target /
```

```sql
BACKUP DATABASE PLUS ARCHIVELOG;
```

RMAN generó automáticamente, además del backup de datafiles y archivelogs, un **autobackup del control file y del SPFILE** — piezas críticas para cualquier recuperación posterior.

<img width="676" height="525" alt="paso2_backup_rman_finalizado" src="https://github.com/user-attachments/assets/ca23f721-30f5-406d-bbed-0bcc95dc7851" />


## Paso 3 — Simulación de pérdida de datos

Para simular una falla real (ej. disco dañado, borrado accidental), se llevó el tablespace de prueba offline y se eliminó físicamente su datafile a nivel de sistema operativo:

```sql
ALTER TABLESPACE test_tbs OFFLINE;
```

```bash
sudo rm -f /opt/oracle/oradata/XE/XEPDB1/test_tbs01.dbf
```

## Paso 4 — Evidencia del incidente

Al intentar poner el tablespace online de nuevo, Oracle confirmó la pérdida del archivo con el error esperado:

```sql
ALTER TABLESPACE test_tbs ONLINE;
```

```
ORA-01157: cannot identify/lock data file 13 - see DBWR trace file
ORA-01110: data file 13: '/opt/oracle/oradata/XE/XEPDB1/test_tbs01.dbf'
```

<img width="787" height="541" alt="paso4_error_ORA-01157" src="https://github.com/user-attachments/assets/5d256e39-8f1f-4d77-a960-db6328ef80b9" />


## Paso 5 — Restore + Recover con RMAN

Se restauró el datafile perdido desde el backup, y se aplicaron los archivelogs generados después del backup para recuperar los datos hasta el último `COMMIT` (recovery point-in-time, no solo hasta el momento del backup):

```sql
RESTORE TABLESPACE "XEPDB1":"TEST_TBS";
RECOVER TABLESPACE "XEPDB1":"TEST_TBS";
```

```sql
ALTER SESSION SET CONTAINER = XEPDB1;
ALTER TABLESPACE test_tbs ONLINE;
```

<img width="838" height="637" alt="paso5a_restore_completado" src="https://github.com/user-attachments/assets/ce1d2b27-ce0b-45d6-bdd5-5c7f6482e3cb" />
<img width="872" height="576" alt="paso5b_recover_finalizado" src="https://github.com/user-attachments/assets/67c91dcc-e180-43b3-bceb-1107d607675e" />
<img width="849" height="525" alt="paso5c_tablespace_online" src="https://github.com/user-attachments/assets/4d9cdd2d-c36a-4bc2-9a8c-b8376b2bb3af" />



> **Nota técnica:** al tratarse de una base multitenant (CDB + PDB), los comandos de RMAN sobre un tablespace de un PDB requieren la sintaxis calificada `"NOMBRE_PDB":"NOMBRE_TABLESPACE"` — sin esto, RMAN busca el objeto en el CDB raíz y no lo encuentra.

## Paso 6 — Verificación de integridad

Tras el recovery, se confirmó que los datos originales seguían intactos:

```sql
CONNECT test_user/test123@localhost:1521/XEPDB1;
SELECT * FROM empleados;
```

```
ID  NOMBRE
--- -----------
1   Ricardo
2   Prueba RMAN
```

<img width="745" height="510" alt="paso6_verificacion_final" src="https://github.com/user-attachments/assets/24339caa-76e8-4638-8c1d-cc1dd8041263" />


## Resultado

Se completó exitosamente el ciclo completo de un incidente de DBA:

**Backup → Pérdida simulada → Diagnóstico del error → Restore → Recover → Verificación**

Los datos se recuperaron sin pérdida, confirmando que la estrategia de backup (modo ARCHIVELOG + `BACKUP DATABASE PLUS ARCHIVELOG`) es efectiva para una recuperación point-in-time completa.

## Aprendizajes técnicos adicionales

Durante el proyecto se resolvieron además varios problemas reales de administración de sistema operativo, comunes en cualquier entorno Oracle sobre Linux:

- Gestión de grupos del sistema (`dba`, `oinstall`) requeridos para autenticación `/ as sysdba` y para operación de RMAN
- Arranque manual de la instancia (`STARTUP`) y del listener (`lsnrctl start`) tras cada reinicio del sistema, al no estar configurados como servicios automáticos
- Resolución de conflictos de segmentos de memoria compartida (`ipcs -m` / `ipcrm`) entre reinicios
- Corrección de un protocolo IPC mal configurado en `listener.ora` que impedía el arranque del listener
- Ajuste de permisos en el árbol de directorios de Oracle (`/opt/oracle`)

## Próximos pasos del proyecto

- Configurar la instancia y el listener como servicios de arranque automático (`systemd`)
- Documentar un escenario de recovery a nivel de base de datos completa (no solo tablespace)
- Explorar tuning básico y gestión de roles/seguridad como siguientes fases del portafolio DBA
