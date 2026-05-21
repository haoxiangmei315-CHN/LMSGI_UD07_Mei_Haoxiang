# 📦 LMSGI_UD07 — Explotación Tecnológica ERP/CRM · WillmanTech S.L.

> Actividad de Evaluación Específica (AEE) — Unidad 7  
> Ciclo DAM/DAW · Sistemas de Gestión Empresarial  
> Centro FP Superior — Campus Cámara de Comercio Sevilla

-----

## ¿De qué va este proyecto?

Este repositorio recoge los tres entregables de la AEE de la UD07. La idea es simular que somos el equipo de desarrollo de **WillmanTech S.L.**, una empresa sevillana que acaba de montar su ERP (basado en Odoo 17) y nos pide que lo dejemos listo para producción.

En concreto había que hacer tres cosas:

1. **Crear una plantilla de factura personalizada** usando el sistema de plantillas QWeb de Odoo.
1. **Preparar los archivos de exportación de datos** para que las facturas puedan viajar a otros sistemas (Agencia Tributaria, contabilidades externas…).
1. **Escribir el manual técnico** para que cualquier administrador o usuario avanzado sepa cómo funciona, cómo instalarlo y cómo mantenerlo.

-----

## Estructura del repositorio

```
📁 LMSGI_UD07_Apellido1_Apellido2_Nombre/
│
├── 📄 report_invoice_willmantech.xml   ← Entregable 1: Plantilla QWeb
│
├── 📁 interoperabilidad/
│   ├── 📄 invoice_export.json          ← Entregable 2a: Exportación JSON
│   └── 📄 invoice_ubl.xml              ← Entregable 2b: Factura UBL/PEPPOL
│
└── 📄 README.md                        ← Este archivo (= Manual de Explotación)
```

-----

## Entregable 1 — Plantilla QWeb (`report_invoice_willmantech.xml`)

Este archivo es la plantilla que Odoo usa para generar el PDF de cada factura. Está escrito en XML con directivas especiales de QWeb (el motor de plantillas de Odoo).

Lo más importante del archivo:

- **`t-foreach`** sobre `invoice_line_ids`: recorre todas las líneas de la factura y pinta una fila por cada producto o servicio.
- **`t-if` con `hay_descuento`**: antes de dibujar la tabla, se calcula si alguna línea tiene descuento. Si no hay ninguno, la columna entera desaparece, tanto del encabezado como de las filas. Así la factura queda limpia.
- **`t-field`** en los campos clave: `doc.name` (número de factura), `doc.date` (fecha), `doc.amount_total` (total a pagar). Usar `t-field` en lugar de poner el dato a mano hace que Odoo lo formatee correctamente según el idioma y la moneda configurados.

> El archivo también incluye al final el bloque `<report>` que registra la plantilla en Odoo y la vincula al botón “Imprimir” de las facturas.

-----

## Entregable 2 — Interoperabilidad de datos

### `invoice_export.json`

Representa cómo Odoo exportaría una factura a un sistema externo. El JSON contiene un array con una factura de ejemplo con todos sus datos relacionales anidados: los datos del cliente dentro de la factura, y las líneas de producto con sus impuestos dentro de cada línea.

Este tipo de estructura es la que se usaría en una integración real con la Agencia Tributaria o con cualquier software de contabilidad externo.

### `invoice_ubl.xml`

Es la misma factura pero en formato **UBL 2.1**, que es el estándar internacional para facturas electrónicas. Lo importante aquí es:

- Los **namespaces `cac` y `cbc`**: sin ellos el documento no es UBL válido. `cbc` agrupa los campos simples (fechas, importes, textos) y `cac` agrupa los bloques compuestos (dirección, línea de factura, impuesto…).
- El **`CustomizationID` de PEPPOL**: es el identificador que le dice a la red europea de facturación electrónica que este documento sigue el perfil EN16931. Sin él, la factura no sería aceptada en la red PEPPOL.
- Los datos son coherentes con el JSON: misma factura, mismos importes, mismo cliente. Solo cambia el formato.

-----

## Entregable 3 — Manual de Explotación

El propio README que estás leyendo actúa como portada, pero el manual técnico completo está desarrollado más abajo (o en el archivo `manual_explotacion_willmantech.md` si se entrega por separado).

Cubre cinco secciones obligatorias según la norma **ISO/IEC/IEEE 26514:2022**:

|Sección                       |Contenido                                                                             |
|------------------------------|--------------------------------------------------------------------------------------|
|1. Introducción y Arquitectura|Módulos activos, diagrama de contenedores Docker                                      |
|2. Instalación y Reinstalación|`docker-compose.yml`, variables `.env`, arranque desde cero                           |
|3. Seguridad y Acceso         |Roles, política de contraseñas, bloqueo del panel de BBDD                             |
|4. Backup y Restauración      |Script completo, automatización con cron, pasos de restauración                       |
|5. Flujo de Facturación       |Paso a paso para el usuario + explicación del pipeline QWeb → HTML → wkhtmltopdf → PDF|

-----

## Criterios de evaluación que cubre

|Criterio                               |Peso|Qué se evalúa                                                      |
|---------------------------------------|----|-------------------------------------------------------------------|
|CE 7.g — Generación de informes        |35% |Plantilla QWeb: sintaxis, iteración, condición, data binding       |
|CE 7.h — Extracción e interoperabilidad|35% |JSON con relaciones anidadas + XML UBL válido con namespaces PEPPOL|
|CE 7.i — Documentación de explotación  |30% |Manual técnico siguiendo ISO/IEC/IEEE 26514:2022                   |

-----

## Tecnologías usadas

- **Odoo 17 Community** — ERP base del sistema
- **QWeb** — Motor de plantillas XML de Odoo para generación de informes
- **wkhtmltopdf** — Conversor HTML → PDF que usa Odoo internamente
- **PostgreSQL 15** — Base de datos relacional del ERP
- **Docker / Docker Compose** — Despliegue en contenedores
- **UBL 2.1 / PEPPOL BIS Billing 3.0** — Estándar de factura electrónica europea
- **ISO/IEC/IEEE 26514:2022** — Norma de documentación técnica para usuarios

-----

*Repositorio creado para la AEE de la UD07 · Fecha límite de entrega: 21 de mayo de 2026*
