# Monitor Agro Colombia

Instrucciones del proyecto. **Lee este archivo completo antes de escribir o
editar código.** Las reglas mantienen el proyecto modular, reproducible y
editable pieza por pieza.

## 0. Documentos de continuidad

- `CLAUDE.md` (este): verdad **técnica y estable** — arquitectura, contratos,
  convenciones, fases, comandos.
- `CONTEXTO_IAS.md`: bitácora **operativa** — estado real, validaciones,
  decisiones recientes, bloqueos, próximo paso. No copia reglas de aquí.
- `BRIEFING_CHAT.md`: briefing **estratégico autosuficiente** para chats sin
  repo — producto, usuarias, criterio, decisiones pendientes; sin código.

Leer primero este archivo y luego `CONTEXTO_IAS.md`; el briefing solo para temas
de producto. Ante contradicción: este manda en lo técnico, `CONTEXTO_IAS.md` en
el punto operativo, `BRIEFING_CHAT.md` en decisiones de producto confirmadas.

## 1. Qué es

Kit de consulta y reporte sobre las **condiciones que afectan la agroexportación
de café de Colombia**: precio internacional del café, USD/COP, precio interno
FNC, producción y exportaciones nacionales mensuales, clima en departamentos
cafeteros y señales de noticias. La interfaz consulta y exporta evidencia y
genera un brief por periodo.
Un índice futuro queda condicionado a conocimiento experto.

**Pivote a Colombia:** nació comparando países LatAm (Colombia, Brasil, Perú,
Honduras, México); se reorientó a comparar departamentos cafeteros entre sí (los
retirados quedan en git). La columna `pais` pasó a `geografia` porque ahora
mezcla niveles (GLOBAL / COLOMBIA / departamento). La frecuencia semanal es
**honesta**: solo datos que cambian semana a semana. Motivación y producto en
`BRIEFING_CHAT.md`.

## 2. Principios de diseño (no negociables)

1. **Un contrato fijo por fuente.** Cada módulo de `fuentes/` expone una sola
   función pública `obtener()` → `DataFrame` con esquema estable. Las tripas se
   pueden reescribir (cambiar de API) sin tocar nada más si el esquema se mantiene.
2. **Todo lo editable vive en `config.py`** (departamentos, coordenadas, monedas,
   tickers, parámetros, pesos futuros del score). La lógica nunca tiene valores
   "quemados"; para agregar un departamento o cambiar un parámetro se edita solo
   `config.py`.
3. **Cada módulo corre solo** con `python -m fuentes.<nombre>`, imprimiendo y
   validando su salida, para detectar límites en su propio paso.

## 3. Estructura de carpetas

```
monitor_agro/
├── config.py            # TODO lo editable
├── main.py              # orquestador: corre las fases en orden
├── fuentes/             # un módulo por fuente, todos con obtener()
│   ├── fx.py            # USD/COP
│   ├── cafe.py          # precio del café
│   ├── produccion.py    # producción nacional mensual FNC
│   ├── exportaciones.py # exportaciones nacionales mensuales FNC
│   ├── referencia_mercado_fnc.py # trío FNC/NY/TRM para calibración
│   ├── clima.py         # clima en zonas cafeteras
│   ├── noticias.py      # señales cualitativas (GDELT)
│   └── contexto.py      # Banco Mundial (fase tardía)
├── procesar/
│   ├── calidad.py       # validaciones y cobertura de snapshots
│   ├── historico.py     # backfill diario y agregación semanal
│   ├── indicadores.py   # tendencias y comparación departamental
│   ├── visualizacion.py # dataset y metadatos para gráficos
│   ├── proyeccion.py    # escenarios Coffee C, USD/COP, FNC, factor y margen
│   ├── calibracion_fnc.py # persiste el ajuste implícito diario de la FNC
│   ├── unir.py          # junta las fuentes en una tabla semanal
│   └── score.py         # metodología del índice (pendiente)
├── reporte/
│   ├── generar.py       # brief Markdown e informe Markdown del simulador
│   ├── excel.py         # libro comercial filtrable con resumen y diccionario
│   └── pdf.py           # brief del periodo en PDF (matplotlib + reportlab)
├── datos/               # historico/ indicadores/ visualizacion/ snapshots/
├── tests/               # pruebas unitarias sin internet
└── app.py               # tablero Streamlit
```

