# Finnegans FileSender — Puntos de Venta

Portal web para subir un archivo Excel y enviar su contenido como puntos de venta a Finnegans (`POST /api/puntoVenta`).

## Cómo usarlo

```bash
npm install       # solo la primera vez
npm start         # levanta el portal en http://localhost:4600
```

Abrí `http://localhost:4600` en el navegador:

1. **Seleccionar archivo** — arrastrá o elegí el `.xlsx`. Opcionalmente indicá un `EmpresaID` por defecto (se usa cuando la columna `SUCURSAL` viene vacía) y un `TransaccionSubtipoID` por defecto.
2. **Vista previa** — el archivo se agrupa por `NUMERO`: cada grupo es un punto de venta y cada fila un ítem. Con "Ver JSON" podés inspeccionar el payload exacto antes de enviar. Destildá los que no quieras mandar.
3. **Enviar** — envía los seleccionados uno por uno. Cada fila muestra su estado (verde = creado, rojo = error; clic en el estado rojo muestra la respuesta completa de Finnegans).

## Formato del Excel

Una fila por ítem. Filas con el mismo `NUMERO` forman un mismo punto de venta. Columnas usadas:

| Columna | Campo API (alias) |
|---|---|
| NUMERO | agrupador de comprobante |
| COMPROBANTE | `IdentificacionExterna` + `NumeroComprobante`; su letra inicial define `ComprobanteTipoImpositivoID` (A→001, B→006 — configurable) |
| FECHA / FECHACOMPROBANTE / FECHABASEVENCIMIENTO | `Fecha` / `FechaComprobante` / `FechaBaseVencimiento` (aaaa-mm-dd) |
| CLIENTE | `ClienteCodigo` |
| CONDICIONPAGO | `CondicionPagoCodigo` |
| MONEDA | `MonedaCodigo` |
| TRANSACCIONSUBTIPO | `TransaccionSubtipoCodigo` (si falta o está vacía, se usa `PTOVTA-FV`) |
| DESCRIPCION | `Descripcion` |
| SUCURSAL | No se utiliza: `EmpresaCodigo` se fija en `ejemplo` |
| MONEDA_COTIZACION + COTIZACION | `Cotizaciones` |
| PRODUCTO / DESCRIPCIONITEM / CANTIDAD / PRECIO | ítem en `Productos` (`ProductoCodigo`, `Descripcion`, `Cantidad`, `Precio`, con `ImporteExento` = precio × cantidad) |

Además el payload incluye automáticamente (según JSON validado contra el tenant):

- `TransaccionTipoCodigo: OPER` y `WorkflowCodigo` tomado de la columna `WORKFLOW` del Excel.
- El arreglo `Conceptos` se omite por completo del JSON.
- Cuando `CONDICIONPAGO` es `TC/TD` y `COMPROBANTEADICIONAL` es `9520 VISA`, el cobro se genera en `PuntoVentaItemsTarjeta` con `OperacionBancariaCodigo` tomado de `CONDICIONPAGO` y cuenta `13100`; `COMPROBANTEADICIONAL` se copia en `Descripcion`, `CLIENTE` se usa como documento del titular y `COMPROBANTE`, conservando solo sus digitos y sin ceros iniciales, como numero de cupon.
- Cuando `CONDICIONPAGO` es `CONTADO`, se genera unicamente `PuntoVentaItemsEfectivo`: para moneda `PES` usa la cuenta `10000` y para `DOL` la cuenta `10010`.
- Cuando `CONDICIONPAGO` es `CTACTE`, no se incluye ningun arreglo de cobro.
- Los demas cobros se generan como `PuntoVentaItemsOtros` contra la cuenta puente `TCV` por el total del comprobante.
- Totales como strings: `Total`, `TotalBruto`, `TotalPagos`, `TotalConceptos`, `TotalRetenciones`, `Vuelto`.
- `VendedorCodigo` se omite (los códigos del Excel no existen en el tenant).

Estas constantes (subtipo, cuenta de cobro, tipos impositivos por letra, vendedor) se ajustan en el bloque `CONFIG` de [mapping.js](mapping.js). Columnas vacías se omiten del payload.

## Configuración

Todos los comprobantes se generan con `EmpresaCodigo: ejemplo`.

Credenciales en `.env` (no se exponen al navegador; el envío pasa por el servidor local):

```
FINNEGANS_CLIENT_ID=...
FINNEGANS_CLIENT_SECRET=...
PORT=4600
```

## Estructura

- `server.js` — Express: sirve el portal, `POST /api/parse` (parsea y arma la vista previa) y `POST /api/enviar` (obtiene el token OAuth de Teamplace y postea cada punto de venta a Finnegans).
- `mapping.js` — parseo del Excel y armado del payload (editable).
- `public/` — frontend (HTML/CSS/JS sin frameworks).
- `ejemplo-pedido-venta.xlsx` — archivo de ejemplo usado para desarrollar el mapeo.
