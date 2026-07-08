# Asapp Electronic — Guía de Integración API Externa

> Versión: 2.0 · Última actualización: 2026-07-08

## Ambientes

| Ambiente | Base URL | Campo `ambiente` en requests |
|----------|----------|-------------------------------|
| **Dev / QA** | `https://electronic-api-dev.asapp.com.ec/api` | `"Pruebas"` |
| **Producción** | `https://electronic-api.asapp.com.ec/api` | `"Produccion"` |

> El campo `ambiente` en el request controla si el comprobante va al SRI PRUEBAS o SRI PRODUCCIÓN — independientemente de la URL del entorno. Para producción real usar siempre URL prod + `"Produccion"`.

> **API keys:** son independientes por entorno. La key generada en el portal de dev no sirve en prod y viceversa.

Esta guía está dirigida a desarrolladores de sistemas externos (ERPs, puntos de venta, sistemas contables) que necesitan emitir comprobantes electrónicos SRI Ecuador a través de la API de Asapp.

---

## Índice

1. [Autenticación](#1-autenticación)
2. [Flujo de emisión](#2-flujo-de-emisión)
3. [Endpoint principal — JSON estructurado](#3-endpoint-principal--json-estructurado)
4. [Campos del request](#4-campos-del-request)
5. [Email — comportamiento](#5-email--comportamiento)
6. [Consultar estado](#6-consultar-estado)
7. [Batch (lote)](#7-batch-lote)
8. [Ejemplos por tipo de comprobante](#8-ejemplos-por-tipo-de-comprobante)
9. [Endpoint secundario — XML pre-generado](#9-endpoint-secundario--xml-pre-generado)
10. [Referencia de campos](#10-referencia-de-campos)
11. [Manejo de errores](#11-manejo-de-errores)

---

## 1. Autenticación

Todos los endpoints de la API externa se autentican con una **API key** generada desde el portal Asapp (**Configuración → Empresa → Integración API**).

```http
POST /v1/api/comprobantes
X-Api-Key: asapp_A3bK9mNpQr2sVwXyZ7cDeFgHjLuTi4oE
Content-Type: application/json
```

Las keys tienen el formato `asapp_` seguido de 38 caracteres alfanuméricos. Se muestran **una única vez** al crearlas. Si se pierde, hay que revocar y generar una nueva.

Respuesta si la key es inválida o fue revocada: `401 Unauthorized`.

---

## 2. Flujo de emisión

La API es **asíncrona**. El proceso completo tiene tres pasos:

```
1. Tu sistema envía el comprobante (JSON estructurado)
        ↓
   API responde con comprobanteId (202 Accepted)
        ↓
2. Asapp procesa en background:
   Genera XML (SRI v1.1.0) → Firma → Envía al SRI → Polling autorización
   → Si AUTORIZADO: genera RIDE PDF + envía email al cliente
        ↓
3. Tu sistema consulta el estado con comprobanteId
   hasta recibir "Finalizado" (autorizado) o "RequiereAccion" (error SRI)
```

Tiempo promedio hasta `Autorizado`: 5–30 segundos.

---

## 3. Endpoint principal — JSON estructurado

**Esta es la opción recomendada para todas las integraciones.**

Tu sistema envía los datos del comprobante en JSON. Asapp se encarga de todo lo demás: genera el XML según la especificación SRI v1.1.0, calcula la ClaveAcceso, firma con XAdES, valida contra XSD y gestiona el flujo con el SRI. Tu sistema nunca necesita conocer el formato XML del SRI.

```
POST /v1/api/comprobantes
```

El campo `tipoDocumento` es el discriminador — un único endpoint maneja los 6 tipos de comprobante SRI.

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

## 4. Campos del request

### Comunes a todos los tipos

| Campo | Tipo | Req. | Descripción |
|-------|------|------|-------------|
| `tipoDocumento` | string | ✅ | `"01"` FAC · `"03"` LC · `"04"` NC · `"05"` ND · `"06"` GR · `"07"` Retención |
| `ambiente` | string | ✅ | `"Pruebas"` o `"Produccion"` |
| `establecimiento` | string(3) | ✅ | Código SRI: `"001"`, `"002"`... Debe existir en Asapp |
| `puntoEmision` | string(3) | ✅ | Código SRI: `"001"`, `"002"`... |
| `secuencial` | long? | — | `null` = Asapp asigna atómicamente · número = tu sistema gestiona |
| `fechaEmision` | datetime | ✅ | ISO 8601 con zona Ecuador: `2026-07-08T10:30:00-05:00` |
| `infoAdicional` | array | — | `[{ "nombre": "...", "valor": "..." }]` — visible en el RIDE |

### Receptor — FAC (01), LC (03), NC (04), ND (05)

| Campo | Tipo | Req. | Descripción |
|-------|------|------|-------------|
| `tipoIdentificacionReceptor` | string(2) | ✅ | Ver tabla de identificación en sección 10 |
| `identificacionCliente` | string(20) | ✅ | RUC, cédula, pasaporte, etc. |
| `razonSocialCliente` | string(300) | ✅ | Nombre o razón social del cliente |
| `direccionComprador` | string(300) | — | Dirección del comprador |
| `emailCliente` | string | — | Email(s) para enviar el RIDE cuando sea autorizado. Acepta múltiples separados por `,` `;` o `\|`. Si se omite, no se envía email. |

### Productos y pagos — FAC (01), LC (03)

| Campo | Tipo | Req. | Descripción |
|-------|------|------|-------------|
| `detalles` | array | ✅ | Líneas del comprobante |
| `pagos` | array | ✅ | Formas de pago |
| `propina` | decimal | — | Default `0` |
| `moneda` | string | — | Default `"DOLAR"` |

**Estructura de cada `detalle`:**

```json
{
  "codigoPrincipal": "PROD-001",
  "descripcion": "Producto o servicio",
  "cantidad": 1,
  "precioUnitario": 100.00,
  "descuento": 0,
  "impuestos": [
    { "codigo": "2", "codigoPorcentaje": "4", "tarifa": 15, "baseImponible": 100.00, "valor": 15.00 }
  ]
}
```

### NC (04) y ND (05) — campos adicionales

| Campo | Tipo | Req. | Descripción |
|-------|------|------|-------------|
| `codDocModificado` | string(2) | ✅ | Tipo del comprobante original (`"01"` = FAC) |
| `numDocModificado` | string(17) | ✅ | ClaveAcceso (49 dígitos) del comprobante original |
| `fechaEmisionDocSustento` | datetime | ✅ | Fecha del comprobante original |
| `motivo` | string(300) | ✅ NC | Razón de la nota de crédito |

### GR (06) — destinatario y transporte

| Campo | Tipo | Req. | Descripción |
|-------|------|------|-------------|
| `tipoIdentificacionDestinatario` | string(2) | ✅ | |
| `identificacionDestinatario` | string(20) | ✅ | |
| `razonSocialDestinatario` | string(300) | ✅ | |
| `dirDestinatario` | string(300) | ✅ | Dirección de entrega |
| `emailDestinatario` | string | — | Email para notificación del RIDE |
| `dirPartida` | string(300) | ✅ | |
| `motivoTraslado` | string(300) | ✅ | |
| `fechaIniTransporte` | datetime | ✅ | |
| `fechaFinTransporte` | datetime | ✅ | |
| `rucTransportista` | string(13) | ✅ | |
| `razonSocialTransportista` | string(300) | ✅ | |
| `placa` | string(20) | ✅ | |
| `codDocSustentoGr` | string(2) | ✅ | `"01"` = factura |
| `numDocSustentoGr` | string(17) | ✅ | Número del doc. de sustento |
| `numAutDocSustentoGr` | string(49) | ✅ | ClaveAcceso del doc. de sustento |
| `fechaEmisionDocSustentoGr` | datetime | ✅ | |

### Retención (07)

| Campo | Tipo | Req. | Descripción |
|-------|------|------|-------------|
| `periodoFiscalMes` | int | ✅ | 1–12 |
| `periodoFiscalAnio` | int | ✅ | Ej: `2026` |
| `tipoIdSujetoRetenido` | string(2) | ✅ | |
| `idSujetoRetenido` | string(20) | ✅ | |
| `razonSocialSujetoRetenido` | string(300) | ✅ | |
| `emailSujetoRetenido` | string | — | Email para notificación del RIDE |
| `docsSustento` | array | ✅ | Documentos de sustento con sus retenciones |

---

## 5. Email — comportamiento

El email con el RIDE se envía **únicamente cuando el SRI autoriza** el comprobante. Para `NoAutorizado`, `Devuelto` o `Timeout` no se envía nada al cliente.

| Tipo | Campo de email |
|------|---------------|
| FAC / LC / NC / ND | `emailCliente` |
| GR | `emailDestinatario` |
| Retención | `emailSujetoRetenido` |

Acepta múltiples destinatarios separados por `,` `;` o `|`:

```json
"emailCliente": "contabilidad@empresa.com,gerencia@empresa.com"
```

Si el campo se omite, no se envía email.

---

## 6. Consultar estado

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
  "fechaAutorizacion": "2026-07-08T14:32:11-05:00",
  "mensajeSriOriginal": null
}
```

### `estadoGeneral` — campo principal para lógica de negocio

| Valor | Significado | Acción |
|-------|-------------|--------|
| `EnProceso` | Firmando / enviando / esperando SRI | Seguir polleando |
| `Finalizado` | **Autorizado por el SRI** ✓ — email enviado | Proceso completo |
| `RequiereAccion` | SRI rechazó o error | Leer `mensajeSriOriginal` |
| `Anulado` | Anulado manualmente en el portal | No reintenta |

### Polling recomendado

```
t+0s → t+5s → t+15s → t+30s → t+60s → t+120s → t+300s (último)
```

---

## 7. Batch (lote)

El endpoint acepta un **array JSON** en el body. Máximo **10 ítems** por request.

- Procesamiento secuencial — si un ítem falla, los siguientes continúan.
- HTTP 207 Multi-Status si hay éxitos y fallos mezclados.
- HTTP 422 si todos fallaron.

```json
// Request
[{ "tipoDocumento": "01", ... }, { "tipoDocumento": "01", ... }]

// Respuesta 207
[
  { "index": 0, "comprobanteId": 1848, "estado": "Generado", "error": null },
  { "index": 1, "comprobanteId": null, "estado": "Error", "error": "Punto de emisión '003' no encontrado." }
]
```

---

## 8. Ejemplos por tipo de comprobante

### 8.1 Factura (01)

```json
{
  "tipoDocumento": "01",
  "ambiente": "Produccion",
  "establecimiento": "001",
  "puntoEmision": "001",
  "secuencial": null,
  "fechaEmision": "2026-07-08T10:30:00-05:00",
  "tipoIdentificacionReceptor": "04",
  "identificacionCliente": "1792456789001",
  "razonSocialCliente": "Empresa ABC S.A.",
  "direccionComprador": "Av. República del Salvador N34-183, Quito",
  "emailCliente": "facturacion@empresaabc.com",
  "propina": 0,
  "moneda": "DOLAR",
  "detalles": [
    {
      "codigoPrincipal": "SRV-001",
      "descripcion": "Servicio de desarrollo de software",
      "cantidad": 1,
      "precioUnitario": 1000.00,
      "descuento": 0,
      "impuestos": [
        { "codigo": "2", "codigoPorcentaje": "4", "tarifa": 15, "baseImponible": 1000.00, "valor": 150.00 }
      ]
    }
  ],
  "pagos": [{ "formaPago": "01", "total": 1150.00 }],
  "infoAdicional": [{ "nombre": "Orden de compra", "valor": "OC-2026-0099" }]
}
```

### 8.2 Liquidación de Compra (03)

```json
{
  "tipoDocumento": "03",
  "ambiente": "Produccion",
  "establecimiento": "001",
  "puntoEmision": "001",
  "secuencial": null,
  "fechaEmision": "2026-07-08T09:00:00-05:00",
  "tipoIdentificacionReceptor": "05",
  "identificacionCliente": "0912345678",
  "razonSocialCliente": "María Elena Torres",
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
        { "codigo": "2", "codigoPorcentaje": "0", "tarifa": 0, "baseImponible": 1400.00, "valor": 0 }
      ]
    }
  ],
  "pagos": [{ "formaPago": "01", "total": 1400.00 }]
}
```

### 8.3 Nota de Crédito (04)

```json
{
  "tipoDocumento": "04",
  "ambiente": "Produccion",
  "establecimiento": "001",
  "puntoEmision": "001",
  "secuencial": null,
  "fechaEmision": "2026-07-08T11:00:00-05:00",
  "tipoIdentificacionReceptor": "04",
  "identificacionCliente": "1792456789001",
  "razonSocialCliente": "Empresa ABC S.A.",
  "emailCliente": "facturacion@empresaabc.com",
  "codDocModificado": "01",
  "numDocModificado": "0807202618021900110010010000001801234567811",
  "fechaEmisionDocSustento": "2026-07-05T00:00:00-05:00",
  "motivo": "Devolución parcial por error en cantidad facturada",
  "moneda": "DOLAR",
  "detalles": [
    {
      "codigoPrincipal": "SRV-001",
      "descripcion": "Devolución parcial servicio",
      "cantidad": 1,
      "precioUnitario": 200.00,
      "descuento": 0,
      "impuestos": [
        { "codigo": "2", "codigoPorcentaje": "4", "tarifa": 15, "baseImponible": 200.00, "valor": 30.00 }
      ]
    }
  ]
}
```

### 8.4 Nota de Débito (05)

```json
{
  "tipoDocumento": "05",
  "ambiente": "Produccion",
  "establecimiento": "001",
  "puntoEmision": "001",
  "secuencial": null,
  "fechaEmision": "2026-07-08T12:00:00-05:00",
  "tipoIdentificacionReceptor": "04",
  "identificacionCliente": "1792456789001",
  "razonSocialCliente": "Empresa ABC S.A.",
  "emailCliente": "facturacion@empresaabc.com",
  "codDocModificado": "01",
  "numDocModificado": "0801202618021900110010010000001651234567812",
  "fechaEmisionDocSustento": "2026-07-01T00:00:00-05:00",
  "moneda": "DOLAR",
  "detalles": [
    {
      "codigoPrincipal": "INT-001",
      "descripcion": "Interés por mora — 30 días",
      "cantidad": 1,
      "precioUnitario": 45.00,
      "descuento": 0,
      "impuestos": [
        { "codigo": "2", "codigoPorcentaje": "4", "tarifa": 15, "baseImponible": 45.00, "valor": 6.75 }
      ]
    }
  ],
  "pagos": [{ "formaPago": "16", "total": 51.75, "plazo": 15, "unidadTiempo": "dias" }]
}
```

### 8.5 Guía de Remisión (06)

```json
{
  "tipoDocumento": "06",
  "ambiente": "Produccion",
  "establecimiento": "001",
  "puntoEmision": "001",
  "secuencial": null,
  "fechaEmision": "2026-07-08T08:00:00-05:00",
  "tipoIdentificacionDestinatario": "04",
  "identificacionDestinatario": "1792456789001",
  "razonSocialDestinatario": "Empresa ABC S.A.",
  "dirDestinatario": "Parque Industrial, Bloque C Local 12, Quito",
  "emailDestinatario": "logistica@empresaabc.com",
  "dirPartida": "Bodega Central, Av. Eloy Alfaro N36-92, Quito",
  "motivoTraslado": "Traslado a cliente",
  "fechaIniTransporte": "2026-07-08T07:00:00-05:00",
  "fechaFinTransporte": "2026-07-08T18:00:00-05:00",
  "rucTransportista": "1791234567001",
  "razonSocialTransportista": "Transporte Rápido S.A.",
  "placa": "PBZ-1234",
  "codDocSustentoGr": "01",
  "numDocSustentoGr": "001-001-000000184",
  "numAutDocSustentoGr": "0807202618021900110010010000001841234567818",
  "fechaEmisionDocSustentoGr": "2026-07-08T00:00:00-05:00",
  "detalles": [
    {
      "codigoPrincipal": "PROD-001",
      "descripcion": "Servidor rack 2U",
      "cantidad": 2,
      "precioUnitario": 0,
      "descuento": 0,
      "impuestos": []
    }
  ]
}
```

### 8.6 Comprobante de Retención (07)

```json
{
  "tipoDocumento": "07",
  "ambiente": "Produccion",
  "establecimiento": "001",
  "puntoEmision": "001",
  "secuencial": null,
  "fechaEmision": "2026-07-08T16:00:00-05:00",
  "periodoFiscalMes": 7,
  "periodoFiscalAnio": 2026,
  "tipoIdSujetoRetenido": "04",
  "idSujetoRetenido": "1792456789001",
  "razonSocialSujetoRetenido": "Empresa ABC S.A.",
  "emailSujetoRetenido": "conta@empresaabc.com",
  "docsSustento": [
    {
      "codSustento": "01",
      "codDocSustento": "01",
      "numDocSustento": "001-001-000000200",
      "fechaEmisionDocSustento": "2026-07-07T00:00:00-05:00",
      "numAutDocSustento": "0707202618021792456789001001001000000200123456781",
      "formaPago": "01",
      "pagoLocExt": "01",
      "retenciones": [
        { "codigo": "1", "codigoRetencion": "303", "baseImponible": 1000.00, "porcentajeRetener": 2 },
        { "codigo": "2", "codigoRetencion": "725", "baseImponible": 1000.00, "porcentajeRetener": 10 }
      ]
    }
  ]
}
```

---

## 9. Endpoint secundario — XML pre-generado

Solo para sistemas que **ya generan su propio XML SRI** y no pueden cambiar ese proceso. Para integraciones nuevas, usar siempre el endpoint JSON (`POST /v1/api/comprobantes`).

```
POST /v1/api/xml
```

```json
{
  "tipoDocumento": "01",
  "ambiente": "Produccion",
  "xmlBase64": "<XML SRI completo en Base64>",
  "emailCliente": "cliente@empresa.com"
}
```

| Campo | Tipo | Req. | Descripción |
|-------|------|------|-------------|
| `tipoDocumento` | string | ✅ | Código SRI del tipo |
| `ambiente` | string | ✅ | `"Pruebas"` o `"Produccion"` |
| `xmlBase64` | string | ✅ | XML SRI v1.1.0 completo en Base64 |
| `emailCliente` | string | — | Email(s) para RIDE. Si se omite, Asapp extrae `<emailCliente>` del `<infoFactura>` en el XML. |

> El XML **no debe incluir** `<emailCliente>` dentro de `<infoFactura>` — el SRI rechaza ese campo. Siempre pasar el email en el campo JSON del request.

---

## 10. Referencia de campos

### Tipo de identificación

| Código | Documento |
|--------|-----------|
| `"04"` | RUC (13 dígitos) |
| `"05"` | Cédula (10 dígitos) |
| `"06"` | Pasaporte |
| `"07"` | Consumidor final — identificación = `"9999999999999"` |
| `"08"` | Identificación exterior |
| `"09"` | Placa |

### IVA — codigoPorcentaje (2026)

El `codigo` del impuesto IVA siempre es `"2"`. Fórmula: `valor = round(base × tarifa / 100, 2)`.

| `codigoPorcentaje` | Tarifa | Vigencia |
|--------------------|--------|---------|
| `"0"` | 0% | Siempre válido |
| `"2"` | Exento de IVA | Siempre válido |
| `"3"` | No objeto de IVA | Siempre válido |
| `"4"` | **15%** | Desde abril 2024 ← usar en 2026 |
| `"6"` | 5% | Bienes específicos |

> `codigoPorcentaje: "2"` era IVA 12% — ya no válido desde 2024. Usar `"4"` para 15%. El SRI rechaza con `[52] ERROR EN DIFERENCIAS`.

### Formas de pago

| Código | Descripción |
|--------|-------------|
| `"01"` | Efectivo / sin sistema financiero |
| `"16"` | Tarjeta de débito |
| `"17"` | Dinero electrónico |
| `"19"` | Tarjeta de crédito |
| `"20"` | Otros con sistema financiero |

### Tipos de comprobante — resumen

| `tipoDocumento` | Nombre | Campo email | Genera XML |
|-----------------|--------|-------------|-----------|
| `"01"` | Factura | `emailCliente` | Asapp ✓ |
| `"03"` | Liquidación de Compra | `emailCliente` | Asapp ✓ |
| `"04"` | Nota de Crédito | `emailCliente` | Asapp ✓ |
| `"05"` | Nota de Débito | `emailCliente` | Asapp ✓ |
| `"06"` | Guía de Remisión | `emailDestinatario` | Asapp ✓ |
| `"07"` | Comprobante de Retención | `emailSujetoRetenido` | Asapp ✓ |

---

## 11. Manejo de errores

### Códigos HTTP

| Código | Cuándo |
|--------|--------|
| `202 Accepted` | Comprobante creado — flujo SRI en curso |
| `207 Multi-Status` | Batch con éxitos y fallos mezclados |
| `400 Bad Request` | JSON malformado, campos requeridos ausentes, batch vacío o >10 ítems |
| `401 Unauthorized` | API key inválida o revocada |
| `422 Unprocessable Entity` | Todos los ítems del batch fallaron |
| `500 Internal Server Error` | Error interno Asapp |

### Errores comunes del SRI (`mensajeSriOriginal`)

| Error | Causa | Solución |
|-------|-------|----------|
| `[52] ERROR EN DIFERENCIAS` | `codigoPorcentaje` incorrecto o `valor ≠ base × tarifa/100` | Usar `"4"` para 15% en 2026 |
| `[35] ARCHIVO NO CUMPLE ESTRUCTURA XML` | XML inválido (solo afecta a `/v1/api/xml`) | Validar XML contra XSD SRI v1.1.0 |
| `[56] ERROR ESTABLECIMIENTO CERRADO` | Establecimiento cerrado en el SRI | Activar en el portal del SRI |
| `RUC no existe` | RUC del receptor inactivo | Verificar en portal SRI |
| `Clave de acceso registrada` | Comprobante duplicado | No reenviar |
| `Fecha de emisión no válida` | `fechaEmision` fuera del rango ±5 días | Corregir la fecha |

Los comprobantes en `RequiereAccion` pueden corregirse y reenviarse desde el portal Asapp.

---

## Soporte

Para dudas técnicas o reporte de errores de integración:  
**admin@csmcodes.com** · Asunto: `[API Externa] <descripción breve>`
