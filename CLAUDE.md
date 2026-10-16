# Asistente de extracción de facturas → JSON

Este proyecto convierte comprobantes fiscales argentinos (AFIP/ARCA) en archivos JSON
estructurados, listos para subir a un sistema posterior que los parsea.

**Tu tarea NO es parsear a ciegas.** Cada vez que el usuario pida procesar una o más
facturas, seguí OBLIGATORIAMENTE el protocolo interactivo de este documento: primero
detectás, después deducís y confirmás con el usuario los puntos ambiguos, y recién al
final escribís el JSON.

## Archivos de referencia

- `FACTURAS_SCHEMA_CONSOLIDADO.json` — **el JSON Schema de salida. Es la ley.** El JSON
  generado debe respetarlo exactamente (mismos campos, mismos tipos, mismos `required`,
  `additionalProperties: false`). No agregues ni quites campos.

---

# PROTOCOLO OBLIGATORIO

Cuando el usuario te pida procesar factura(s), ejecutá estas tres fases en orden.
Nunca escribas un JSON de salida sin haber cerrado la Fase B.

## FASE A — Detección (automática, sin preguntar todavía)

0. **Leé TODAS las páginas de TODOS los archivos antes de empezar.** No proceses solo la
   primera página. Un listado de productos puede continuar en páginas siguientes: esos
   ítems pertenecen a la MISMA factura. No dupliques encabezados repetidos como si fueran
   productos.

1. **Contá los comprobantes reales.** Un archivo puede tener una factura, varias, una
   factura de varias páginas, o copias (Original/Duplicado/Triplicado/Copia) del MISMO
   comprobante. Las copias del mismo comprobante = **una sola factura** (mismo tipo,
   letra, punto de venta, número, fecha y contenido). No dupliques ítems ni totales.

2. **Identificá el emisor** (proveedor que emite): CUIT y razón social. Buscá si ya
   existe su perfil en `perfiles/{cuit_emisor}.json`.

   **Si el emisor NO aparece en el texto del PDF** (encabezado/logo suele venir como
   imagen), NO lo dejes en `null` sin más: rasterizá la zona del encabezado y leela
   visualmente. Procedimiento con PyMuPDF (no requiere poppler):
   ```python
   import fitz
   d = fitz.open("archivo.pdf"); p = d[0]; r = p.rect
   # franja superior; ampliá el clip si el CUIT queda cortado a la derecha
   clip = fitz.Rect(0, 0, r.width, r.height*0.16)
   p.get_pixmap(matrix=fitz.Matrix(4,4), clip=clip).save("header.png")
   ```
   Después leé `header.png` con la herramienta Read (interpreta imágenes) y extraé razón
   social, CUIT, domicilio, IIBB e inicio de actividades. Dejá una advertencia indicando
   que el emisor se obtuvo del encabezado en imagen. Esto es clave: el nombre del archivo
   de salida depende del CUIT del emisor.

3. **Mapeá la tabla de ítems**: identificá qué representa cada columna antes de leer
   valores (código, descripción, cantidad, precio unitario, bonif./descuento, IVA,
   importe, etc.).

4. **Fijá el nivel de confianza de lectura:**
   - **PDF con texto** (seleccionable) → confianza ALTA. Podés deducir con soltura.
   - **PDF escaneado / imagen** → confianza BAJA. Sé más cauto: confirmá números que
     puedan estar borrosos o columnas que puedan estar desalineadas, y dejá `null` +
     `advertencias` ante cualquier ilegibilidad. No completes caracteres por suposición.

## FASE B — Perfil del proveedor y confirmación

El perfil guarda cómo factura ese proveedor, para no volver a preguntar lo mismo cada vez.

### Si YA existe `perfiles/{cuit_emisor}.json`

Aplicá sus reglas y mostrale al usuario un **resumen corto** para confirmar, por ejemplo:

> Proveedor **{razón social}** ({cuit}). Perfil conocido: IVA por ítem · cantidad por
> unidad · descuentos por ítem · con percepciones. ¿Sigo con estas reglas? (sí / ajusto)

Si el usuario confirma, saltá a la Fase C. Si algo cambió, actualizá el perfil.

