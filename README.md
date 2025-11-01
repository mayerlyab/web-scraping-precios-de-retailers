# web-scraping-precios-de-retailers
Automatization para extraer los precios de la competencia y especificaciones en el top 3 de retailers de Colombia
Web Scraping AC LATAM — Resumen técnico (Septiembre)

  Propósito: Documentar, de forma ejecutiva y reproducible, cómo
  funcionan los scrapers, el pipeline de limpieza/consolidación y los
  análisis de septiembre para monitorear precios de aires acondicionados
  (mini‑split) en retailers de LATAM. Este README está listo para
  publicarse en GitHub.

------------------------------------------------------------------------

🎯 Objetivos del proyecto

-   Extraer precio regular y precio promoción de páginas de producto de
    retailers.
-   Estandarizar campos clave: Retailer, Marca, Capacidad (BTU), Voltaje
    (110/115/220V), Tecnología (Inverter/On‑Off), Modelo, Moneda,
    Precio_Normal, Precio_Promo, URL.
-   Consolidar snapshots mensuales (ej. Julio vs Septiembre) y generar
    tableros/CSV para análisis.
-   Exportar entregables en Excel/CSV para pricing y benchmarking
    competitivo.

------------------------------------------------------------------------

🗂️ Estructura sugerida del repo

    ac-latam-scraping/
    ├─ scrapers/
    │  ├─ alkosto_scraper.py
    │  ├─ exito_scraper.py
    │  ├─ olimpica_scraper.py
    │  ├─ ... (otros retailers por país)
    │  └─ utils_selectors.py
    ├─ cleaners/
    │  ├─ normalize_fields.py
    │  ├─ dedupe_merge.py
    │  └─ currency_fx.py
    ├─ analytics/
    │  ├─ colombiaretailmkt.py     # análisis exploratorio de septiembre (subido)
    │  └─ notebooks/               # opcional (EDA)
    ├─ data/
    │  ├─ raw/                     # snapshots crudos (YYYYMMDD_*.csv)
    │  ├─ interim/                 # limpiezas intermedias
    │  └─ processed/               # consolidados (mensuales)
    ├─ outputs/
    │  ├─ excel/
    │  ├─ charts/
    │  └─ csv/
    ├─ .env.example
    ├─ requirements.txt
    └─ README.md

  Nota: Los nombres de archivo exactos pueden variar. Este README resume
  cómo funciona cada pieza en el pipeline de septiembre.

------------------------------------------------------------------------

⚙️ Requisitos

-   Python 3.10+
-   Navegadores y drivers si usas Selenium; browsers instalados si usas
    Playwright.
-   Paquetes principales:
    -   pandas, numpy
    -   requests, beautifulsoup4 (para páginas simples)
    -   selenium o playwright (para páginas dinámicas)
    -   lxml, selectolax (parsing rápido)
    -   tenacity (reintentos)
    -   matplotlib, seaborn (gráficas en análisis)

Instalación rápida:

    pip install -r requirements.txt
    # Playwright (si aplica)
    python -m playwright install

Variables de entorno (.env):

    # Ejemplo
    HEADLESS=true
    MAX_RETRIES=3
    REQUEST_TIMEOUT=20
    PROXY=
    USER_AGENT="Mozilla/5.0 (Windows NT 10.0; Win64; x64)..."

------------------------------------------------------------------------

🧪 Cómo correr el pipeline (Septiembre)

1.  Scraping por retailer (país):

        python scrapers/alkosto_scraper.py --out data/raw/2025-09-xx_alkosto.csv
        python scrapers/exito_scraper.py   --out data/raw/2025-09-xx_exito.csv
        # ...

2.  Normalización y consolidación:

        python cleaners/normalize_fields.py  --in data/raw --out data/interim/2025-09_normalized.csv
        python cleaners/dedupe_merge.py     --in data/interim/2025-09_normalized.csv                                        --out data/processed/2025-09_consolidated.csv

3.  Análisis (gráficas/CSV):

        # Script de análisis incluido
        python analytics/colombiaretailmkt.py

