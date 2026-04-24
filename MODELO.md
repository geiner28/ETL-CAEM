# Documentación del Modelo de Datos — ETL CAEM

> Esquema en estrella (star schema) que centraliza los oficios de embargo/desembargo emitidos en Colombia, normalizando entidades remitentes (juzgados y entidades coactivas), bancos receptores y geografía (departamento/municipio).

- **Motor:** MySQL / Cloud SQL — esquema `ETL`
- **Tabla de hechos:** `fact_oficios`
- **Dimensiones:** `dim_entidades`, `dim_entidades_judiciales`, `dim_entidades_coactivas`, `dim_variantes`, `dim_entidad_bancaria`, `dim_departamentos`, `dim_municipios`
- **Definición SQL:** [schema.sql](schema.sql)

---

## 1. Diagrama de relaciones

```
                       ┌───────────────────────┐
                       │   dim_departamentos   │
                       │  PK departamento_id   │
                       └───────────┬───────────┘
                                   │ 1
                                   │
                          ┌────────┴────────┐
                        N │                 │ N
              ┌───────────▼─────────┐  ┌────▼─────────────────┐
              │   dim_municipios    │  │     dim_entidades    │◄────────┐
              │  PK municipio_id    │  │     PK entidad_id    │         │
              │  FK departamento_id │  │  FK municipio_id     │         │
              └─────────┬───────────┘  │  FK departamento_id  │         │
                        │ 1            └─────────┬────────────┘         │
                        │                        │ 1                    │
                        │                        ├──────────────┐       │
                        │                        │ 1            │ 1     │
                        │              ┌─────────▼──────────┐  ┌▼──────────────┐
                        │              │ dim_entidades_     │  │ dim_entidades_│
                        │              │ judiciales         │  │ coactivas     │
                        │              │ PK/FK entidad_id   │  │ PK/FK         │
                        │              └────────────────────┘  └───────────────┘
                        │                        ▲
                        │                        │ 1
                        │                ┌───────┴──────────┐
                        │              N │  dim_variantes   │
                        │                │ FK entidad_id    │
                        │                └──────────────────┘
                        │
                        │ N            ┌────────────────────────┐
              ┌─────────▼──────────────┴┐                       │
              │      fact_oficios       │ N    ┌────────────────▼─────────┐
              │   PK oficio_id          ├─────►│   dim_entidad_bancaria   │
              │   FK entidad_remitente  │      │   PK entidad_bancaria_id │
              │   FK entidad_bancaria   │      └──────────────────────────┘
              │   FK municipio_id       │
              │   FK departamento_id    │
              └─────────────────────────┘
```

**Cardinalidades clave**

| Relación | Tipo | Descripción |
|---|---|---|
| `dim_departamentos` 1—N `dim_municipios` | Jerarquía geográfica | Un depto contiene muchos municipios |
| `dim_municipios` N—1 `dim_departamentos` | FK obligatoria | |
| `dim_entidades` 1—1 `dim_entidades_judiciales` | Especialización opcional | Solo si la entidad es juzgado |
| `dim_entidades` 1—1 `dim_entidades_coactivas` | Especialización opcional | Solo si la entidad es coactiva |
| `dim_entidades` 1—N `dim_variantes` | Aliases / nombres crudos | Cada variante observada en los oficios originales |
| `fact_oficios` N—1 `dim_entidades` | Remitente del oficio | |
| `fact_oficios` N—1 `dim_entidad_bancaria` | Banco que recibe el oficio | |
| `fact_oficios` N—1 `dim_municipios` / `dim_departamentos` | Ubicación del demandado | |

---

## 2. Tabla de hechos

### `fact_oficios` — un registro por oficio recibido

