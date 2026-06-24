# Monitor Agro Colombia

Instrucciones del proyecto. **Lee este archivo completo antes de escribir o
editar cualquier código.** Estas reglas existen para que el proyecto sea
modular, reproducible y fácil de editar pieza por pieza más adelante.

## 0. Alcance de los documentos de continuidad

- `CLAUDE.md` (este archivo) es la fuente de verdad **técnica y estable**:
  arquitectura, contratos, convenciones, fases y comandos de verificación.
- `CONTEXTO_IAS.md` es la bitácora **operativa y cambiante**: estado real del
  repositorio, últimas validaciones, decisiones recientes, bloqueos y próximo
  paso. No debe copiar las reglas de este archivo.
- `BRIEFING_CHAT.md` es un briefing **estratégico y autosuficiente** para chats
  sin acceso al repositorio. Explica producto, usuarias, criterio y decisiones
  pendientes; evita detalles internos de código.

La única superposición permitida es un resumen mínimo de producto y estado en
`BRIEFING_CHAT.md`, porque ese archivo debe entenderse por sí solo fuera del
repositorio. Ante una contradicción: este archivo manda en asuntos técnicos,
`CONTEXTO_IAS.md` manda sobre el punto operativo actual y `BRIEFING_CHAT.md`
manda sobre decisiones de producto confirmadas por el usuario.

Un asistente de código debe leer primero este archivo y después
`CONTEXTO_IAS.md`. El briefing solo es necesario para conversaciones de
producto o cuando el usuario lo entregue expresamente.

---

## 1. Qué es este proyecto

Un kit de consulta y reporte sobre las **condiciones que afectan la
agroexportación de café de Colombia**. Integra precio internacional del café,
tipo de cambio USD/COP, precio interno de la FNC, producción nacional mensual,
clima en los **departamentos cafeteros** y señales de noticias. La interfaz
permite consultar y exportar evidencia y generar un brief por periodo. Un
posible índice futuro sigue condicionado a conocimiento experto.

**Pivote a Colombia:** el proyecto nació comparando países de LatAm (Colombia,
Brasil, Perú, Honduras, México). Se reorientó a comparar los departamentos
cafeteros de Colombia entre sí. Los países retirados quedan recuperables en el
historial de git. La columna de geografía pasó de `pais` a `geografia` para
reflejar que ahora mezcla niveles (GLOBAL / COLOMBIA / departamento).

La frecuencia semanal es **honesta**: solo usamos datos que de verdad cambian
semana a semana. La motivación de portafolio, las beneficiarias y los criterios
de producto viven en `BRIEFING_CHAT.md`.

## 2. Principios de diseño (no negociables)

1. **Un contrato fijo por fuente.** Cada módulo de datos en `fuentes/` expone
   una única función pública `obtener()` que devuelve un `DataFrame` de pandas
   con un esquema documentado y estable. Las tripas de un módulo se pueden
   reescribir (p. ej. cambiar de API) sin tocar nada más, siempre que el
   esquema de salida se mantenga.

2. **Todo lo editable vive en `config.py`.** Departamentos, coordenadas,
   monedas, tickers, parámetros de las fuentes y futuros pesos del score. La
   lógica **nunca** tiene valores "quemados". Para agregar un departamento o
   cambiar un parámetro, se edita solo `config.py`.

3. **Cada módulo corre solo.** Todo módulo de `fuentes/` debe poder ejecutarse
   de forma aislada con `python -m fuentes.<nombre>` e imprimir/validar su
   propia salida. Así detectamos limitaciones en su propio paso, no al final.

## 3. Estructura de carpetas

```
monitor_agro/
├── CLAUDE.md            # este archivo
├── config.py            # TODO lo editable
├── requirements.txt
├── .gitignore
├── main.py              # orquestador: corre las fases en orden
├── fuentes/             # un módulo por fuente, todos con obtener()
│   ├── __init__.py
│   ├── fx.py            # tipo de cambio
│   ├── cafe.py          # precio del café
│   ├── clima.py         # clima en zonas cafeteras
│   ├── noticias.py      # señales cualitativas (GDELT)
│   └── contexto.py      # Banco Mundial (fase tardía)
├── procesar/
│   ├── __init__.py
│   ├── calidad.py       # validaciones y cobertura de snapshots
│   ├── historico.py     # backfill diario y agregación semanal
│   ├── indicadores.py   # tendencias y comparación departamental
│   ├── visualizacion.py  # dataset y metadatos listos para gráficos
│   ├── proyeccion.py     # escenarios Coffee C, USD/COP, precio FNC y margen
│   ├── unir.py          # junta las fuentes en una tabla semanal
│   └── score.py         # metodología del índice
├── reporte/
│   ├── __init__.py
│   └── generar.py       # brief ejecutivo Markdown por periodo
├── datos/
│   ├── historico/       # series diarias y semanales desde 2023
│   ├── indicadores/     # derivados estadísticos y último resumen
│   ├── visualizacion/    # series, catálogo y resumen para gráficos
│   └── snapshots/       # foto semanal archivada (histórico)
├── tests/               # pruebas unitarias sin depender de internet
└── app.py               # visualizaciones básicas en Streamlit
```

