# Manual de Explotación — WillmanTech S.L. ERP/CRM

> **Documento técnico elaborado conforme al estándar ISO/IEC/IEEE 26514:2022**
> *Systems and software engineering — Design and development of information for users*

| Campo | Valor |
|---|---|
| **Versión del documento** | 1.0 |
| **Fecha de emisión** | 15/05/2024 |
| **Sistema** | WillmanTech ERP (Odoo 17 Community) |
| **Responsable técnico** | Dpto. de Desarrollo DAM/DAW |
| **Clasificación** | Uso interno — Confidencial |

---

## Tabla de contenidos

1. [Introducción y Arquitectura del Sistema](#1-introducción-y-arquitectura-del-sistema)
2. [Guía de Instalación y Reinstalación](#2-guía-de-instalación-y-reinstalación)
3. [Seguridad y Control de Acceso](#3-seguridad-y-control-de-acceso)
4. [Procedimiento de Backup y Restauración](#4-procedimiento-de-backup-y-restauración)
5. [Flujo Operativo de Facturación e Informes](#5-flujo-operativo-de-facturación-e-informes)

---

## 1. Introducción y Arquitectura del Sistema

### 1.1 Propósito de este documento

Este manual describe cómo poner en marcha, mantener y operar de forma segura el sistema ERP/CRM de WillmanTech S.L. Está dirigido a dos tipos de usuarios:

- **Administradores de sistemas**: técnicos que se encargan de instalar, configurar y mantener la plataforma.
- **Usuarios finales con perfil avanzado**: personal de contabilidad o comercial que necesita entender cómo funciona el sistema por dentro para resolver incidencias básicas.

No se asume conocimiento previo de Odoo, pero sí un nivel básico de trabajo con Linux y línea de comandos.

### 1.2 Descripción general del sistema

El ERP de WillmanTech S.L. está construido sobre **Odoo 17 Community Edition**, un sistema de gestión empresarial de código abierto. Se gestiona principalmente el ciclo de ventas y facturación, aunque el sistema está preparado para ampliar módulos en el futuro.

Los módulos actualmente activados son:

| Módulo técnico | Nombre visible en Odoo | Función principal |
|---|---|---|
| `account` | Contabilidad / Facturación | Gestión de facturas, asientos, impuestos |
| `sale` | Ventas | Presupuestos, pedidos de venta, clientes |
| `stock` | Inventario | Control de almacén y movimientos de productos |
| `contacts` | Contactos | Directorio centralizado de clientes y proveedores |
| `willmantech_invoices` | Facturas WillmanTech | Módulo personalizado con plantilla QWeb propia |

### 1.3 Arquitectura lógica: despliegue con Docker Compose

El sistema corre íntegramente en contenedores Docker. Esto significa que todos los servicios necesarios (el propio Odoo y su base de datos) se arrancan y paran de forma coordinada con un solo comando, sin instalar nada directamente en el servidor.

```
┌─────────────────────────────────────────────────────────┐
│                   Servidor Host (Ubuntu 22.04)           │
│                                                          │
│  ┌──────────────────────┐   ┌────────────────────────┐  │
│  │  Contenedor: web     │   │  Contenedor: db         │  │
│  │  Imagen: odoo:17     │   │  Imagen: postgres:15    │  │
│  │  Puerto: 8069 (HTTP) │◄─►│  Puerto interno: 5432   │  │
│  │  Puerto: 8072 (WS)   │   │  Volumen: pgdata        │  │
│  └──────────────────────┘   └────────────────────────┘  │
│            │                                             │
│            ▼                                             │
│  Volumen Docker: odoo-data (filestore + adjuntos)        │
└─────────────────────────────────────────────────────────┘
         ▲
         │  Acceso externo
    Puerto 80/443 (Nginx inverso recomendado en producción)
```

La comunicación entre el contenedor `web` (Odoo) y el contenedor `db` (PostgreSQL) es interna a la red Docker; desde fuera solo es accesible el puerto `8069`.

---

## 2. Guía de Instalación y Reinstalación

### 2.1 Requisitos previos del sistema

Antes de empezar, el servidor debe tener instalado:

- **Docker Engine** ≥ 24.x
- **Docker Compose** ≥ 2.x (plugin integrado en Docker moderno)
- Al menos **4 GB de RAM** y **20 GB de disco libre**
- Sistema operativo: Ubuntu 22.04 LTS (recomendado) o Debian 12

Para verificar que Docker está disponible:

```bash
docker --version
docker compose version
```

### 2.2 Estructura de directorios del proyecto

```
/opt/willmantech-erp/
├── docker-compose.yml          # Definición de los servicios
├── .env                        # Variables de entorno (¡no subir a Git!)
├── config/
│   └── odoo.conf               # Configuración de Odoo
├── addons/
│   └── willmantech_invoices/   # Módulo personalizado de facturas
└── backups/                    # Carpeta local para copias de seguridad
```

### 2.3 Archivo `docker-compose.yml`

Este archivo le dice a Docker qué contenedores levantar y cómo conectarlos:

```yaml
version: '3.8'

services:

  # ── Servicio de base de datos ──────────────────────────────────
  db:
    image: postgres:15
    container_name: willmantech_db
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      # Volumen persistente: aquí viven los datos aunque el contenedor se borre
      - pgdata:/var/lib/postgresql/data
    networks:
      - odoo_net

  # ── Servicio de Odoo ───────────────────────────────────────────
  web:
    image: odoo:17
    container_name: willmantech_odoo
    restart: unless-stopped
    depends_on:
      - db
    ports:
      # Puerto externo 8069 → puerto interno 8069 de Odoo
      - "8069:8069"
    environment:
      HOST: db
      PORT: 5432
      USER: ${POSTGRES_USER}
      PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - odoo-data:/var/lib/odoo
      - ./config:/etc/odoo
      - ./addons:/mnt/extra-addons
    networks:
      - odoo_net

# ── Volúmenes persistentes ─────────────────────────────────────
volumes:
  pgdata:
  odoo-data:

# ── Red interna entre contenedores ────────────────────────────
networks:
  odoo_net:
    driver: bridge
```

### 2.4 Archivo `.env` — Variables de entorno

Crea este archivo en `/opt/willmantech-erp/.env`. **Nunca lo compartas ni lo incluyas en el repositorio de código.**

```dotenv
# Credenciales de PostgreSQL
POSTGRES_DB=willmantech_db
POSTGRES_USER=odoo_user
POSTGRES_PASSWORD=C4mb14M3_P0r_F4v0r!

# Contraseña maestra de Odoo (para gestión de bases de datos desde la web)
ODOO_MASTER_PASSWORD=W1llm4nT3ch_M4st3r!
```

> ⚠️ **Importante**: cambia siempre estas contraseñas antes de arrancar en producción. Nunca uses contraseñas por defecto.

### 2.5 Archivo `config/odoo.conf`

```ini
[options]
; Contraseña maestra de administración de bases de datos
admin_passwd = W1llm4nT3ch_M4st3r!

; Conexión a la base de datos
db_host = db
db_port = 5432
db_user = odoo_user
db_password = C4mb14M3_P0r_F4v0r!
db_name = willmantech_db

; Ruta donde se guardan ficheros adjuntos (facturas en PDF, imágenes, etc.)
data_dir = /var/lib/odoo

; Carpeta con módulos personalizados
addons_path = /mnt/extra-addons,/usr/lib/python3/dist-packages/odoo/addons

; Nivel de log (info en producción, debug solo para depurar)
log_level = info

; Workers (procesos paralelos): 0 = modo desarrollo, >0 = producción
workers = 2
```

### 2.6 Arrancar el sistema desde cero

```bash
# 1. Ir a la carpeta del proyecto
cd /opt/willmantech-erp

# 2. Arrancar todos los contenedores en segundo plano (-d = detached)
docker compose up -d

# 3. Verificar que ambos contenedores están en marcha
docker compose ps

# 4. Ver los logs en tiempo real si algo falla
docker compose logs -f web
```

Si todo va bien, verás en el navegador la pantalla de inicio de Odoo en:
`http://IP_DEL_SERVIDOR:8069`

### 2.7 Parar y reiniciar el sistema

```bash
# Parar sin borrar datos
docker compose down

# Parar Y borrar todos los volúmenes (¡DESTRUYE TODOS LOS DATOS!)
# Usar solo para reinstalación limpia o entorno de pruebas
docker compose down -v
```

### 2.8 Instalar o actualizar el módulo personalizado

Cuando se modifica el código del módulo `willmantech_invoices`, hay que indicarle a Odoo que recargue ese módulo:

```bash
# Actualizar el módulo sin perder datos
docker compose run --rm web odoo \
    -d willmantech_db \
    -u willmantech_invoices \
    --stop-after-init
```

---

## 3. Seguridad y Control de Acceso

### 3.1 Modelo de roles en Odoo

Odoo usa un sistema de grupos de usuario para controlar quién puede hacer qué. En WillmanTech S.L. se han configurado tres perfiles principales:

| Rol | Nombre en Odoo | Acceso principal |
|---|---|---|
| **Administrador** | `Configuración / Administrador` | Acceso total: configuración, usuarios, módulos |
| **Contable** | `Facturación / Contable` | Crear/modificar/validar facturas, ver informes |
| **Comercial** | `Ventas / Usuario` | Crear presupuestos y pedidos, ver clientes |

> Un usuario puede tener varios roles asignados a la vez. Por ejemplo, un jefe de ventas podría tener el rol comercial y además acceso de lectura a facturación.

### 3.2 Cómo crear un usuario y asignarle un rol

1. Ir a **Ajustes → Usuarios y empresas → Usuarios**.
2. Hacer clic en **Nuevo**.
3. Rellenar nombre, email y asignar el perfil correspondiente en la sección de permisos.
4. Guardar y enviar la invitación por email.

### 3.3 Política de contraseñas

Para activar las restricciones de contraseña, ve a **Ajustes → Permisos → Política de contraseñas**:

- Longitud mínima: **10 caracteres**
- Debe contener mayúsculas, minúsculas, números y un carácter especial
- Caducidad de contraseña: **90 días**
- Bloqueo tras **5 intentos fallidos**

### 3.4 Restricción de acceso a la gestión de bases de datos

El panel de gestión de bases de datos de Odoo (`/web/database/manager`) debe bloquearse en producción. Para ello, añade esta línea a `odoo.conf`:

```ini
; Desactiva el gestor de BBDD desde el navegador (seguridad en producción)
list_db = False
```

### 3.5 Acceso SSH al servidor

- Deshabilitar el acceso como `root` directamente via SSH.
- Usar autenticación por clave pública (no contraseña).
- Restringir el acceso al puerto 22 solo desde IPs conocidas mediante firewall (`ufw` en Ubuntu).

```bash
# Ejemplo: permitir SSH solo desde la IP de la oficina
sudo ufw allow from 80.80.80.80 to any port 22
sudo ufw enable
```

---

## 4. Procedimiento de Backup y Restauración

### 4.1 ¿Qué hay que respaldar?

Un sistema Odoo tiene dos partes que deben copiarse juntas:

1. **Base de datos PostgreSQL**: contiene todos los datos (facturas, clientes, configuración...).
2. **Filestore** (almacén de archivos): contiene los archivos adjuntos, PDFs generados, imágenes de productos... Está en el volumen `odoo-data`, dentro de `/var/lib/odoo/filestore/`.

Si solo copias la base de datos y no el filestore, perderás los archivos adjuntos.

### 4.2 Script de backup completo

Guarda este script como `/opt/willmantech-erp/backup.sh` y dale permisos de ejecución (`chmod +x backup.sh`):

```bash
#!/bin/bash
# ─────────────────────────────────────────────────────
# backup.sh — Copia de seguridad completa WillmanTech ERP
# ─────────────────────────────────────────────────────

# Variables de configuración
FECHA=$(date +%Y%m%d_%H%M%S)
DIR_BACKUP="/opt/willmantech-erp/backups"
NOMBRE_DB="willmantech_db"
CONTENEDOR_DB="willmantech_db"
USUARIO_DB="odoo_user"

# Crear directorio de backups si no existe
mkdir -p "$DIR_BACKUP"

echo ">>> [1/3] Iniciando backup de base de datos..."
# pg_dump vuelca la base de datos en formato comprimido (-Fc)
docker exec "$CONTENEDOR_DB" \
    pg_dump -U "$USUARIO_DB" -Fc "$NOMBRE_DB" \
    > "$DIR_BACKUP/db_${FECHA}.dump"

echo ">>> [2/3] Iniciando backup del filestore..."
# Copiamos el volumen de datos de Odoo (adjuntos y ficheros)
docker run --rm \
    --volumes-from willmantech_odoo \
    -v "$DIR_BACKUP":/backup \
    ubuntu \
    tar czf /backup/filestore_${FECHA}.tar.gz /var/lib/odoo/filestore

echo ">>> [3/3] Backup completado."
echo "    Base de datos : $DIR_BACKUP/db_${FECHA}.dump"
echo "    Filestore     : $DIR_BACKUP/filestore_${FECHA}.tar.gz"

# Opcional: borrar backups con más de 30 días para no llenar el disco
find "$DIR_BACKUP" -name "*.dump" -mtime +30 -delete
find "$DIR_BACKUP" -name "*.tar.gz" -mtime +30 -delete
```

### 4.3 Automatizar el backup con cron

Para que el backup se ejecute solo cada noche a las 2:00 AM:

```bash
# Abrir el editor de cron del usuario root
sudo crontab -e

# Añadir esta línea al final del archivo:
0 2 * * * /opt/willmantech-erp/backup.sh >> /var/log/willmantech_backup.log 2>&1
```

### 4.4 Procedimiento de restauración

En caso de pérdida de datos o migración a otro servidor:

```bash
# PASO 1: Parar Odoo (no la base de datos)
docker compose stop web

# PASO 2: Borrar la base de datos existente y crear una nueva vacía
docker exec willmantech_db \
    dropdb -U odoo_user willmantech_db

docker exec willmantech_db \
    createdb -U odoo_user willmantech_db

# PASO 3: Restaurar el volcado de base de datos
# Sustituye FECHA por la fecha del backup que quieres restaurar
docker exec -i willmantech_db \
    pg_restore -U odoo_user -d willmantech_db \
    < /opt/willmantech-erp/backups/db_FECHA.dump

# PASO 4: Restaurar el filestore
docker run --rm \
    --volumes-from willmantech_odoo \
    -v /opt/willmantech-erp/backups:/backup \
    ubuntu \
    tar xzf /backup/filestore_FECHA.tar.gz -C /

# PASO 5: Volver a arrancar Odoo
docker compose start web
```

---

## 5. Flujo Operativo de Facturación e Informes

### 5.1 Cómo generar una factura paso a paso

Esta sección explica el proceso completo desde que un comercial crea un pedido hasta que el cliente recibe su factura en PDF.

**Paso 1 — Confirmar un pedido de venta**

1. Ve al menú **Ventas → Pedidos → Pedidos de venta**.
2. Abre el pedido correspondiente o crea uno nuevo con **Nuevo**.
3. Añade las líneas de producto, cantidades y precios.
4. Haz clic en **Confirmar**. El pedido pasa a estado *Pedido de venta*.

**Paso 2 — Crear la factura desde el pedido**

1. Dentro del pedido confirmado, haz clic en el botón **Crear factura** (esquina superior derecha).
2. Se abrirá un diálogo. Elige **Factura normal** y pulsa **Crear y ver factura**.
3. Odoo genera automáticamente la factura con los datos del pedido.

**Paso 3 — Revisar y validar la factura**

1. Revisa que los datos del cliente, líneas, precios e impuestos son correctos.
2. Si hay un descuento acordado, introdúcelo en la columna *Descuento (%)* de cada línea.
3. Haz clic en **Confirmar**. La factura cambia de estado *Borrador* a *Publicado* y se le asigna el número definitivo (ej: `FACT/2024/0042`).

**Paso 4 — Generar el PDF de la factura**

1. Con la factura en estado *Publicado*, haz clic en **Imprimir → Factura**.
2. Odoo genera el PDF y lo descarga automáticamente.

### 5.2 Cómo funciona la generación del PDF por dentro

Entender este proceso es útil para depurar problemas cuando el PDF no se genera correctamente.

```
┌──────────────────────────────────────────────────────────────────┐
│         PIPELINE DE GENERACIÓN DE PDF EN ODOO                    │
│                                                                  │
│  1. El usuario pulsa "Imprimir"                                  │
│          │                                                       │
│          ▼                                                       │
│  2. Odoo carga la plantilla QWeb XML                             │
│     (report_invoice_willmantech.xml)                             │
│     El motor QWeb sustituye todas las directivas t-field,        │
│     t-foreach y t-if con los datos reales de la factura,         │
│     generando un documento HTML completo y válido.               │
│          │                                                       │
│          ▼                                                       │
│  3. Odoo pasa ese HTML al conversor wkhtmltopdf                  │
│     wkhtmltopdf es una herramienta de línea de comandos que      │
│     funciona como un navegador web sin pantalla (headless):      │
│     "renderiza" el HTML exactamente como lo haría Chrome,        │
│     respetando el CSS, y captura el resultado como PDF.          │
│          │                                                       │
│          ▼                                                       │
│  4. El PDF resultante se devuelve al navegador del usuario       │
│     y también se guarda en el filestore de Odoo como adjunto     │
│     de la factura.                                               │
└──────────────────────────────────────────────────────────────────┘
```

En resumen: **QWeb convierte los datos en HTML → wkhtmltopdf convierte ese HTML en PDF**.

### 5.3 Diagnóstico de problemas frecuentes en los PDFs

| Síntoma | Causa probable | Solución |
|---|---|---|
| El PDF se descarga en blanco | wkhtmltopdf no está instalado en el contenedor | Verificar con `docker exec willmantech_odoo which wkhtmltopdf` |
| El PDF muestra `[object Object]` en lugar de datos | Error en una directiva `t-field` de la plantilla XML | Revisar el nombre del campo en el XML |
| La columna "Descuento" aparece aunque no hay descuentos | La condición `t-if` no está bien escrita | Verificar la expresión Python en el atributo `t-if` |
| Tablas descolocadas o textos cortados | Problema de CSS en la plantilla QWeb | Ajustar estilos en el XML o en la hoja de estilos del módulo |

### 5.4 Ver el historial de facturas generadas

- Ve a **Contabilidad → Clientes → Facturas**.
- Usa los filtros para buscar por cliente, fecha, estado o número de factura.
- Puedes descargar cualquier factura anterior en PDF desde el botón **Imprimir** dentro de cada registro.

---

*Fin del Manual de Explotación v1.0 — WillmanTech S.L.*

*Elaborado conforme al estándar ISO/IEC/IEEE 26514:2022 — Design and development of information for users.*