## 4. El contrato de las fuentes

Cada módulo de `fuentes/` expone `def obtener() -> pandas.DataFrame`.

**Contrato numérico** (`fx.py`, `cafe.py`, `clima.py`, `precio_interno.py`,
`produccion.py`, `exportaciones.py`, `contexto.py`): DataFrame largo/tidy con
estas columnas exactas:

| columna     | tipo  | descripción                                       |
|-------------|-------|---------------------------------------------------|
| `fecha`     | date  | fecha del dato                                    |
| `geografia` | str   | nivel geográfico (ver abajo)                      |
| `variable`  | str   | nombre del indicador (ej. `fx_usd_local`)         |
| `valor`     | float | valor numérico                                    |
| `unidad`    | str   | unidad (ej. `USc/lb`, `COP/USD`, `mm`, `°C`)      |
| `fuente`    | str   | nombre de la fuente (ej. `yfinance`, `open-meteo`)|

`geografia` tiene tres niveles: `"GLOBAL"` (café), `"COLOMBIA"` (FX y precio
interno) y el nombre del **departamento** (clima, ej. `"Huila"`).

**Contrato noticias** (`noticias.py`, cualitativo): `fecha`, `geografia`,
`titulo`, `url`, `fuente`, `idioma`, `tono` (float, opcional), `categoria` (str,
opcional). La clasificación IA de `categoria` es de una fase posterior.

Regla: si una fuente no entrega algo, devolver un DataFrame **vacío con las
columnas correctas**, nunca un error sin manejar, y documentar la limitación en
el módulo. Para backfill, `obtener(desde, hasta)` usa el mismo contrato y
devuelve las observaciones diarias del rango inclusivo; sin argumentos conserva
el comportamiento operativo.

## 5. Convenciones de código

- **Español** en identificadores y comentarios (consistente con `config.py`).
- Funciones pequeñas, una responsabilidad, con type hints razonables.
- Cada módulo de `fuentes/` cierra con `if __name__ == "__main__":` que llama a
  `obtener()` e imprime un resumen (`.head()`, `.shape`, tipos).
- Nada de imports innecesarios ni dependencias nuevas sin agregarlas a
  `requirements.txt` y avisar.

## 6. Fuentes y limitaciones conocidas

- **Café (`cafe.py`)** — diario vía `yfinance`, ticker `KC=F` (ICE Coffee C). La
  más frágil (raspa Yahoo). Alpha Vantage solo como contexto mensual (key
  gratuita). Global → `geografia="GLOBAL"`.
- **FX (`fx.py`)** — solo USD/COP (`config.TICKER_FX = "USDCOP=X"`) vía
  `yfinance` (Frankfurter/BCE no cubre COP). `geografia="COLOMBIA"`.
- **Precio interno (`precio_interno.py`)** — precio de referencia FNC, scraping
  del HTML de estadísticas cafeteras (frágil). `geografia="COLOMBIA"`.
- **Referencia de mercado FNC (`referencia_mercado_fnc.py`)** — extrae de una
  misma publicación diaria el precio FNC, Coffee C y TRM para calibrar el
  simulador sin mezclar proveedores ni horas de cierre. Se persiste aparte en
  `datos/historico/calibracion_fnc.csv`.
- **Producción (`produccion.py`)** — nacional registrada mensual, del Excel FNC,
  en miles de sacos de 60 kg, un punto por mes sin relleno. `geografia="COLOMBIA"`.
- **Exportaciones (`exportaciones.py`)** — volumen mensual exportado, del Excel
  FNC, en miles de sacos de 60 kg de café verde equivalente, un punto por mes
  sin relleno. `geografia="COLOMBIA"`.
- **Clima (`clima.py`)** — Open-Meteo, gratis/sin key, uso no comercial. Una
  coordenada por departamento (`config.REGIONES_CAFE`); `geografia` = departamento.
- **Noticias (`noticias.py`)** — GDELT DOC 2.0 (`gdeltdoc`), gratis/sin key,
  multilingüe, nivel nacional (`config.PAIS_FIPS`). Mezcla fuentes confiables y
  obscuras: filtrar con criterio, nunca una sola noticia como hecho.
- **Contexto (`contexto.py`)** — API Banco Mundial, gratis/sin key. Solo telón de
  fondo anual, no dato semanal.