### Si NO existe el perfil

Corré la ronda de confirmación de los **ejes problemáticos** (abajo). La postura por
defecto es **deducir → proponer → esperar el OK del usuario**: intentá resolver cada eje
por tu cuenta, mostrá tu razonamiento y esperá confirmación. Solo preguntá abierto
(sin proponer) cuando realmente no puedas determinarlo.

Al cerrar la ronda, **guardá el perfil** en `perfiles/{cuit_emisor}.json` (ver formato
al final) para las próximas veces.

> Regla de lote: si procesás varios archivos y aparece **otro emisor**, repetí esta fase
> para ese proveedor. No mezcles reglas entre proveedores distintos.

### Ejes problemáticos a confirmar

Presentá lo detectado en lenguaje claro y pedí confirmación. Ejemplos de redacción:

**1. Cantidad y unidad**
- Si ves cantidad + precio unitario sin indicios de bulto:
  > "Detecté **cantidad por unidad** y **precio unitario**. Tomo `cantidad` = lo impreso,
  > `factor_unidades` = 1. ¿Confirmás?"
- Si hay precio logístico + precio unitario, o descripción tipo `12x350g`, o un total de
  bultos/cajas:
  > "Parece **venta por bulto**: 1 renglón = N unidades (factor ≈ …, deducido de
  > {precio_bulto} / {precio_unitario}). Normalizaría `cantidad` a unidades. ¿Confirmás
  > o es por unidad?"
- Recordá: no infieras bulto solo por números de la descripción; necesitás evidencia de
  precios/importes. Ante duda real → `tipo_cantidad_original: "desconocida"`,
  `factor_unidades: null`.

**2. Descuentos / bonificaciones / recargos**
- Por ítem:
  > "Hay descuento **por ítem**. `precio_unitario_sin_iva` lo calculo como
  > `importe_neto_linea / cantidad` (ya neto, después del descuento). Los ajustes quedan
  > listados en `ajustes_linea`. ¿OK?"
- General de factura:
  > "El descuento es **general de la factura**, no lo bajo a los ítems: va en
  > `totales.descuentos`. Los precios de línea quedan sin tocar. ¿OK?"
- Nunca sumes porcentajes sucesivos como uno solo (5% y 10% → `0.95 × 0.90`, no 15%).

**3. Alícuotas de IVA** (cuando el usuario dice "IVA" se refiere a la alícuota)
- Alícuota explícita por ítem (columna IVA/%IVA/Alícuota alineada a la fila) → extraela
  directo, **no preguntes**. Es lo ideal.
- Una sola alícuota en el totalizador y sin exentos/no gravados/otras alícuotas:
  > "Hay **una única alícuota ({X}%)** en la factura. Se la asigno a todos los ítems
  > gravados. ¿Confirmás?"
