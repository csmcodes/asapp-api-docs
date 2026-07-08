# Asapp Electronic — Guía de Integración API Externa

> Versión: 1.1 · Última actualización: 2026-07-08

## Ambientes

| Ambiente | Base URL | Campo `ambiente` en requests |
|----------|----------|-------------------------------|
| **Desarrollo** | `https://electronic-api-dev.asapp.com.ec/api` | `"Pruebas"` |
| **Producción** | `https://electronic-api.asapp.com.ec/api` | `"Produccion"` |

> **Importante:** El campo `ambiente` en cada request debe coincidir con el entorno donde estás operando. Los comprobantes emitidos en `"Pruebas"` no tienen validez tributaria.

Esta guía está dirigida a desarrolladores de sistemas externos (ERPs, sistemas contables, puntos de venta) que necesitan emitir comprobantes electrónicos SRI Ecuador a través de la API de Asapp.

---

## Índice

1. [Autenticación](#1-autenticación)
2. [Flujo de emisión](#2-flujo-de-emisión)
3. [Endpoint JSON — crear comprobante](#3-endpoint-json--crear-comprobante)
4. [Endpoint XML — registrar XML pre-generado](#4-endpoint-xml--registrar-xml-pre-generado)
5. [Consultar estado](#5-consultar-estado)
6. [Ejemplos por tipo de comprobante](#6-ejemplos-por-tipo-de-comprobante)
7. [Referencia de campos](#7-referencia-de-campos)
8. [Manejo de errores](#8-manejo-de-errores)
9. [Procesamiento en lote (batch)](#9-procesamiento-en-lote-batch)

---

## 1. Autenticación

Todos los endpoints de la API externa se autentican con una **API key** generada desde el portal Asapp (**Configuración → Empresa → Integración API**).

### Cómo obtener una API key

1. Accede al portal Asapp con tu usuario administrador.
2. Ve a **Configuración → Empresa → Integración API**.
3. Haz clic en **Generar key**, asígnale un nombre descriptivo (ej.: `"ERP ContaPlus"`).
4. **Copia la key completa** — se muestra una única vez y no puede recuperarse.

Las keys tienen el formato `asapp_` seguido de 38 caracteres alfanuméricos:

```
asapp_A3bK9mNpQr2sVwXyZ7cDeFgHjLuTi4oE
```

### Uso en requests

Envía la key en el header `X-Api-Key` en cada request:

```http
POST /v1/api/comprobantes
X-Api-Key: asapp_A3bK9mNpQr2sVwXyZ7cDeFgHjLuTi4oE
Content-Type: application/json
```

Si la key es inválida o fue revocada, recibirás `401 Unauthorized`.

---

## 2. Flujo de emisión

La API es **asíncrona**. El proceso completo tiene tres pasos:

```
1. Tu sistema envía el comprobante
        ↓
   API responde con comprobanteId (202 Accepted)
        ↓
2. Asapp procesa en background:
   firma XML → envía al SRI → espera autorización → genera RIDE → envía email
        ↓
3. Tu sistema consulta el estado con comprobanteId
   hasta recibir "Finalizado" (autorizado) o "RequiereAccion" (error SRI)
```

**No bloquees tu proceso esperando la autorización SRI.** El tiempo promedio desde envío hasta `Autorizado` es 5–30 segundos, pero puede demorar más si el SRI está congestionado. Implementa un polling con backoff (ver sección 5).

---

## 3. Endpoint JSON — crear comprobante

**Acepta cualquier tipo de comprobante en un único endpoint.** El campo `tipoDocumento` actúa como discriminador.

```
POST /v1/api/comprobantes
```

### Secuenciales

Tienes dos opciones:

| Opción | `secuencial` | Comportamiento |
|--------|-------------|----------------|
| **Asapp gestiona** | `null` (omitir) | Asapp asigna el siguiente número atómico por `establecimiento + puntoEmision + tipoDocumento` |
| **Tu sistema gestiona** | número entero | Asapp usa el valor que envías; tú eres responsable de evitar duplicados |

**Recomendación:** usa `secuencial: null` a menos que necesites control explícito de la numeración.

### Establecimiento y punto de emisión

Envía los **códigos SRI** (`"001"`, `"002"`...), no IDs internos. Estos deben existir configurados en tu empresa dentro de Asapp.

### Respuesta exitosa

```json
HTTP 202 Accepted

{
  "comprobanteId": 1847,
  "claveAcceso": "0805202601179240190100110010010000001841234567818",
  "numeroComprobante": "001-001-000000184",
  "estado": "Generado",
  "error": null
}
```

---

## 4. Endpoint XML — registrar XML pre-generado

Si tu sistema ya genera el XML SRI compliant, puedes enviarlo directamente. Asapp lo firma, envía al SRI y gestiona el ciclo completo.

```
POST /v1/api/xml
```

```json
{
  "tipoDocumento": "01",
  "ambiente": "Produccion",
  "xmlBase64": "<XML completo codificado en Base64>",
  "emailCliente": "cliente@empresa.com"
}
```

| Campo | Tipo | Req. | Descripción |
|-------|------|------|-------------|
| `tipoDocumento` | string | ✅ | `"01"` Factura · `"03"` LC · `"04"` NC · `"05"` ND · `"06"` GR · `"07"` Retención |
| `ambiente` | string | ✅ | `"Pruebas"` o `"Produccion"` |
| `xmlBase64` | string | ✅ | XML SRI completo codificado en Base64. Debe ser válido según XSD v1.1.0. |
| `emailCliente` | string | — | Email(s) destino para enviar el RIDE una vez autorizado. Acepta múltiples direcciones separadas por `,`, `;` o `\|`. Si se omite, Asapp usa el email declarado en el XML (`<emailCliente>` en `<infoFactura>`). |

> **Nota:** El email solo se envía si el SRI autoriza el comprobante. No se envía para comprobantes rechazados o con error.

---

## 5. Consultar estado

```
GET /v1/comprobantes/{comprobanteId}/estado
X-Api-Key: asapp_...
```

### Respuesta

```json
{
  "comprobanteId": 1847,
  "estadoSri": "Autorizado",
  "estadoGeneral": "Finalizado",
  "numeroAutorizacion": "0805202618021900110010010000001841234567818",
  "fechaAutorizacion": "2026-05-08T14:32:11-05:00",
  "mensajeSriOriginal": null
}
```

### Valores de `estadoGeneral` (campo principal para lógica de negocio)

| `estadoGeneral` | Significado | ¿Qué hacer? |
|-----------------|-------------|-------------|
| `EnProceso` | Firmando / enviando al SRI / esperando respuesta | Seguir polleando |
| `Finalizado` | **Autorizado por el SRI** ✓ | Proceso completo |
| `RequiereAccion` | SRI devolvió error o no autorizó | Revisar `mensajeSriOriginal` |
| `Anulado` | Anulado manualmente en el portal | No reintenta |

### Valores de `estadoSri` (detalle técnico)

`Generado → Firmado → Enviado → Recibido → Autorizado` (camino exitoso)  
`Devuelto` / `NoAutorizado` / `Timeout` → requieren acción

### Estrategia de polling recomendada

```
t+0s   → consulta (probable: EnProceso)
t+5s   → consulta
t+15s  → consulta
t+30s  → consulta
t+60s  → consulta
t+120s → consulta
t+300s → última consulta; si sigue EnProceso, revisar más tarde
```

Si `estadoGeneral == "RequiereAccion"`, lee `mensajeSriOriginal` para diagnosticar el error SRI. Los comprobantes `Devueltos` pueden corregirse y reenviarse desde el portal Asapp.

---

## 6. Ejemplos por tipo de comprobante

### 6.1 Factura (tipoDocumento: `"01"`)

```json
{
  "tipoDocumento": "01",
  "ambiente": "Produccion",
  "establecimiento": "001",
  "puntoEmision": "001",
  "secuencial": null,
  "fechaEmision": "2026-05-08T10:30:00-05:00",

  "tipoIdentificacionReceptor": "05",
  "identificacionCliente": "1712345678",
  "razonSocialCliente": "Juan Carlos Pérez",
  "direccionComprador": "Av. República del Salvador N34-183",
  "emailCliente": "juan.perez@email.com",

  "propina": 0,
  "moneda": "DOLAR",

  "detalles": [
    {
      "codigoPrincipal": "PROD-001",
      "descripcion": "Servicio de consultoría contable",
      "cantidad": 1,
      "precioUnitario": 200.00,
      "descuento": 0,
      "impuestos": [
        {
          "codigo": "2",
          "codigoPorcentaje": "4",
          "tarifa": 15,
          "baseImponible": 200.00,
          "valor": 30.00
        }
      ]
    },
    {
      "codigoPrincipal": "PROD-002",
      "descripcion": "Software contable licencia mensual",
      "cantidad": 3,
      "precioUnitario": 50.00,
      "descuento": 15.00,
      "impuestos": [
        {
          "codigo": "2",
          "codigoPorcentaje": "4",
          "tarifa": 15,
          "baseImponible": 135.00,
          "valor": 20.25
        }
      ]
    }
  ],

  "pagos": [
    {
      "formaPago": "01",
      "total": 385.25
    }
  ],

  "infoAdicional": [
    { "nombre": "Numero Pedido", "valor": "PO-2026-0512" }
  ]
}
```

### 6.2 Liquidación de Compra (tipoDocumento: `"03"`)

Documenta una compra a un proveedor sin RUC (persona natural informal).

```json
{
  "tipoDocumento": "03",
  "ambiente": "Produccion",
  "establecimiento": "001",
  "puntoEmision": "001",
  "secuencial": null,
  "fechaEmision": "2026-05-08T09:00:00-05:00",

  "tipoIdentificacionReceptor": "05",
  "identificacionCliente": "0912345678",
  "razonSocialCliente": "María Elena Torres",
  "direccionComprador": "Barrio La Florida, Calle Guayaquil S/N",
  "emailCliente": "maria.torres@gmail.com",

  "moneda": "DOLAR",

  "detalles": [
    {
      "codigoPrincipal": "AGRIC-001",
      "descripcion": "Maíz duro seco quintal",
      "cantidad": 50,
      "precioUnitario": 28.00,
      "descuento": 0,
      "impuestos": [
        {
          "codigo": "2",
          "codigoPorcentaje": "0",
          "tarifa": 0,
          "baseImponible": 1400.00,
          "valor": 0
        }
      ]
    }
  ],

  "pagos": [
    {
      "formaPago": "01",
      "total": 1400.00
    }
  ]
}
```

### 6.3 Nota de Crédito (tipoDocumento: `"04"`)

Modifica o anula parcialmente una factura previamente autorizada.

```json
{
  "tipoDocumento": "04",
  "ambiente": "Produccion",
  "establecimiento": "001",
  "puntoEmision": "001",
  "secuencial": null,
  "fechaEmision": "2026-05-08T11:00:00-05:00",

  "tipoIdentificacionReceptor": "05",
  "identificacionCliente": "1712345678",
  "razonSocialCliente": "Juan Carlos Pérez",
  "emailCliente": "juan.perez@email.com",

  "codDocModificado": "01",
  "numDocModificado": "0805202618021900110010010000001801234567811",
  "fechaEmisionDocSustento": "2026-05-05T00:00:00-05:00",
  "motivo": "Devolución parcial por producto defectuoso",
  "razonDevolucion": "01",

  "moneda": "DOLAR",

  "detalles": [
    {
      "codigoPrincipal": "PROD-002",
      "descripcion": "Software contable licencia mensual — devolución",
      "cantidad": 1,
      "precioUnitario": 50.00,
      "descuento": 0,
      "impuestos": [
        {
          "codigo": "2",
          "codigoPorcentaje": "4",
          "tarifa": 15,
          "baseImponible": 50.00,
          "valor": 7.50
        }
      ]
    }
  ]
}
```

### 6.4 Nota de Débito (tipoDocumento: `"05"`)

Carga adicional sobre una factura existente (intereses, gastos, diferencias de precio).

```json
{
  "tipoDocumento": "05",
  "ambiente": "Produccion",
  "establecimiento": "001",
  "puntoEmision": "001",
  "secuencial": null,
  "fechaEmision": "2026-05-08T12:00:00-05:00",

  "tipoIdentificacionReceptor": "04",
  "identificacionCliente": "1792456789001",
  "razonSocialCliente": "Empresa ABC S.A.",
  "emailCliente": "facturacion@empresaabc.com",

  "codDocModificado": "01",
  "numDocModificado": "0801202618021900110010010000001651234567812",
  "fechaEmisionDocSustento": "2026-05-01T00:00:00-05:00",

  "moneda": "DOLAR",

  "detalles": [
    {
      "codigoPrincipal": "INT-001",
      "descripcion": "Interés por mora — 30 días",
      "cantidad": 1,
      "precioUnitario": 45.00,
      "descuento": 0,
      "impuestos": [
        {
          "codigo": "2",
          "codigoPorcentaje": "4",
          "tarifa": 15,
          "baseImponible": 45.00,
          "valor": 6.75
        }
      ]
    }
  ],

  "pagos": [
    {
      "formaPago": "16",
      "total": 51.75,
      "plazo": 15,
      "unidadTiempo": "dias"
    }
  ]
}
```

### 6.5 Guía de Remisión (tipoDocumento: `"06"`)

Ampara el traslado de mercadería.

```json
{
  "tipoDocumento": "06",
  "ambiente": "Produccion",
  "establecimiento": "001",
  "puntoEmision": "001",
  "secuencial": null,
  "fechaEmision": "2026-05-08T08:00:00-05:00",

  "tipoIdentificacionDestinatario": "04",
  "identificacionDestinatario": "1792456789001",
  "razonSocialDestinatario": "Empresa ABC S.A.",
  "dirDestinatario": "Parque Industrial, Bloque C Local 12, Quito",
  "emailDestinatario": "logistica@empresaabc.com",

  "dirPartida": "Bodega Central, Av. Eloy Alfaro N36-92, Quito",
  "motivoTraslado": "Traslado a cliente",
  "fechaIniTransporte": "2026-05-08T07:00:00-05:00",
  "fechaFinTransporte": "2026-05-08T18:00:00-05:00",

  "tipoIdentificacionTransportista": "04",
  "rucTransportista": "1791234567001",
  "razonSocialTransportista": "Transporte Rápido S.A.",
  "placa": "PBZ-1234",

  "codDocSustentoGr": "01",
  "numDocSustentoGr": "001-001-000000165",
  "numAutDocSustentoGr": "0801202618021900110010010000001651234567812",
  "fechaEmisionDocSustentoGr": "2026-05-08T00:00:00-05:00",

  "detalles": [
    {
      "codigoPrincipal": "PROD-001",
      "descripcion": "Servidor rack 2U",
      "cantidad": 2,
      "precioUnitario": 0,
      "descuento": 0,
      "impuestos": []
    }
  ],

  "infoAdicional": [
    { "nombre": "Orden de entrega", "valor": "OE-2026-0387" }
  ]
}
```

> **Nota:** En la Guía de Remisión los `detalles` describen las mercaderías transportadas. El `precioUnitario` puede ser 0 cuando no aplica valoración (la GR no es un documento de cobro).

### 6.6 Comprobante de Retención (tipoDocumento: `"07"`)

```json
{
  "tipoDocumento": "07",
  "ambiente": "Produccion",
  "establecimiento": "001",
  "puntoEmision": "001",
  "secuencial": null,
  "fechaEmision": "2026-05-08T16:00:00-05:00",

  "periodoFiscalMes": 5,
  "periodoFiscalAnio": 2026,

  "tipoIdSujetoRetenido": "04",
  "idSujetoRetenido": "1791234560001",
  "razonSocialSujetoRetenido": "Proveedor Tech S.A.",
  "emailSujetoRetenido": "conta@proveedortech.com",

  "docsSustento": [
    {
      "codSustento": "01",
      "codDocSustento": "01",
      "numDocSustento": "001-001-000000200",
      "fechaEmisionDocSustento": "2026-05-07T00:00:00-05:00",
      "numAutDocSustento": "0705202618021791234560001001001000000200123456781",
      "formaPago": "01",
      "pagoLocExt": "01",
      "retenciones": [
        {
          "codigo": "1",
          "codigoRetencion": "303",
          "baseImponible": 1000.00,
          "porcentajeRetener": 2
        },
        {
          "codigo": "2",
          "codigoRetencion": "725",
          "baseImponible": 1000.00,
          "porcentajeRetener": 10
        }
      ]
    }
  ]
}
```

---

## 7. Referencia de campos

### Campos comunes a todos los tipos

| Campo | Tipo | Req. | Descripción |
|-------|------|------|-------------|
| `tipoDocumento` | string | ✅ | `"01"` Factura · `"03"` LC · `"04"` NC · `"05"` ND · `"06"` GR · `"07"` Retención |
| `ambiente` | string | ✅ | `"Pruebas"` o `"Produccion"` |
| `establecimiento` | string(3) | ✅ | Código SRI, ej: `"001"` |
| `puntoEmision` | string(3) | ✅ | Código SRI, ej: `"001"` |
| `secuencial` | long? | — | `null` = Asapp asigna · número = tu sistema gestiona |
| `fechaEmision` | datetime | ✅ | ISO 8601 con zona horaria Ecuador (`-05:00`) |
| `infoAdicional` | array | — | Campos adicionales visibles en el RIDE |

### Tipo de identificación (`tipoIdentificacionReceptor`)

| Código | Documento |
|--------|-----------|
| `"04"` | RUC (13 dígitos) |
| `"05"` | Cédula (10 dígitos) |
| `"06"` | Pasaporte |
| `"07"` | Consumidor final |
| `"08"` | Identificación exterior |
| `"09"` | Placa |

### IVA — código de porcentaje (`codigoPorcentaje` en impuestos)

| Código | Tarifa | Código impuesto (`codigo`) |
|--------|--------|---------------------------|
| `"0"` | 0% | `"2"` |
| `"2"` | Exento de IVA | `"2"` |
| `"3"` | No objeto de IVA | `"2"` |
| `"4"` | 15% | `"2"` |
| `"6"` | 5% | `"2"` |

### Formas de pago (`formaPago`)

| Código | Descripción |
|--------|-------------|
| `"01"` | Sin utilización del sistema financiero (efectivo) |
| `"16"` | Tarjeta de débito |
| `"19"` | Tarjeta de crédito |
| `"20"` | Otros con utilización del sistema financiero |
| `"17"` | Dinero electrónico |

### Códigos documento modificado (`codDocModificado` en NC/ND)

| Código | Documento |
|--------|-----------|
| `"01"` | Factura |
| `"03"` | Liquidación de compra |
| `"04"` | Nota de crédito |
| `"05"` | Nota de débito |
| `"07"` | Comprobante de retención |

---

## 8. Manejo de errores

### Códigos HTTP

| Código | Situación |
|--------|-----------|
| `202 Accepted` | Comprobante creado, flujo SRI iniciado |
| `207 Multi-Status` | Batch con éxitos parciales (algunos ítems fallaron) |
| `400 Bad Request` | JSON inválido, campos requeridos faltantes, batch vacío o >50 ítems |
| `401 Unauthorized` | API key inválida o revocada |
| `422 Unprocessable Entity` | Todos los ítems del batch fallaron |
| `500 Internal Server Error` | Error interno |

### Estructura de error en ítem

Cuando un comprobante falla (en respuesta individual o en batch), el campo `estado` será `"Error"` y `error` contendrá el mensaje:

```json
{
  "comprobanteId": null,
  "estado": "Error",
  "error": "Establecimiento '002' no encontrado. Verifica que el establecimiento esté configurado en Asapp."
}
```

### Errores comunes del SRI (`mensajeSriOriginal`)

Cuando `estadoGeneral == "RequiereAccion"`, lee `mensajeSriOriginal` en la respuesta de estado:

| Mensaje SRI | Causa más común |
|-------------|-----------------|
| `"RUC no existe"` | El RUC del receptor no está activo en el SRI |
| `"Clave de acceso registrada"` | Ya existe un comprobante con la misma clave de acceso |
| `"Fecha de emisión no válida"` | `fechaEmision` fuera del rango permitido (±5 días) |
| `"El número de RUC del emisor no coincide"` | El certificado .p12 de la empresa no corresponde al RUC |

Los comprobantes en estado `RequiereAccion` pueden corregirse y reenviarse desde el portal Asapp. **La API externa no expone corrección de comprobantes** — eso se gestiona directamente en el portal.

---

## 9. Procesamiento en lote (batch)

Ambos endpoints aceptan un **array JSON** en el body. El procesamiento es secuencial y soporta fallos parciales.

- Máximo **50 ítems** por request.
- Si un ítem falla, los siguientes continúan procesándose.
- La respuesta incluye un objeto por cada ítem con su `index` (posición 0-based).

### Request batch

```json
[
  {
    "tipoDocumento": "01",
    "establecimiento": "001",
    ...
  },
  {
    "tipoDocumento": "01",
    "establecimiento": "001",
    ...
  }
]
```

### Respuesta batch (207 Multi-Status — éxito parcial)

```json
[
  {
    "index": 0,
    "comprobanteId": 1848,
    "claveAcceso": "...",
    "numeroComprobante": "001-001-000000185",
    "estado": "Generado",
    "error": null
  },
  {
    "index": 1,
    "comprobanteId": null,
    "claveAcceso": null,
    "numeroComprobante": null,
    "estado": "Error",
    "error": "Punto de emisión '003' no encontrado en establecimiento '001'."
  }
]
```

### Recomendaciones para batch

- Agrupa comprobantes del mismo `establecimiento + puntoEmision + tipoDocumento` si usas `secuencial: null` — el processing secuencial garantiza secuenciales consecutivos.
- Envía batches en horarios de baja concurrencia si el volumen es alto (>20 comprobantes).
- Registra los `comprobanteId` exitosos para el polling posterior — no dependers del orden del array.

---

## Soporte

Para dudas técnicas o reporte de errores de integración:  
**admin@csmcodes.com** · Asunto: `[API Externa] <descripción breve>`