------------------------------------------------------------------------

🧩 Scrapers de septiembre (comportamiento y salidas)

Cada scraper entrega un CSV con este esquema mínimo (cabeceras):

    Retailer, Country, Brand, Model, Capacity, Voltage, Tech, Price_Normal, Price_Promo, Currency, Price_Type, URL, Timestamp

-   Price_Type: Normal o Promo (cuando el retailer separa
    explícitamente).
-   Capacity: número en BTU (ej. 9000, 12000, 18000, 24000).
-   Voltage: valores detectados del título/descripción (110/115/220V).
-   Tech: Inverter o On-Off.
-   Currency: moneda local del retailer; FX se maneja en
    cleaners/currency_fx.py.

🇨🇴 Colombia (ejemplos)

scrapers/alkosto_scraper.py

-   Tipo de página: dinámica con lazy loading e infinite scroll parcial.
-   Técnicas: Playwright/Selenium + esperas por selectores de tarjeta de
    producto.
-   Extracción: Título, Marca, Modelo (si aparece), Precio lista, Precio
    promo, URL, badges (Inverter).
-   Voltaje: regex desde título/descripción (110V, 115V, 220V).
-   Salida: data/raw/2025-09-xx_alkosto.csv con Price_Type =
    Normal/Promo diferenciados.
-   Riesgos/Anti‑bot: throttling y bloqueos por frecuencia; usar
    HEADLESS=true, USER_AGENT y sleep jitter.

scrapers/exito_scraper.py

-   Tipo de página: grid paginado con precios y badges de descuento.
-   Extracción: Igual a Alkosto; validar promo cuando hay % OFF.
-   Notas: Algunos modelos comparten PDP con variantes; normalizar por
    Model + Capacity.

scrapers/olimpica_scraper.py

-   Tipo de página: variantes por marca/capacidad en listados mixtos.
-   Extracción: Título, Marca, Capacidad, Tecnología; promo detectada
    por sello de oferta.
-   Voltaje: desde título (ej. 12.000 BTU 220V).

  Otros países (RD, VE, Guyana, Surinam) siguen el mismo patrón: scraper
  por retailer → CSV local → normalización → consolidado mensual.
  Ajustar selectors y waits según cada UI.

------------------------------------------------------------------------

🧹 Limpieza y consolidación (septiembre)

cleaners/normalize_fields.py

-   Propósito: Uniformizar cabeceras y valores para merge.
-   Acciones clave:
    -   strip + lower en cabeceras; mapping a estándar (Price_Normal,
        Price_Promo, etc.).
    -   Regex para Capacity (BTU) y Voltage (110/115/220).
    -   Normalización de Brand (tabla de sinónimos: LG = LG Electronics,
        MABE/Mabe, etc.).
    -   Cálculo de RRP_USD (si hay Currency + TRM).
-   Salida: data/interim/2025-09_normalized.csv

cleaners/dedupe_merge.py

-   Propósito: Deduplicar y unir por clave [Retailer, Brand, Model,
    Capacity, Voltage, Tech].
-   Acciones: drop_duplicates, prioridad a filas con Price_Promo válido
    y fecha más reciente.
-   Salida: data/processed/2025-09_consolidated.csv

cleaners/currency_fx.py (opcional)

-   Propósito: Aplicar TRM/FX del día al campo RRP_USD.
-   Entradas: Currency, Local_Price.
-   Salida: agrega RRP_USD y TRM_Date.

------------------------------------------------------------------------

📊 Análisis de septiembre — analytics/colombiaretailmkt.py

  Archivo incluido por ti y usado en septiembre. Resumen funcional:

Entradas: - datacomarketv3.csv (consolidado ya normalizado; contiene
Price_Type, RRP/TRM, RETIQ, Brand, Capacity, Estimate, etc.).

