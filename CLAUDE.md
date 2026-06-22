# Monitor Agro Colombia

Instrucciones del proyecto. **Lee este archivo completo antes de escribir o
editar cualquier código.** Estas reglas existen para que el proyecto sea
modular, reproducible y fácil de editar pieza por pieza más adelante.

---

## 1. Qué es este proyecto

Un monitor semanal de las **condiciones que afectan la agroexportación de café
de Colombia**. Cada semana recoge precio internacional del café, tipo de cambio
USD/COP, precio interno de la FNC, clima en los **departamentos cafeteros** y
señales de noticias, calcula un índice exploratorio de oportunidad/riesgo por
departamento y produce un reporte ejecutivo. La salida final será un tablero en
Streamlit alimentado por snapshots semanales archivados.

**Pivote a Colombia:** el proyecto nació comparando países de LatAm (Colombia,
Brasil, Perú, Honduras, México). Se reorientó a comparar los departamentos
cafeteros de Colombia entre sí. Los países retirados quedan recuperables en el
historial de git. La columna de geografía pasó de `pais` a `geografia` para
reflejar que ahora mezcla niveles (GLOBAL / COLOMBIA / departamento).

Es un proyecto de portafolio: combina análisis de datos, negocios
internacionales, economía y geopolítica. La frecuencia semanal es **honesta**:
solo usamos datos que de verdad cambian semana a semana.

## 2. Principios de diseño (no negociables)

1. **Un contrato fijo por fuente.** Cada módulo de datos en `fuentes/` expone
   una única función pública `obtener()` que devuelve un `DataFrame` de pandas
   con un esquema documentado y estable. Las tripas de un módulo se pueden
   reescribir (p. ej. cambiar de API) sin tocar nada más, siempre que el
   esquema de salida se mantenga.

2. **Todo lo editable vive en `config.py`.** Países, coordenadas, monedas,
   tickers, pesos del score, parámetros de las fuentes. La lógica **nunca**
   tiene valores "quemados". Para agregar un país o cambiar un peso, se edita
   solo `config.py`.

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
│   ├── unir.py          # junta las fuentes en una tabla semanal
│   └── score.py         # metodología del índice
├── reporte/
│   ├── __init__.py
│   └── generar.py       # resumen ejecutivo + tablas
├── datos/
│   ├── historico/       # series diarias y semanales desde 2023
│   └── snapshots/       # foto semanal archivada (histórico)
├── tests/               # pruebas unitarias sin depender de internet
└── app.py               # Streamlit (fase tardía)
```

## 4. El contrato de las fuentes

Cada módulo de `fuentes/` expone `def obtener() -> pandas.DataFrame`.

**Contrato numérico** (para `fx.py`, `cafe.py`, `clima.py`, `precio_interno.py`,
`contexto.py`). DataFrame en formato largo/tidy con estas columnas exactas:

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
  `procesar/historico.py` construye un histórico separado, diario y semanal,
  desde 2023; solo incluye semanas cerradas y su actualización es idempotente.
- **Fase 3 — Score.** Metodología del índice, con datos reales en mano.
- **Fase 4 — Reporte.** Resumen ejecutivo + tablas (la IA entra aquí).
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
```

## 9. Disciplina de edición

- Ediciones **quirúrgicas**: cambiar lo mínimo necesario para la tarea.
- **No romper el contrato** de las fuentes (sección 4). Si hay que cambiar el
  esquema, avisar y actualizar este CLAUDE.md y todos los módulos afectados.
- No mezclar dos fases en un mismo cambio.
- Si una fuente revela una limitación, documentarla en el módulo y mencionarla,
  no esconderla.

## 10. Secretos y API keys

- **Nunca** escribir keys en el código ni en `config.py`.
- Las keys (Alpha Vantage en fases tardías; Anthropic si se usa para el
  resumen) van en un archivo `.env` que **está en `.gitignore`** y se leen como
  variables de entorno.
- `.env`, `.venv/` y `__pycache__/` nunca se suben al repositorio.
