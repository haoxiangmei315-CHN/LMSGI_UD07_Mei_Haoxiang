# LMSGI_UD07 - Explotación Tecnológica en ERP/CRM
### WillmanTech S.L. — DAM/DAW 2024/2025

---

## ¿De qué va este repositorio?

Este repositorio contiene los tres entregables de la actividad UD07 de Sistemas de Gestión Empresarial. La actividad consiste en simular que somos el equipo de desarrollo de una empresa llamada WillmanTech S.L. y tenemos que poner en producción su sistema ERP/CRM: personalizar las facturas, preparar los archivos de exportación de datos y escribir un manual para que los administradores sepan cómo usar y mantener el sistema.

---

## Estructura del repositorio

```
├── report_invoice_willmantech.xml
├── interoperabilidad/
│   ├── invoice_export.xml
│   └── invoice_ubl.xml
└── manual_explotacion_willmantech.md
```

---

## Pasos que he seguido

### Entregable 1 — Plantilla QWeb (report_invoice_willmantech.xml)

Lo primero fue entender cómo funciona QWeb, que es el motor de plantillas que usa Odoo para generar los informes en PDF. Básicamente es XML con unas etiquetas especiales que empiezan por `t-` y que permiten meter lógica dentro del documento.

Empecé viendo ejemplos de plantillas que hay en la propia instalación de Odoo para entender la estructura mínima que necesita el archivo: el `<odoo>` exterior, el `t-call` a `web.html_container` y el `t-call` a `web.external_layout` que ya mete la cabecera y el pie de página de la empresa.

Después fui añadiendo las partes una a una:
1. Primero los datos de cabecera: número de factura, fecha y datos del cliente.
2. Luego la tabla de líneas con el `t-foreach` para que se repita por cada línea de la factura.
3. Por último los totales y la forma de pago.

La parte que más me costó fue la condición del descuento, porque no es solo ocultar una celda, hay que ocultar tanto la columna de la cabecera como la celda de cada fila, y tienen que usar la misma condición para que no quede descolocada la tabla.

### Entregable 2 — Archivos de interoperabilidad

Para este entregable el enunciado pedía un JSON, pero como en clase no hemos trabajado con ese formato, lo hice en XML igualmente, que es lo que hemos visto durante el curso.

**invoice_export.xml**: Este archivo simula lo que devolvería el ERP si le hiciéramos una consulta para exportar una factura. Lo estructuré intentando que se pareciera a cómo Odoo guarda los datos internamente: la factura tiene dentro los datos del cliente y las líneas de productos como elementos anidados.

**invoice_ubl.xml**: Este fue el más complicado con diferencia. El estándar UBL tiene sus propios namespaces (`cac` para componentes agregados y `cbc` para componentes básicos) y hay que ponerlos todos en la etiqueta raíz. Además tuve que buscar cuál era el ID de personalización correcto para que sea compatible con la red PEPPOL, que es la que se usa en Europa para la factura electrónica entre empresas. Al final el `CustomizationID` correcto es:

```
urn:cen.eu:en16931:2017#compliant#urn:fdc:peppol.eu:2017:poacc:billing:3.0
```

### Entregable 3 — Manual de explotación

Para el manual seguí las secciones que pedía el enunciado e intenté escribirlo de forma que cualquier persona que llegue nueva al proyecto pueda entenderlo sin haber visto el sistema antes. Las cinco secciones obligatorias son: arquitectura, instalación, seguridad, backup y flujo de facturación.

Lo más entretenido fue explicar el pipeline de generación de PDF, porque hay que entender que Odoo no genera el PDF directamente, sino que primero convierte la plantilla XML en HTML y luego usa `wkhtmltopdf` para convertir ese HTML en el PDF final.

---

## Problemas que me encontré y cómo los resolví

**Problema 1 — La condición del descuento no funcionaba como esperaba**

Al principio puse el `t-if` solo en la celda de datos de cada línea, pero me di cuenta de que la cabecera de la columna seguía apareciendo aunque no hubiera descuentos. Tuve que poner la misma condición también en el `<th>` de la cabecera de la tabla para que las dos desaparecieran a la vez.

**Problema 2 — Los namespaces del UBL**

El archivo `invoice_ubl.xml` al principio no lo tenía bien porque había puesto los namespaces `cac:` y `cbc:` dentro de las etiquetas donde los usaba por primera vez, en vez de declararlos todos en la etiqueta raíz `<Invoice>`. Eso hacía que el XML no estuviera bien formado. Lo corregí moviendo todas las declaraciones de namespace al principio.

**Problema 3 — Entender qué campos usar en la plantilla QWeb**

Al principio no sabía exactamente cómo se llamaban los campos del modelo de facturas en Odoo. Por ejemplo, no sabía si la fecha era `doc.date` o `doc.invoice_date`. Tuve que mirar la documentación técnica de Odoo y algunos ejemplos para confirmar los nombres correctos de los campos (`invoice_date`, `invoice_line_ids`, `amount_untaxed`, etc.).

**Problema 4 — El formato de los importes en el UBL**

En el estándar UBL los importes monetarios llevan el atributo `currencyID` dentro de la propia etiqueta, no como un elemento separado. Al principio lo tenía mal puesto y el XML no seguía bien el esquema. Lo corregí poniendo `currencyID="EUR"` como atributo directamente en cada etiqueta de importe.

---

## Recursos que he consultado

- Documentación oficial de Odoo sobre QWeb: https://www.odoo.com/documentation/16.0/developer/reference/frontend/qweb.html
- Especificación PEPPOL BIS Billing 3.0: https://docs.peppol.eu/poacc/billing/3.0/
- Apuntes de clase de LMSGI — Unidad 7
- Docker Compose docs: https://docs.docker.com/compose/

---

*Repositorio creado para la entrega de la UD07 — Sistemas de Gestión Empresarial.*

