# Análisis exploratorio: ventas Colombia 2024

> Leer en: [English](README.md) | **Español**

> Documentación técnica del proyecto · Bootcamp Talento Tech AVD
> Ejecutor: Víctor Aguilar

---

## Pregunta problema

¿Cuáles son los productos más rentables por ciudad y canal de venta en Colombia durante 2024, y qué vendedor tiene el mayor desempeño según la satisfacción del cliente?

**Objetivo:** identificar oportunidades de optimización comercial mediante el cruce de rentabilidad geográfica, comportamiento por canal y desempeño individual de la fuerza de ventas.

---

## Contexto

| Atributo | Valor |
|---|---|
| Alcance temporal | Año 2024 |
| Alcance geográfico | Colombia (4 ciudades principales) |
| Unidad de análisis | Transacción de venta |
| Audiencia | Equipo comercial y dirección |

---

## Dataset

- **Origen:** archivo `ventas_sucias.csv`, dataset sintético generado con fines pedagógicos para entrenar habilidades de auditoría y limpieza de datos. Las anomalías están inducidas intencionalmente.
- **Grano:** una fila representa una transacción individual de venta. Clave primaria: `id`.
- **Volumen original:** 49 registros, 11 columnas.
- **Tras limpieza:** 46 registros válidos. Se eliminan 2 duplicados lógicos y 1 fila irrecuperable.
- **Columna derivada:** se agrega `mes` (formato YYYY-MM) para análisis temporal agrupado.

---

## Diccionario de datos

| Columna | Tipo | Descripción | Observaciones de calidad |
|---|---|---|---|
| `id` | int | Identificador único de la transacción | Sin problemas |
| `fecha` | string | Fecha de la transacción | Formatos mixtos: ISO, DD/MM/YYYY, YYYY/MM/DD |
| `vendedor` | string | Nombre del vendedor responsable | Algunos vacíos; usar para exclusión, no imputar |
| `ciudad` | string | Ciudad donde se efectuó la venta | Capitalización inconsistente, tildes faltantes |
| `categoria` | string | Categoría del producto | Algunos vacíos |
| `producto` | string | Nombre específico del producto | Sin problemas relevantes |
| `cantidad` | int | Unidades vendidas en la transacción | Algunos valores vacíos |
| `precio_unitario` | int | Precio por unidad en pesos colombianos | Formato mixto: `$`, puntos de miles, comas |
| `ingreso_total` | int | cantidad × precio_unitario | Algunos vacíos; reconstruir con multiplicación |
| `canal_venta` | string | Canal de comercialización | Sinónimos: online/Online/e-commerce, fisica/TIENDA |
| `satisfaccion_cliente` | float | Calificación del cliente, escala 1-5 | Valores fuera de rango: -1, 0, 8; un NaN |
| `mes` | string | Año-mes derivado de `fecha` | Generada en limpieza, formato YYYY-MM |

---

## Hallazgos de calidad

Auditoría realizada con inspección manual y validación cruzada de tipos, rangos y patrones.

| Anomalía | Severidad | Impacto en análisis |
|---|---|---|
| Vendedor vacío | Alta | Contamina rankings con categoría "(En blanco)" o "Desconocido" |
| Satisfacción fuera de rango | Alta | Distorsiona promedio y mediana; afecta segmentación |
| Sinónimos en canal | Alta | 6 valores cuando debería haber 2 |
| Formato de precio mixto | Media | Impide operaciones aritméticas |
| Capitalización inconsistente | Media | Genera categorías falsas en agrupaciones |
| Duplicados lógicos | Media | Infla ingresos y conteo de transacciones |
| Formato de fecha inconsistente | Media | Series temporales con huecos |
| Campos vacíos | Baja | Imputables con regla de negocio |
| NaN en numéricas | Baja | Pandas los excluye automáticamente |

---

## Demo visual

El reporte interactivo completo está disponible en [`reports/eda_ventas_sucias_anomalias.html`](reports/eda_ventas_sucias_anomalias.html). Tres vistas clave:

**1. Memo analítico y ficha del dataset** — Contexto del proyecto, pregunta de negocio y resumen del dataset.

![Memo analítico](assets/screenshots/01_analytical_memo.png)

**2. Inspector de anomalías** — Tabla interactiva con detección de anomalías fila por fila, clasificadas por tipo y severidad.

![Inspector de anomalías](assets/screenshots/02_anomaly_inspector.png)

**3. Dashboard EDA** — Vista ejecutiva con KPIs, tendencias de ingresos, top de vendedores y detección de outliers (box plot de satisfacción). Todos los gráficos se recalculan después de ejecutar el pipeline de limpieza.