| Columna | Tipo | Descripción |
|---|---|---|
| `oficio_id` | `VARCHAR(20)` **PK** | Identificador único del oficio en el sistema |
| `entidad_remitente_id` | `INT` **FK → dim_entidades** | Entidad que emite el oficio (juzgado o coactiva) |
| `entidad_bancaria_id` | `INT` **FK → dim_entidad_bancaria** | Banco al que va dirigido |
| `estado` | `VARCHAR(30)` | `CONFIRMADO`, `PROCESADO`, `RECONFIRMADO`, `PROCESADO_CON_ERRORES`, `EN_PROCESO`, … |
| `numero_oficio` | `VARCHAR(100)` | Número original del oficio |
| `fecha_oficio` | `DATE` | Fecha de emisión |
| `fecha_recepcion` | `DATE` | Fecha en que el banco lo recibe |
| `titulo_embargo` | `VARCHAR(50)` | `COACTIVO` \| `JUDICIAL` |
| `titulo_orden` | `VARCHAR(50)` | `EMBARGO` \| `DESEMBARGO` \| `REQUERIMIENTO` |
| `monto` | `DECIMAL(18,2)` | Monto declarado en el oficio |
| `monto_a_embargar` | `DECIMAL(18,2)` | Monto efectivamente a embargar (sumado en KPIs con tope 10 000 M COP) |
| `nombre_demandado` | `VARCHAR(300)` | Persona o empresa demandada |
| `id_demandado` | `VARCHAR(30)` | Cédula / NIT del demandado |
| `tipo_id_demandado` | `VARCHAR(20)` | `CEDULA`, `CEDULA_EXTRANJERIA`, `PASAPORTE`, `TARJETA_IDENTIDAD`, `CARNET_DIPLOMATICO`, `NIT`, `FIDEICOMISO`, `SOCIEDAD_EXTRANJERIA_SIN_NIT` |
| `direccion_remitente` | `VARCHAR(500)` | Dirección reportada por el remitente |
| `correo_remitente` | `VARCHAR(200)` | Email reportado por el remitente |
| `nombre_funcionario` | `VARCHAR(200)` | Funcionario que firma el oficio |
| `municipio_id` | `INT` **FK → dim_municipios** | Municipio del demandado |
| `departamento_id` | `INT` **FK → dim_departamentos** | Depto del demandado |
| `fuente_ubicacion` | `VARCHAR(30)` | Origen del dato de ubicación (`OFICIO`, `ENTIDAD`, `INFERIDA`, …) |
| `referencia` | `VARCHAR(200)` | Referencia interna del caso |
| `expediente` | `VARCHAR(200)` | Número de expediente judicial / coactivo |
| `created_at` | `DATETIME` | Timestamp de ingreso al sistema |
| `confirmed_at` | `DATETIME` | Timestamp de confirmación |
| `processed_at` | `DATETIME` | Timestamp de procesamiento final |

**Índices**
- `idx_oficios_entidad (entidad_remitente_id)`
- `idx_oficios_estado (estado)`
- `idx_oficios_muni (municipio_id)`
- `idx_oficios_depto (departamento_id)`
- `idx_oficios_fecha (fecha_oficio)`

**Métricas derivadas en el dashboard**
- *Tiempo de respuesta* = `DATEDIFF(confirmed_at, created_at)` (0–365 días).
- *Resolución* = `DATEDIFF(processed_at, created_at)` (1–365 días).
- *Sumas de monto*: siempre con cap `monto_a_embargar <= 10 000 000 000`.

---

## 3. Dimensión maestra de entidades remitentes

### `dim_entidades` — entidad consolidada (juzgado o coactiva)