- Varias alícuotas o IVA global sin discriminar por ítem → **conciliación combinatoria
  con salvaguarda de unicidad** (ver la regla completa en "Alícuota de IVA por
  conciliación única", en Reglas núcleo). En resumen: probás combinaciones; si **una
  sola** reconcilia con el IVA total impreso, la usás y confirmás; si hay cero o varias,
  `alicuota_iva: null` y lo resuelve el sistema posterior.
- Prohibido: asumir 21% por ser Factura A o por "IVA Responsable Inscripto"; asignar una
  alícuota por conocimiento externo del producto (nombre/categoría del producto); asignar
  una combinación de alícuotas cuando NO es la única que reconcilia.

**4. Percepciones / tributos**
- Informá dónde los cargaste (`tributos[]` y `totales.percepciones` / `totales.tributos`).
  No los confundas con el IVA discriminado. No agrupes conceptos que aparecen separados.

## FASE C — Generación del JSON

Recién con los ejes confirmados, generá la salida.

1. Construí el objeto respetando **exactamente** `FACTURAS_SCHEMA_CONSOLIDADO.json`.
   El schema envuelve todo en `{ "facturas": [ ... ] }`.
2. Un archivo de salida **por cada comprobante real** (no por página, no por copia).
3. Nombre del archivo:
   ```
   {cuit_emisor}-{punto_venta}-{numero_comprobante}-{timestamp}.json
   ```
   - `cuit_emisor`: solo dígitos, sin guiones.
   - `punto_venta` y `numero_comprobante`: **conservá los ceros a la izquierda**.
   - `timestamp`: formato `YYYYMMDDHHMMSS` (momento de la generación).
   - Ejemplo: `30712345678-00003-00001234-20260810143022.json`
4. Guardalo en la raíz del proyecto.
5. Después de escribir, informá al usuario: qué archivos generó, cuántas facturas, y
   listá las `advertencias` relevantes si las hubo.

---

# REGLAS NÚCLEO DE EXTRACCIÓN

Válidas siempre, en todas las fases. Ante conflicto, **la precisión gana**: preferí
`null` antes que un dato posiblemente incorrecto.

## Principios

1. No inventes información ni uses conocimiento externo (productos, CUIT, EAN, precios,
   alícuotas habituales). Todo sale del comprobante.
2. Si un campo no aparece, no es legible o no puede determinarse con certeza → `null`.
   Si una colección está vacía → `[]`.
3. No corrijas valores impresos aunque parezcan matemáticamente mal. Si hay una
   inconsistencia relevante, conservá lo impreso y anotala en `advertencias`.
4. Diferenciá siempre **emisor** (encabezado, quien factura) de **receptor**. No cruces
   CUIT, razón social, domicilio ni condición fiscal entre ambos.

## Cálculos: qué está PROHIBIDO y qué está PERMITIDO

**Prohibido calcular / inferir:**
- `importe_neto_linea` (debe salir de un valor monetario explícito del comprobante).
- IVA, bases imponibles, netos o totales que falten.
- Un precio neto dividiendo un precio con IVA por la alícuota.
- Alícuotas por combinación matemática cuando la combinación **no es única** (ver la
  excepción controlada "Alícuota de IVA por conciliación única" más abajo).

**Permitido (únicas excepciones):**
- Inferir `factor_unidades` cuando haya evidencia suficiente (relación entre precios
  comparables, o conciliación con importe de línea y ajustes explícitos).
- `cantidad = cantidad_original × factor_unidades`.
- `precio_unitario_sin_iva = importe_neto_linea / cantidad`, solo si `importe_neto_linea`
  es explícito y confiable, `cantidad` está bien normalizada a unidades comerciales, y el
  neto es antes de IVA. Conservá hasta 6 decimales; quitá ceros finales sobrantes.
- **Base imponible por alícuota, por agrupación:** cuando CADA ítem tiene alícuota
  explícita Y `importe_neto_linea` explícito, la `base_imponible` de cada alícuota se
  obtiene sumando los netos de los ítems que comparten esa alícuota. Esto es una
  agrupación determinística (no la inferencia combinatoria prohibida: no adivinás qué ítem
  va a qué alícuota, ya lo dice el comprobante). Condición obligatoria: el resultado
  `base × alícuota` debe **reconciliar** con el importe de IVA impreso de esa alícuota.
  Si reconcilia, cargá la base; si no reconcilia, dejá `base_imponible: null` y agregá
  una `advertencia`. No apliques esta agrupación si las alícuotas por ítem no son
  explícitas.
- **Alícuota de IVA por conciliación única:** cuando la alícuota NO viene por ítem ni
  discriminada (solo hay un IVA global, o varias alícuotas que no se pueden asociar a los
  ítems), está permitido DEDUCIR la alícuota de cada ítem por conciliación combinatoria,
  **pero solo si la combinación es única**. Procedimiento obligatorio:
  1. Necesitás `importe_neto_linea` explícito por ítem y el importe de IVA total impreso.
  2. Probá asignar a cada ítem una alícuota candidata y buscá las combinaciones cuyo
     `Σ(neto_i × alícuota_i)` reconcilie con el IVA total impreso (tolerancia ~1 centavo).
  3. **Empezá con el conjunto `{10.5, 21}`** (es lo habitual; rara vez hay otra). Solo si
     con `{10.5, 21}` NINGUNA combinación reconcilia, ampliá a `{0, 2.5, 5, 10.5, 21, 27}`.
  4. Contá cuántas combinaciones reconcilian:
     - **exactamente una** → usá esas alícuotas por ítem, cargá `iva[]` discriminado
       (base = Σ netos de esa alícuota; importe = base × alícuota) y dejá una `advertencia`
       aclarando que las alícuotas y el desglose fueron DEDUCIDOS (solo el neto y el IVA
       total son impresos). Confirmá con el usuario antes de escribir.
     - **cero o más de una** → `alicuota_iva: null` por ítem; la resuelve el sistema
       posterior. Nunca elijas una entre varias combinaciones posibles.
  Esta deducción se apoya solo en la matemática (netos + IVA total), nunca en el nombre o
  la categoría del producto. Verificá la unicidad con un barrido explícito, no de memoria.

## Normalización de valores

- **Importes:** string decimal, punto como separador decimal, **sin** separador de miles,
  sin símbolo de moneda. `$ 1.234,56` → `"1234.56"`. Conservá la precisión original; no
  redondees valores impresos.
- **Separador decimal según el proveedor:** no asumas que el punto o la coma es siempre
  miles o decimal. Determiná el formato mirando el resto de la columna, los precios, los
  totales y la consistencia matemática de las líneas. Ej: en ciertos sistemas `1.000`
  puede significar **1 unidad** (formato decimal), no mil. Aplicá el mismo criterio a
  cantidades e importes de ese proveedor.
- **Fechas:** `YYYY-MM-DD`. Si es ambigua → `null`. No inventes componentes.
- **CUIT / EAN / códigos:** solo dígitos, sin guiones ni espacios. No confundas CUIT con
  Ingresos Brutos, CAE/CAEA, DNI, EAN, número de comprobante ni código interno.
- **Alícuotas:** string numérico sin `%`. `21%` → `"21.0"`; `10,5%` → `"10.5"`;
  `0%` → `"0.0"`.
- **Punto de venta / número de comprobante:** conservá ceros a la izquierda (son strings).
- **Códigos (CAE, CAEA, etc.):** strings, sin modificar dígitos.

## Ítems

- **Incorporá ABSOLUTAMENTE TODOS los ítems. No dejes ninguno afuera.** Recorré todas las
  páginas y todas las filas de producto; extraé cada producto, artículo o servicio, en
  orden, sin omitir ni resumir descripciones (uní las que ocupan varias líneas). Líneas
  repetidas reales = ítems independientes. Antes de cerrar, verificá que la cantidad de
  ítems extraídos coincida con lo que muestra el comprobante.
- No conviertas en producto una fila que sea subtotal, transporte, encabezado, leyenda
  fiscal o continuación de descripción.
- Trazabilidad de cantidad: siempre completá `cantidad_original`,
  `tipo_cantidad_original` (`unidad_comercial` | `unidad_logistica` | `desconocida`),
  `factor_unidades` y `cantidad`.
- `codigo` (interno del proveedor) ≠ `codigo_ean` (código de barras). Ante duda → `null`
  en cada uno; no los inventes ni los derives de la descripción.
- `precio_unitario_sin_iva_sin_descuento` y `precio_unitario_sin_iva` deben referirse a
  **la misma unidad comercial** que `cantidad`. Nunca mezcles cantidad en unidades con
  precio de bulto.
- **Comparar precios solo de la misma etapa para deducir bulto (`factor_unidades`).**
  Para inferir que un renglón es un bulto de N unidades, compará únicamente precios
  **comparables**: neto vs neto, antes-de-descuento vs antes-de-descuento. Nunca compares
  precio de lista vs bonificado, ni sin IVA vs con IVA, ni base vs `U.Final` con IVA.
- **Detectar si un precio incluye IVA.** Un nombre como `U.Final`, `Precio Final` o
  similar NO prueba que el valor sea neto sin IVA. Si el valor coincide con
  `precio_base × ajustes × (1 + IVA)`, incluye IVA: **no** lo uses como precio neto.
- **Prioridad para `precio_unitario_sin_iva`:**
  1. Si NO hay descuentos/bonificaciones/recargos y existe precio unitario base explícito
     (por unidad comercial) → usá ese valor (tiene más precisión que dividir el importe).
  2. Si HAY ajustes y `importe_neto_linea` es explícito y confiable →
     `importe_neto_linea / cantidad`.
  3. Si el único precio disponible es el de bulto → `null` (no dividas bulto / factor).
     Igual que si solo hay un precio con IVA sin neto explícito → `null`.
- Validaciones internas (cuando los datos existan):
  `cantidad ≈ cantidad_original × factor_unidades` y
  `cantidad × precio_unitario_sin_iva ≈ importe_neto_linea`. Diferencias por redondeo son
  tolerables; diferencias materiales invalidan la interpretación.

## Advertencias

Usá `advertencias[]` solo para lo que afecta la calidad/confiabilidad de la extracción:
campo ilegible, página cortada, valor parcialmente visible, columnas desalineadas, dos
valores posibles, imposibilidad de asociar productos con múltiples alícuotas, etc. No la
uses para explicar decisiones normales.

---

# FORMATO DEL PERFIL DE PROVEEDOR

Guardá cada perfil como `perfiles/{cuit_emisor}.json` (creá la carpeta si no existe).
Sirve para reconocer al proveedor la próxima vez y evitar repreguntar.

**El perfil describe la ESTRUCTURA y las CAPACIDADES del proveedor, no el estado puntual
de una factura.** Si la tabla del proveedor tiene una columna de bonificación/descuento,
la regla es `descuentos: "por_item"` **aunque en la factura analizada esa columna venga
vacía**: en otra factura puede traer valor. Nunca fijes una regla como "sin descuentos"
solo porque una factura concreta no los tenía. En `notas`, documentá el mapeo de columnas
y cómo se interpretan tanto cuando el campo viene vacío como cuando trae valor (ej:
bonificación vacía → `precio_lista = precio`, `ajustes_linea: []`; bonificación con valor
→ va a `ajustes_linea` y `precio_lista ≠ precio`).

```json
{
  "cuit_emisor": "30712345678",
  "razon_social": "PROVEEDOR EJEMPLO S.A.",
  "reglas": {
    "cantidad": "unidad_comercial",
    "iva": "por_item",
    "descuentos": "por_item",
    "percepciones": true
  },
  "notas": "Columna IVA por ítem alineada. Precio unitario ya neto sin IVA."
}
```

Valores posibles:
- `cantidad`: `"unidad_comercial"` | `"bulto"` | `"mixto"`
- `iva`: `"por_item"` | `"totalizador"`
- `descuentos`: `"por_item"` | `"general"` | `"ninguno"`
- `percepciones`: `true` | `false`

El valor de `iva` describe **de dónde sale la alícuota** en ese proveedor, no cuántas
alícuotas trae una factura puntual:
- `"por_item"`: cada ítem trae su alícuota explícita en la tabla (columna IVA/%IVA/
  Alícuota). Se extrae directo por ítem.
- `"totalizador"`: la alícuota NO está por ítem; solo se informa en el totalizador de la
  factura (a veces discriminada por alícuota, a veces como un único IVA global). Hay que
  resolverla desde ahí. Resolución **por factura**:
  - Si el totalizador muestra **una sola** alícuota gravada → asignala a todos los ítems
    gravados.
  - Si hay **varias** alícuotas / un IVA global sin discriminar → aplicá la regla
    "Alícuota de IVA por conciliación única" (Reglas núcleo): probá `{10.5, 21}` primero
    y, si no reconcilia, `{0, 2.5, 5, 10.5, 21, 27}`. Si **una sola** combinación
    reconcilia → usala y cargá `iva[]` discriminado; si **cero o varias** →
    `alicuota_iva: null` por ítem (lo resuelve el sistema posterior). Confirmá con el
    usuario cuando deduzcas.
  - Si el totalizador SÍ trae las alícuotas discriminadas con base e importe, cargalas en
    `iva[]` directamente desde ahí.

Nunca uses un valor de perfil que describa el estado de una sola factura (ej: "una sola
alícuota"): registrá la estructura del proveedor (`"totalizador"`) y dejá que la cantidad
de alícuotas se resuelva factura por factura.

Actualizá el perfil si en una factura nueva del mismo proveedor las reglas cambian, y
avisale al usuario del cambio.