![Dashboard EDA](assets/screenshots/03_eda_dashboard.png)

---

## Pipeline de limpieza (Python / pandas)

### Fechas

```python
df["fecha"] = pd.to_datetime(
    df["fecha"],
    infer_datetime_format=True,
    errors="coerce"
)
```

Detección automática del patrón mixto. Los que no parsean quedan NaT y se revisan aparte.

### Vendedor

```python
df["vendedor"] = df["vendedor"].str.strip().str.title()
sin_vendedor = df[df["vendedor"].isna() | (df["vendedor"] == "")]
df_ranking = df[df["vendedor"].notna() & (df["vendedor"] != "")]
```

No se imputa porque es identidad, no atributo. Los vacíos se reportan por separado y se excluyen del ranking.

### Ciudad

```python
mapa_ciudades = {"bogota": "Bogotá", "medellin": "Medellín"}
df["ciudad"] = (df["ciudad"].str.strip().str.lower()
                .replace(mapa_ciudades)
                .str.title())
```

Diccionario de variantes con tildes para consolidar capitalizaciones inconsistentes.

### Categoría

```python
df["categoria"] = df["categoria"].str.strip().str.title()
```

### Precio unitario

```python
df["precio_unitario"] = (df["precio_unitario"]
    .astype(str)
    .str.replace(r"[$,.]", "", regex=True)
    .astype(int))
```

El punto se trata como separador de miles. Validar contra rango esperado antes de convertir.

### Ingreso total

```python
mask = df["ingreso_total"].isna()
df.loc[mask, "ingreso_total"] = (
    df.loc[mask, "cantidad"] * df.loc[mask, "precio_unitario"]
)
```

Reconstrucción determinística cuando ambos insumos existen. No se inventa lo que no se puede derivar.

### Canal de venta

```python
mapa_canal = {
    "e-commerce": "Online", "online": "Online",
    "fisica": "Tienda", "TIENDA": "Tienda", "tienda": "Tienda"
}
df["canal_venta"] = df["canal_venta"].str.strip().replace(mapa_canal)
```

Consolidar 6 variantes a las 2 categorías reales del negocio.

### Satisfacción del cliente

```python
df["satisfaccion_cliente"] = pd.to_numeric(
    df["satisfaccion_cliente"], errors="coerce"
).clip(1, 5)
```

Valores fuera de rango se acotan al extremo válido más cercano.

### Duplicados

```python
cols_clave = ["fecha", "vendedor", "ciudad", "producto",
              "cantidad", "precio_unitario"]
df = df.drop_duplicates(subset=cols_clave, keep="first").reset_index(drop=True)
```

Conservar la primera ocurrencia para mantener trazabilidad cronológica.

---

## Decisiones analíticas clave

1. **Vendedor vacío no se imputa.** Excluir y reportar aparte. Imputar con "Desconocido" en una columna de ranking distorsiona el resultado: "Desconocido" puede liderar el ranking al acumular transacciones no atribuidas.

2. **Satisfacción acotada con `clip(1, 5)`.** Los valores -1, 0 y 8 son errores de captura. Acotar al rango válido preserva la transacción para otros análisis sin contaminar la métrica.

3. **Duplicados eliminados por subset de columnas clave, no por `id` completo.** Dos registros con id distinto pero misma fecha/vendedor/ciudad/producto/cantidad/precio son lógicamente la misma venta.

4. **Ingreso total reconstruido, no imputado.** Si `cantidad` y `precio_unitario` existen, el ingreso se calcula determinísticamente. No se inventa nada.

5. **Fila irrecuperable eliminada.** 1 registro con múltiples campos críticos vacíos simultáneamente (sin posibilidad de reconstrucción) se descarta y se documenta.

---

## Resultados

### KPIs generales

| Métrica | Valor |
|---|---|
| Ingreso total | $57.747.000 COP |
| Transacciones válidas | 46 |
| Ticket promedio | $1.255.370 COP |
| Satisfacción promedio | 4.15 / 5.0 |

### Ingresos por ciudad

| Ciudad | Ingreso total | Transacciones | Ticket promedio | % del total |
|---|---|---|---|---|
| Bogotá | $33.074.000 | 15 | $2.204.933 | 57.3% |
| Medellín | $8.898.000 | 11 | $808.909 | 15.4% |
| Cali | $8.677.000 | 11 | $788.818 | 15.0% |
| Barranquilla | $7.014.000 | 8 | $876.750 | 12.1% |
| Sin Ciudad | $84.000 | 1 | $84.000 | 0.1% |