## 4. El contrato de las fuentes

Cada módulo de `fuentes/` expone `def obtener() -> pandas.DataFrame`.

**Contrato numérico** (para `fx.py`, `cafe.py`, `clima.py`, `precio_interno.py`,
`produccion.py`, `contexto.py`). DataFrame en formato largo/tidy con estas
columnas exactas:

| columna     | tipo            | descripción                                            |
|-------------|-----------------|--------------------------------------------------------|
| `fecha`     | date            | fecha del dato                                         |
| `geografia` | str             | nivel geográfico del dato (ver abajo)                  |
| `variable`  | str             | nombre del indicador (ej. `fx_usd_local`)              |
| `valor`     | float           | valor numérico                                         |
| `unidad`    | str             | unidad (ej. `USc/lb`, `COP/USD`, `mm`, `°C`)           |
| `fuente`    | str             | nombre de la fuente (ej. `yfinance`, `open-meteo`)     |

Tras el **pivote a Colombia**, la columna `geografia` tiene tres niveles:
`"GLOBAL"` (café), `"COLOMBIA"` (FX y precio interno) y el nombre del
**departamento cafetero** (clima, ej. `"Huila"`).

**Contrato noticias** (para `noticias.py`, por su naturaleza cualitativa).
DataFrame con: `fecha`, `geografia`, `titulo`, `url`, `fuente`, `idioma`,
`tono` (float, opcional), `categoria` (str, opcional). La clasificación por
IA de la `categoria` se añade en una fase posterior, no al inicio.

Regla: si una fuente no puede entregar algo, devuelve un DataFrame **vacío
pero con las columnas correctas**, nunca un error sin manejar. Y deja un
comentario en el módulo documentando la limitación encontrada.

Para backfill, `obtener(desde, hasta)` usa el mismo contrato y devuelve todas
las observaciones diarias del rango inclusivo. Sin argumentos conserva el
comportamiento operativo original.

## 5. Convenciones de código

- **Idioma:** identificadores y comentarios en **español**, consistente con
  `config.py`. (Se puede renombrar a inglés más adelante si se quiere; ahora
  prima la consistencia.)
- Funciones pequeñas y con una sola responsabilidad.
- Type hints siempre que sea razonable.
- Cada módulo de `fuentes/` termina con un bloque
  `if __name__ == "__main__":` que llama a `obtener()` e imprime un resumen
  (`.head()`, `.shape`, tipos) para poder probarlo aislado.
- Nada de imports innecesarios ni dependencias nuevas sin agregarlas a
  `requirements.txt` y avisar.

## 6. Fuentes de datos y limitaciones conocidas

- **Café (`cafe.py`)** — precio diario vía `yfinance`, ticker `KC=F` (futuro
  ICE Coffee C). Es la fuente más frágil: es no oficial (raspa Yahoo) y se
  puede romper. Alpha Vantage sirve solo como contexto **mensual** (requiere
  API key gratuita). El precio del café es **global**, así que va con
  `geografia="GLOBAL"`.
- **FX (`fx.py`)** — tras el pivote a Colombia, solo **USD/COP**
  (`config.TICKER_FX = "USDCOP=X"`) vía `yfinance`. Frankfurter/BCE no cubre
  COP, por eso se usa yfinance. `geografia="COLOMBIA"`.
- **Precio interno (`precio_interno.py`)** — precio interno de referencia de la
  FNC, raspado del HTML de la página de estadísticas cafeteras. Misma fragilidad
  que el café (scraping). `geografia="COLOMBIA"`.
- **Producción (`produccion.py`)** — producción nacional registrada mensual,
  tomada del Excel "Precios, área y producción de café" de la FNC. Se expresa
  en miles de sacos de 60 kg y conserva un punto por mes, sin relleno semanal.
  `geografia="COLOMBIA"`.
- **Clima (`clima.py`)** — Open-Meteo, gratis y sin key. Solo uso no comercial.
  Se consulta una coordenada por **departamento cafetero** (`config.REGIONES_CAFE`);
  `geografia` = nombre del departamento.
- **Noticias (`noticias.py`)** — GDELT DOC 2.0 vía el cliente `gdeltdoc`,
  gratis y sin key, multilingüe (incluye español). Tras el pivote se consulta
  solo a nivel nacional (`config.PAIS_FIPS`). Mezcla fuentes confiables y
  obscuras: filtrar con criterio, nunca tomar una sola noticia como hecho.