| Columna | Tipo | Descripción |
|---|---|---|
| `entidad_id` | `INT` **PK** | Identificador único |
| `nombre_normalizado` | `VARCHAR(500)` | Nombre canónico tras normalización |
| `nombre_real` | `VARCHAR(500)` | Nombre oficial verificado contra fuente externa |
| `tipo` | `VARCHAR(50)` | `JUDICIAL`, `COACTIVO`, `OTRO` |
| `subtipo` | `VARCHAR(50)` | Subclase dentro del tipo |
| `categoria` | `VARCHAR(20)` | Categoría agregada (p. ej. nivel territorial) |
| `nit` | `VARCHAR(50)` | NIT |
| `cod_institucion` | `VARCHAR(50)` | Código institucional |
| `email` | `VARCHAR(200)` | Correo de contacto |
| `direccion` | `VARCHAR(500)` | Dirección física |
| `telefono` | `VARCHAR(100)` | Teléfono |
| `ciudad` | `VARCHAR(150)` | Ciudad (texto) |
| `municipio_id` | `INT` **FK → dim_municipios** | Municipio normalizado |
| `departamento_id` | `INT` **FK → dim_departamentos** | Depto normalizado |
| `orden` | `VARCHAR(50)` | Orden (nacional/territorial) |
| `sector` | `VARCHAR(100)` | Sector de la entidad |
| `naturaleza_juridica` | `VARCHAR(100)` | Naturaleza jurídica |
| `estado` | `VARCHAR(30)` | Estado operativo |
| `representante` | `VARCHAR(200)` | Representante legal |
| `total_registros` | `INT` | # de oficios asociados (precalculado) |
| `num_variantes` | `INT` | # de aliases en `dim_variantes` |

**Índices:** `idx_entidades_tipo`, `idx_entidades_cat`, `idx_entidades_muni`, `idx_entidades_depto`.

---

### `dim_entidades_judiciales` — extensión 1:1 para juzgados

`entidad_id` es a la vez **PK** y **FK → dim_entidades.entidad_id**. Solo existe fila si la entidad es de tipo `JUDICIAL`.

| Columna | Tipo | Descripción |
|---|---|---|
| `entidad_id` | `INT` **PK/FK** | Hereda de `dim_entidades` |
| `nombre_extraido` | `VARCHAR(500)` | Nombre tal como aparece en los oficios |
| `nombre_real` | `VARCHAR(500)` | Nombre oficial cruzado con la Rama Judicial |
| `ciudad` | `VARCHAR(150)` | |
| `email_extraido` / `email_real` | `VARCHAR(200)` | Email observado vs. verificado |
| `codigo_despacho` | `VARCHAR(20)` | Código DANE/Rama Judicial del despacho |
| `numero_despacho` | `VARCHAR(20)` | Número de despacho |
| `jurisdiccion` | `VARCHAR(50)` | Jurisdicción (civil, laboral, etc.) |
| `distrito` | `VARCHAR(100)` | Distrito judicial |
| `circuito` | `VARCHAR(100)` | Circuito judicial |
| `juez` | `VARCHAR(200)` | Juez titular |
| `direccion`, `telefono` | | Datos de contacto |
| `area` | `VARCHAR(50)` | Área del despacho |
| `total_registros` | `INT` | # de oficios asociados |

---

### `dim_entidades_coactivas` — extensión 1:1 para entidades coactivas

`entidad_id` es **PK/FK → dim_entidades.entidad_id**. Solo existe fila si es `COACTIVO`.

| Columna | Tipo | Descripción |
|---|---|---|
| `entidad_id` | `INT` **PK/FK** | Hereda de `dim_entidades` |
| `nombre_extraido` / `nombre_real` | `VARCHAR(500)` | Nombre crudo vs. oficial |
| `ciudad` | `VARCHAR(150)` | |
| `email_extraido` / `email_real` | `VARCHAR(200)` | |
| `nit`, `cod_institucion` | | Identificadores oficiales |
| `orden` | `VARCHAR(50)` | Orden territorial |
| `sector`, `naturaleza_juridica`, `tipo_institucion` | | Clasificación |
| `direccion`, `telefono`, `pagina_web` | | Contacto |
| `estado` | `VARCHAR(30)` | Estado operativo |
| `representante`, `cargo_representante` | | Representante legal |
| `total_registros` | `INT` | # de oficios asociados |

---

### `dim_variantes` — aliases observados de cada entidad

| Columna | Tipo | Descripción |
|---|---|---|
| `variante_id` | `INT` **PK** | |
| `entidad_id` | `INT` **FK → dim_entidades** | Entidad canónica a la que mapea |
| `nombre_normalizado` | `VARCHAR(500)` | Nombre canónico (denormalizado por conveniencia) |
| `variante_original` | `VARCHAR(500)` | Texto exacto encontrado en los oficios crudos |
| `conteo` | `INT` | # de veces que apareció esa variante |