## 7. Fases del proyecto

Avanzar **en orden**; no pasar de fase hasta que la anterior corra y se verifique.

- **Fase 0 — Esqueleto.** Carpetas, `config.py`, `requirements.txt`,
  `.gitignore`, `main.py` y stubs con `obtener()` vacío. *Verificable:*
  `python main.py` corre sin error.
- **Fase 1 — Fuentes** 1a FX → 1b Café → 1c Clima → 1d Noticias. Cada una corre
  sola mostrando su DataFrame.
- **Fase 2 — Unir.** `unir.py` junta en tabla semanal y guarda el snapshot;
  `calidad.py` valida fechas, duplicados, nulos, valores y cobertura antes de
  guardar (no se sobrescribe sin autorización). `historico.py` construye el
  histórico desde 2023: producción y exportaciones mensuales sin forward-fill,
  solo semanas cerradas para el resto, idempotente.
- **Bloque 3 — Indicadores.** `indicadores.py`: cambios, promedios móviles,
  anomalías y comparaciones entre departamentos. Sin oportunidad/riesgo/juicio.
- **Bloque 3.5 — Preparación visual.** `visualizacion.py`: etiquetas, categorías,
  orden, colores e índice base 100. Capa neutral, sin criterio ni score.
- **Visualizaciones para feedback.** `app.py` muestra panorama comercial y
  simulador. El clima permanece en el pipeline, pero no en la interfaz visible.
- **Simulador.** `proyeccion.py` estima el FNC desde Coffee C y USD/COP con el
  coeficiente implícito del último trío coherente publicado por la FNC. Si esa
  referencia falla, usa una calibración estadística reciente como respaldo. El
  FNC observado sirve para calibrar y comparar, no como piso. Aplica un ajuste
  aproximado por factor de rendimiento; la interfaz edita costo por carga y
  estima margen bruto. No es pronóstico ni modela prima, calidad, logística o
  causalidad.
- **Fase 3 — Score.** Metodología del índice, con datos reales (pendiente).
- **Fase 4 — Reporte.** Brief por periodo con cifras, fuentes, cadencias y
  limitaciones. Salidas reutilizables: libro Excel con resumen, series
  filtrables y diccionario (`reporte/excel.py`), y PDF de tres páginas con
  gráficas (`reporte/pdf.py`); `reporte/generar.py` conserva la versión
  Markdown y el informe del simulador.
- **Fase 5 — Streamlit.** Tablero leyendo los snapshots.
- **Fase 6 — Automatización.** GitHub Actions cada 2 días.
- **Fase 7 — Banco Mundial + pulido para LinkedIn.**

## 8. Cómo ejecutar y verificar

Con el entorno virtual activo (`(.venv)` visible):

```
.venv\Scripts\Activate.ps1               # Windows PowerShell
python -m fuentes.fx                     # probar un módulo aislado
python main.py                           # correr el orquestador
python -m unittest discover -s tests -v  # pruebas sin internet
python -m procesar.historico             # actualizar el histórico desde 2023
python -m procesar.indicadores           # tendencias y resumen reciente
python -m procesar.visualizacion         # preparar series para gráficos
streamlit run app.py                     # tablero en localhost:8501
```

## 9. Disciplina de edición

- Ediciones **quirúrgicas**: lo mínimo necesario.
- **No romper el contrato** de las fuentes (sección 4). Si hay que cambiar el
  esquema, avisar y actualizar este archivo y todos los módulos afectados.
- No mezclar dos fases en un mismo cambio.
- Si una fuente revela una limitación, documentarla en el módulo, no esconderla.
- **Documentación:** tras cada cambio validado, actualizar **solo**
  `CONTEXTO_IAS.md` (si cambió estado, decisión, limitación o próximo paso).
  `README.md`, `ACERCA_DE.md` y `BRIEFING_CHAT.md` se actualizan solo por
  petición manual del usuario. Este `CLAUDE.md` se toca sin pedir permiso únicamente
  ante un cambio crítico (arquitectura, contrato, convención).

## 10. Secretos y API keys

- **Nunca** keys en el código ni en `config.py`. Van en `.env` (en `.gitignore`),
  leídas como variables de entorno (Alpha Vantage en fases tardías; Anthropic si
  se usa para el resumen).
- `.env`, `.venv/` y `__pycache__/` nunca se suben.