- **Contexto (`contexto.py`)** — API del Banco Mundial, gratis y sin key. Solo
  como telón de fondo anual, nunca como dato que "cambia" cada semana.

## 7. Fases del proyecto

Avanzar **en orden**. **No pasar a la siguiente fase hasta que la anterior
corra y se haya verificado.** Si algo no da, parar ahí y decidir.

- **Fase 0 — Esqueleto.** Estructura de carpetas, `config.py`,
  `requirements.txt`, `.gitignore`, `main.py` que corra sin hacer nada todavía,
  y los stubs vacíos de cada módulo (con `obtener()` devolviendo un DataFrame
  vacío con las columnas correctas). *Verificable:* `python main.py` corre sin
  error.
- **Fase 1 — Fuentes, una por una:** 1a FX → 1b Café → 1c Clima → 1d Noticias.
  Cada submódulo termina corriendo solo y mostrando su DataFrame.
- **Fase 2 — Unir.** `procesar/unir.py` junta todo en una tabla semanal y
  guarda el primer snapshot en `datos/snapshots/`. `procesar/calidad.py`
  valida fechas, duplicados, nulos, valores y cobertura antes de guardar. Un
  snapshot existente no se sobrescribe salvo autorización explícita.
  `procesar/historico.py` construye un histórico separado desde 2023; conserva
  las observaciones mensuales de producción sin forward-fill, solo incluye
  semanas cerradas para las demás series y su actualización es idempotente.
- **Bloque 3 — Indicadores descriptivos.** `procesar/indicadores.py` calcula
  cambios, promedios móviles, anomalías estadísticas y comparaciones entre
  departamentos. No asigna todavía oportunidad, riesgo, bueno ni malo.
- **Bloque 3.5 — Preparación visual.** `procesar/visualizacion.py` añade
  etiquetas, categorías, orden, colores e índice base 100. Es una capa de
  presentación neutral; no contiene criterio experto ni score.
- **Visualizaciones básicas para feedback.** `app.py` presenta panorama
  comercial, evolución por departamento y comparación climática. No equivale
  aún al tablero final ni adelanta el score.
- **Simulador de escenarios.** `procesar/proyeccion.py` desplaza el precio FNC
  observado en proporción a cambios supuestos de Coffee C y USD/COP. La
  interfaz permite editar el costo medio por carga y estimar margen bruto. No
  es un pronóstico ni modela todavía prima, calidad, logística o causalidad.
- **Fase 3 — Score.** Metodología del índice, con datos reales en mano.
- **Fase 4 — Reporte.** Brief ejecutivo Markdown por periodo, con cifras,
  fuentes, cadencias y limitaciones.
- **Fase 5 — Streamlit.** Tablero leyendo los snapshots.
- **Fase 6 — Automatización.** GitHub Actions semanal.
- **Fase 7 — Banco Mundial + pulido para LinkedIn.**

## 8. Cómo ejecutar y verificar

Siempre con el entorno virtual activo (`(.venv)` visible en la terminal):

```
.venv\Scripts\Activate.ps1      # Windows PowerShell
python -m fuentes.fx            # probar un módulo aislado
python main.py                  # correr el orquestador
python -m unittest discover -s tests -v  # pruebas sin internet
python -m procesar.historico    # actualizar el histórico desde 2023
python -m procesar.indicadores  # calcular tendencias y resumen reciente
python -m procesar.visualizacion  # preparar series para gráficos
streamlit run app.py           # abrir visualizaciones en localhost:8501
```

## 9. Disciplina de edición

- Ediciones **quirúrgicas**: cambiar lo mínimo necesario para la tarea.
- **No romper el contrato** de las fuentes (sección 4). Si hay que cambiar el
  esquema, avisar y actualizar este CLAUDE.md y todos los módulos afectados.
- No mezclar dos fases en un mismo cambio.
- Si una fuente revela una limitación, documentarla en el módulo y mencionarla,
  no esconderla.
- Al cerrar un cambio validado, actualizar `CONTEXTO_IAS.md` solo si cambia el
  estado, una decisión, una limitación o el próximo paso. Actualizar
  `BRIEFING_CHAT.md` únicamente cuando cambie el panorama estratégico que un
  chat sin acceso al repositorio necesita conocer.

## 10. Secretos y API keys

- **Nunca** escribir keys en el código ni en `config.py`.
- Las keys (Alpha Vantage en fases tardías; Anthropic si se usa para el
  resumen) van en un archivo `.env` que **está en `.gitignore`** y se leen como
  variables de entorno.
- `.env`, `.venv/` y `__pycache__/` nunca se suben al repositorio.