### Ingresos por canal

| Canal | Ingreso total | Transacciones | % del total |
|---|---|---|---|
| Online | $40.455.000 | 25 | 70.1% |
| Tienda | $17.292.000 | 21 | 29.9% |

### Top 5 productos

| Producto | Ingreso |
|---|---|
| Laptop | $22.500.000 |
| Celular | $12.600.000 |
| Tablet | $7.200.000 |
| Audífonos | $3.420.000 |
| Licuadora | $3.200.000 |

### Ranking de vendedores

Metodología del score:

```
score = (ingreso_normalizado) × (satisfaccion_normalizada)
ingreso_normalizado = ingreso_vendedor / max(ingreso)
satisfaccion_normalizada = satisfaccion_media / 5
```

| Posición | Vendedor | Ingreso | Satisfacción media | Transacciones | Score |
|---|---|---|---|---|---|
| 1 | Ana Torres | $21.990.000 | 4.44 / 5 | 10 | 0.888 |
| 2 | Carlos Ríos | $15.715.000 | 3.98 / 5 | 9 | 0.569 |
| 3 | Laura Díaz | $7.625.000 | 4.19 / 5 | 9 | 0.291 |
| 4 | María López | $5.954.000 | 4.49 / 5 | 8 | 0.243 |
| 5 | Pedro Gómez | $3.963.000 | 3.63 / 5 | 9 | 0.131 |

**Conclusión:** Ana Torres lidera ambas dimensiones (ingreso y satisfacción combinadas), con un score de 0.888 en una escala de 0 a 1.

---

## Limitaciones del análisis

1. **Tamaño muestral reducido:** 46 transacciones post-limpieza son insuficientes para inferencia estadística robusta. Los hallazgos son descriptivos, no predictivos.

2. **Ausencia de costos:** no hay costo unitario ni gastos asociados. La "rentabilidad" se aproxima por ingresos, no por margen neto.

3. **Sin información del cliente:** no hay datos demográficos ni de fidelización para segmentación avanzada.

4. **Cobertura geográfica limitada:** solo 4 ciudades principales; no representa zonas intermedias ni rurales.

5. **Sesgo de la métrica de satisfacción:** la escala 1-5 no captura la dispersión emocional que sí muestran escalas más amplias como NPS (Net Promoter Score, escala 0-10 que clasifica a los clientes en Detractores 0-6, Pasivos 7-8 y Promotores 9-10, calculando un índice entre -100 y +100) o encuestas 1-10. Un cliente "muy satisfecho" y un "evangelizador" son indistinguibles en 1-5.

**Recomendación:** tratar estos resultados como hipótesis preliminares, no como conclusiones cerradas. Validar con datasets de mayor volumen y datos de costo antes de tomar decisiones de inversión.

---

## Estructura del repositorio

```
retail-data-cleaning-pipeline/
├── assets/
│   └── screenshots/
│       ├── 01_analytical_memo.png
│       ├── 02_anomaly_inspector.png
│       └── 03_eda_dashboard.png
├── data/
│   ├── raw/
│   │   └── ventas_sucias.csv          # Dataset original con anomalías inducidas
│   └── processed/
│       └── ventas_limpias.csv         # Dataset limpio (46 registros)
├── notebooks/
│   └── 01_limpieza_ventas.ipynb       # Pipeline completo de limpieza paso a paso
├── reports/
│   └── eda_ventas_sucias_anomalias.html   # Inspector interactivo de anomalías
├── .gitignore
├── README.md                              # Versión en inglés (principal)
└── README.es.md                           # Este documento
```

---

## Cómo reproducir el análisis

```bash
# 1. Clonar el repo
git clone https://github.com/va-mathml/retail-data-cleaning-pipeline.git
cd retail-data-cleaning-pipeline

# 2. Instalar dependencias
pip install pandas matplotlib jupyter

# 3. Ejecutar el notebook
jupyter notebook notebooks/01_limpieza_ventas.ipynb
```

El notebook es autocontenido. Todas las rutas son relativas. No se requieren claves de API externas.

---

## Sobre el autor

Víctor Aguilar · Ejecutor Bootcamp AVD · Talento Tech 2026

Matemático y físico titulado (Universidad del Valle), educador en IA/ML e instructor de analítica de datos en Cali, Colombia. Actualmente construyendo un portafolio de proyectos end-to-end en datos y ML orientados a roles remotos internacionales.

GitHub: [va-mathml](https://github.com/va-mathml) | LinkedIn: [vaguilar-ai](https://linkedin.com/in/vaguilar-ai)