**Índice:** `idx_variantes_entidad (entidad_id)`.

> Esta tabla soporta el proceso de normalización: cada variante textual cruda apunta a su `entidad_id` consolidado.

---

## 4. Dimensión bancaria

### `dim_entidad_bancaria` — bancos receptores

| Columna | Tipo | Descripción |
|---|---|---|
| `entidad_bancaria_id` | `INT` **PK** | |
| `nombre` | `VARCHAR` | Nombre del banco (p. ej. `CITIBANK`, `SANTANDER`) |

Referenciada por `fact_oficios.entidad_bancaria_id`.

---

## 5. Dimensiones geográficas

### `dim_departamentos`

| Columna | Tipo | Descripción |
|---|---|---|
| `departamento_id` | `INT` **PK** | |
| `nombre` | `VARCHAR(100)` **UNIQUE** | Nombre del departamento |

### `dim_municipios`

| Columna | Tipo | Descripción |
|---|---|---|
| `municipio_id` | `INT` **PK** | |
| `nombre` | `VARCHAR(150)` | Nombre del municipio |
| `departamento_id` | `INT` **FK → dim_departamentos** | Depto al que pertenece |

**Índice:** `idx_municipios_depto (departamento_id)`.

> La geografía se referencia desde `fact_oficios` (ubicación del demandado) y desde `dim_entidades` (sede de la entidad remitente).

---

## 6. Reglas y convenciones del modelo

1. **Star schema clásico:** `fact_oficios` es la única tabla de hechos; las dimensiones nunca se enlazan entre sí salvo por la jerarquía geográfica y la especialización 1:1 de entidades.
2. **Especialización por herencia (1:1):** una entidad de `dim_entidades` puede tener registro en `dim_entidades_judiciales` *o* `dim_entidades_coactivas`, según `tipo`. No ambas.
3. **Normalización de nombres:** la trazabilidad nombre crudo → entidad canónica vive en `dim_variantes`.
4. **Geografía dual:** la ubicación en `fact_oficios` corresponde al **demandado**, no al remitente. Para la sede de la entidad remitente usar `dim_entidades.municipio_id` / `departamento_id`.
5. **Outliers de monto:** todas las agregaciones de `monto_a_embargar` aplican el cap `<= 10 000 000 000` (ver dashboard).
6. **Timestamps de ciclo de vida:** `created_at → confirmed_at → processed_at` definen el embudo de procesamiento del oficio.

---

## 7. Joins de referencia rápida

```sql
-- Oficio enriquecido con entidad remitente, banco y geografía del demandado
SELECT f.oficio_id,
       e.nombre_normalizado     AS remitente,
       e.tipo                   AS tipo_remitente,
       b.nombre                 AS banco,
       d.nombre                 AS departamento,
       m.nombre                 AS municipio,
       f.titulo_orden, f.titulo_embargo,
       f.monto_a_embargar, f.estado, f.fecha_oficio
FROM fact_oficios f
LEFT JOIN dim_entidades        e ON f.entidad_remitente_id = e.entidad_id
LEFT JOIN dim_entidad_bancaria b ON f.entidad_bancaria_id  = b.entidad_bancaria_id
LEFT JOIN dim_departamentos    d ON f.departamento_id      = d.departamento_id
LEFT JOIN dim_municipios       m ON f.municipio_id         = m.municipio_id;
```

```sql
-- Detalle judicial de un oficio (solo si el remitente es JUDICIAL)
SELECT f.oficio_id, j.codigo_despacho, j.distrito, j.circuito, j.juez
FROM fact_oficios f
JOIN dim_entidades            e ON f.entidad_remitente_id = e.entidad_id
JOIN dim_entidades_judiciales j ON e.entidad_id           = j.entidad_id
WHERE e.tipo = 'JUDICIAL';
```

```sql
-- Variantes crudas que mapearon a una entidad
SELECT v.variante_original, v.conteo
FROM dim_variantes v
WHERE v.entidad_id = ?
ORDER BY v.conteo DESC;
```
