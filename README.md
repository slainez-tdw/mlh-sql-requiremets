¡Perfecto! A partir de la conversación, traduje los requerimientos en **historias de usuario** claras para que tu **agente de Bedrock** (vía Lambda → DB) sepa exactamente qué consultar y cómo responder. Incluyo criterios de aceptación, dimensiones/medidas y notas de negocio para que sea implementable sin ambigüedades.

---

# Épica A — Visión general de ventas

## HU-A1: Ver ventas totales

**Como** analista de ventas
**Quiero** ver cuántos **quintales** y **cajas** se han vendido
**Para** medir el volumen total y compararlo con objetivos

**Criterios de aceptación**

* **Dado** un rango de fechas obligatorio
* **Cuando** consulto el dashboard/end-point
* **Entonces** recibo:

  * Volumen en **quintales** (Q)
  * Volumen en **cajas** (solo para categorías “**Leche**” y “**Levadura**”)
  * **Ventas en Lempiras** (L)
  * Fecha y filtros aplicados

**Notas**

* Si la categoría ≠ {Leche, Levadura} → mostrar volumen en **quintales**; si ∈ {Leche, Levadura} → además mostrar **cajas**.

---

# Épica B — Análisis por dimensiones (segmentación)

## HU-B1: Filtrar por tipo de cliente

**Como** gerente comercial
**Quiero** segmentar ventas por **tipo de cliente**
**Para** entender el aporte por canal/perfil

**Criterios de aceptación**

* Filtro “tipo\_cliente” (lista múltiple)
* Devuelve Q, Cajas (si aplica), L y participación (%) dentro del filtro

## HU-B2: Filtrar por razón social / compañía (IN vs MH)

**Como** analista
**Quiero** filtrar por **Razón Social** (campo DB: **COMPAÑIA**)
**Para** ver resultados por **Indalcen (IN)** y/o **Molinos Harineros Sula (MH)**

**Criterios de aceptación**

* Filtro “compañia” ∈ {IN, MH} (multi-select)
* Presente en el payload de entrada y eco en la respuesta (para trazabilidad)
* Si no se envía, devolver ambas y desglosar

## HU-B3: Desglose por Centro de Distribución → Ruta → Categoría → Artículo

**Como** analista
**Quiero** un **drill-down jerárquico**
**Para** pasar de visión macro a micro

**Criterios de aceptación**

* Nivel 0: Totales
* Nivel 1: **Compañía**
* Nivel 2: **Centro de Distribución**
* Nivel 3: **Ruta**
* Nivel 4: **Categoría de artículos**
* Nivel 5: **Artículo**
* En cada nivel: Q, Cajas (si aplica), L y % participación del nivel respecto al superior

---

# Épica C — Contribución en Lempiras

## HU-C1: Ver aporte en moneda

**Como** gerencia
**Quiero** el **aporte en Lempiras** por los mismos cortes
**Para** medir ingresos por segmento

**Criterios de aceptación**

* Métrica **ventas\_L** disponible en todos los niveles de desglose
* Mostrar **% de participación** sobre el total del filtro activo

---

# Épica D — Reglas especiales por categoría

## HU-D1: Manejo de unidades por categoría

**Como** usuario de negocio
**Quiero** que el sistema use **cajas solo para Leche y Levadura**
**Para** respetar cómo se transa cada familia

**Criterios de aceptación**

* Si categoría ∈ {Leche, Levadura} → incluir “cajas\_vendidas”
* Si categoría ∉ {Leche, Levadura} → omitir “cajas\_vendidas” o devolver 0 explícito
* Documentar la regla en la respuesta (metadatos)

---

# Épica E — Participación (share) por categoría y SKU

## HU-E1: % participación por compañía en una **categoría** (p. ej. “TRIGO NACIONAL”)

**Como** analista
**Quiero** ver el **% de participación** de **Indalcen** (o la compañía seleccionada) dentro de una **categoría**
**Para** entender liderazgo por familia

**Criterios de aceptación**

* Filtros: categoría (obligatorio), compañía (opcional, defecto: todas)
* Devuelve: Q, Cajas (si aplica), L y **% participación** de cada compañía dentro de la categoría

## HU-E2: % participación por **producto específico**

**Como** analista
**Quiero** ver la **participación** de la compañía en un **SKU** (p. ej., “ESPAGUETI MI PASTA”)
**Para** medir performance puntual

**Criterios de aceptación**

* Filtros: artículo/SKU (obligatorio), compañía (opcional)
* Devuelve: Q, Cajas (si aplica), L y **% participación** del SKU frente a su categoría y al total del filtro

---

# Épica F — Comparativos YoY

## HU-F1: Comparar año actual vs año anterior en **quintales**

**Como** gerencia
**Quiero** comparar **AA vs AA-1** en **quintales**
**Para** ver crecimiento o caída

**Criterios de aceptación**

* Parámetro “modo\_comparativo” = **YoY**
* Devuelve: Q\_actual, Q\_año\_anterior, **ΔQ** y **%Δ**
* Permite aplicar todos los filtros (compañía, CD, ruta, categoría, artículo)
* (Opcional) Incluir la misma lógica en **Lempiras**

---

# Épica G — Flujo Macro → Micro

## HU-G1: Vista macro y desglose por compañía

**Como** usuario
**Quiero** una **vista macro** y luego un reporte **por compañía**
**Para** mantener la práctica actual de análisis

**Criterios de aceptación**

* Endpoint entrega un bloque “macro” (totales) y un arreglo “por\_compañia” con su propio desglose
* Cada bloque conserva filtros y metadatos de consulta

---

# Épica H — Parámetros, validaciones y metadatos

## HU-H1: Parámetros de consulta y validación