Pasos principales: 1. Estandarización de columnas (quita saltos de
línea/espacios) y rename: - RRP → RRP_COP - TRM → RRP_USD 2. Split por
tipo de precio: normal_prices vs promo_prices usando Price_Type. 3.
Gráfico Capacities vs RRP USD (scatter Normal vs Promo). 4. Merge por
Capacity entre Normal y Promo; cálculo de: -
Diferencia = RRP_USD_Normal - RRP_USD_Promo -
%Diferencia = (Diferencia / RRP_USD_Normal) * 100 - Gráfica de
diferencia absoluta y relativa por capacidad. 5. Promedios por Capacity
y RETIQ (barras de RRP_USD). 6. Clase C: groupby para Estimate →
promedio, mínimo, máximo + gráficas y data labels. 7. Clases C vs E:
groupby por Brand, RETIQ, Capacity → barras de Precio_Promedio. 8. Clase
E por Marca/Capacidad: barras de Precio_Promedio (RRP_USD). 9. Función
plot_min_max(...) (reutilizable): genera barras de mínimo/máximo por
Brand y Capacity, tanto para RRP_USD como Estimate.

Salidas/artefactos generados: - CSV:
brand_capacity_metrics_clase_c_e.csv - CSV:
class_e_metrics_rrp_usd.csv - Gráficas: scatter, line (diferencias), bar
(promedios por clase, min/max por marca/capacidad).

Dependencias clave: pandas, matplotlib, seaborn.

  Este script está orientado al análisis comparativo (no al scraping) y
  parte de un dataset consolidado (septiembre).

------------------------------------------------------------------------

🧾 Esquema de datos (mínimo requerido)

  ------------------------------------------------------------------------
  Campo                          Tipo              Descripción
  ------------------------------ ----------------- -----------------------
  Retailer                       string            Nombre del retailer

  Country                        string            País del retailer

  Brand                          string            Marca del AC

  Model                          string            Modelo si disponible

  Capacity                       int               Capacidad en BTU (9000,
                                                   12000, 18000, 24000,
                                                   etc.)

  Voltage                        string            110/115/220V (extraído
                                                   del título/descripción)

  Tech                           string            Inverter / On-Off

  Price_Normal                   float             Precio regular en
                                                   moneda local

  Price_Promo                    float             Precio promoción en
                                                   moneda local

  Currency                       string            Moneda local

  RRP_USD                        float             Precio referencial en
                                                   USD (si aplica TRM)

  Price_Type                     string            Normal / Promo

  URL                            string            Enlace a la PDP

  Timestamp                      datetime          Fecha/hora de captura
  ------------------------------------------------------------------------

------------------------------------------------------------------------

🧯 Troubleshooting (septiembre)

-   Páginas con scroll infinito: usar scroll step y wait_for_selector
    por card.
-   Precios dinámicos/JS: preferir Playwright; capturar XHR si la API es
    pública.
-   Duplicados por variante: normalizar Model y Capacity; dedupe por
    clave.
-   Promos implícitas (% OFF): calcular Price_Promo a partir de
    Price_Normal y el % si el valor no aparece directo.
-   Bloqueos: rotar USER_AGENT, introducir jitter y limitar QPS; usar
    MAX_RETRIES.

------------------------------------------------------------------------

📌 Buenas prácticas usadas en septiembre

-   Regex robustas para BTU y Voltaje desde títulos con formato libre.
-   Clasificador simple para Inverter/On‑Off por keywords y badges.
-   Convenciones de archivo: YYYY-MM-DD_retailer.csv y
    YYYY-MM_consolidated.csv.
-   Exportables Excel/CSV listos para negocio (pricing y benchmark).

------------------------------------------------------------------------

🗺️ Roadmap corto

-   Agregar pruebas unitarias de selectors y parsers.
-   Headless CI (GitHub Actions) para snapshots semanales.
-   Conector a Google Sheets para compartir resultados en vivo.
-   Detección de WiFi/R32/RETIQ más precisa por ficha técnica JSON‑LD.

------------------------------------------------------------------------

📣 Créditos

-   Líder del proyecto: Mayerly Alviarez (AUX LATAM).
-   Última actualización: Septiembre 2025.