**Como** consumidor del API (agente)
**Quiero** conocer y validar los **parámetros**
**Para** asegurar consultas consistentes

**Criterios de aceptación**

* Parámetros soportados (todos **opcionales** salvo fechas):

  * `fecha_inicio`, `fecha_fin` (obligatorios)
  * `compañia` ∈ {IN, MH}
  * `tipo_cliente`
  * `centro_distribucion`
  * `ruta`
  * `categoria`
  * `articulo`
  * `modo_comparativo` ∈ {YoY, Ninguno}
* Respuesta incluye **echo** de parámetros + reglas aplicadas (p. ej., “cajas solo para Leche/Levadura”)

## HU-H2: Mensaje explicativo previo a error

**Como** operador del agente
**Quiero** que, antes de lanzar un error, el sistema **explique qué endpoint/consulta está intentando ejecutar**
**Para** depurar rápidamente

**Criterios de aceptación**

* En caso de validación fallida o error de ejecución:

  * Incluir campo `intentado` con: endpoint, filtros, jerarquía solicitada
  * Mensaje claro de qué faltó (p. ej., “fecha\_inicio/fecha\_fin obligatorias”)

---

# Épica I — Formatos de salida y KPIs

## HU-I1: Respuesta estándar con métricas y participaciones

**Como** consumidor (frontend/Bedrock)
**Quiero** un **formato JSON consistente**
**Para** graficar y tabular sin post-procesar

**Criterios de aceptación**

* Estructura sugerida:

```json
{
  "filtros": { "fecha_inicio": "YYYY-MM-DD", "fecha_fin": "YYYY-MM-DD", "compañia": ["IN","MH"], "tipo_cliente": ["..."], "centro_distribucion": ["..."], "ruta": ["..."], "categoria": ["..."], "articulo": ["..."], "modo_comparativo": "YoY" },
  "reglas_aplicadas": { "cajas_solo_para": ["Leche", "Levadura"] },
  "macro": { "quintales": 0, "cajas": 0, "lempiras": 0 },
  "por_compañia": [
    {
      "compañia": "IN",
      "totales": { "Q": 0, "Cajas": 0, "L": 0, "share_Q": 0.0, "share_L": 0.0 },
      "desglose": {
        "centros": [
          {
            "centro_distribucion": "CD1",
            "Q": 0, "Cajas": 0, "L": 0, "share_Q": 0.0,
            "rutas": [
              {
                "ruta": "R1",
                "Q": 0, "Cajas": 0, "L": 0, "share_Q": 0.0,
                "categorias": [
                  {
                    "categoria": "TRIGO NACIONAL",
                    "Q": 0, "Cajas": 0, "L": 0, "share_Q": 0.0,
                    "articulos": [
                      { "articulo": "ESPAGUETI MI PASTA", "Q": 0, "Cajas": 0, "L": 0, "share_Q_cat": 0.0 }
                    ]
                  }
                ]
              }
            ]
          }
        ]
      }
    }
  ],
  "comparativos": {
    "YoY": { "Q_actual": 0, "Q_aa": 0, "delta_Q": 0, "pct_Q": 0.0 }
  }
}
```

**KPIs mínimos por bloque**

* **Q** (quintales), **Cajas** (si aplica), **L** (lempiras)
* **share\_Q** y **share\_L** a cada nivel de desglose
* En comparativos: **Δ** y **%Δ**

---

# Épica J — Consultas frecuentes (plantillas)

## HU-J1: Macro general del período

* Sin filtros de compañía; devuelve totales, por compañía y desglose jerárquico.

## HU-J2: Participación de IN en “TRIGO NACIONAL”

* Filtros: `compañia = IN`, `categoria = TRIGO NACIONAL`
* KPIs: Q, L, **share** dentro de la categoría

## HU-J3: Participación en SKU “ESPAGUETI MI PASTA”

* Filtros: `articulo = ESPAGUETI MI PASTA`
* Reportar share del SKU vs su **categoría** y vs **total** del filtro

## HU-J4: YoY por compañía en quintales

* `modo_comparativo = YoY`, desglosado por `compañia`

---

## Dimensiones y Medidas (resumen implementable)

**Dimensiones (filtros)**

* Fecha (rango obligatorio)
* COMPAÑIA ∈ {IN, MH}
* Tipo de cliente
* Centro de distribución
* Ruta
* Categoría
* Artículo (SKU)

**Medidas**

* **quintales** (Q)
* **cajas** (solo para {Leche, Levadura})
* **lempiras** (L)
* **share** por nivel (Q y L)
* **YoY**: Q\_actual, Q\_aa, ΔQ, %Δ (y opcional L)

---

## Reglas de negocio clave (derivadas de la conversación)

1. **Unidades**: Cajas solo aplican a **Leche** y **Levadura**; el resto se expresa en **quintales**.
2. **Compañía**: Filtro de **Razón Social** se mapea al campo **COMPAÑIA** (valores típicos: **IN**, **MH**).
3. **Jerarquía de análisis**: Macro → Compañía → Centro de Distribución → Ruta → Categoría → Artículo.
4. **Participación**: Debe poder calcularse por **categoría** y por **SKU** específico.
5. **Comparativo**: **Año actual vs año anterior** (al menos en **quintales**).
6. **Transparencia de ejecución**: Antes de cualquier error, el sistema debe **resumir el endpoint/consulta** e **informar parámetros recibidos**.

---

Si quieres, en el siguiente paso te dejo:

* Un **contrato de request/response** para el Lambda (tipos TS incluidos).
* **SQL esqueleto** para cada HU (agregaciones, group by y cálculos de share).
* Casos límite (sin ventas, filtros vacíos, artículo no pertenece a categoría, etc.).
