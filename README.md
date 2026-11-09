# BCV Scraper

Scraper en Python de las **tasas oficiales del BCV** (dólar USD y euro EUR)
publicadas en <https://www.bcv.org.ve/glosario/cambio-oficial>.

Imprime el resultado como JSON, puede generar un **HTML autónomo** con las
tasas y, mediante GitHub Actions, se ejecuta cada hora para mantener
actualizado el archivo `tasas.json` de este repositorio.

## Estructura

| Archivo | Descripción |
|---------|-------------|
| `bcv_scraper.py` | Descarga la página del BCV, extrae USD/EUR y la fecha valor. También genera un HTML autónomo (`--html`). |
| `templates/tasas.html` | Plantilla Jinja2 de la página HTML de tasas. |
| `requirements-tasas.txt` | Dependencias de Python. |
| `test_bcv_scraper.py` | Pruebas sin red del parser y del formato de números. |
| `tasas.json` | Últimas tasas USD/EUR generadas por el scraper (lo actualiza el workflow). |
| `.github/workflows/actualizar-tasas.yml` | Workflow que regenera `tasas.json` cada hora y lo commitea. |

## Instalación

```bash
pip install -r requirements-tasas.txt
```

## Uso

```bash
python3 bcv_scraper.py                          # imprime el JSON de tasas
python3 bcv_scraper.py > tasas.json             # guarda las tasas
python3 bcv_scraper.py --html tasas.html        # genera la página HTML
python3 bcv_scraper.py --url https://...        # consulta otra URL equivalente
```

Salida:

```json
{
  "usd": 897.5383,
  "eur": 1001.18602288,
  "usd_bs": "897,54",
  "eur_bs": "1.001,19",
  "usd_exacta": "897,53830000",
  "eur_exacta": "1.001,18602288",
  "fecha_valor": "2026-11-09",
  "fecha_valor_texto": "Lunes, 09 Noviembre 2026",
  "fuente": "https://www.bcv.org.ve/glosario/cambio-oficial",
  "obtenido_en": "2026-11-08T18:24:38-04:00",
  "actualizado_texto": "08/11/2026 18:24:38"
}
```

## Cómo funciona el scraping

- Toma los bloques `<div id="dolar">` y `<div id="euro">`, lee el
  `<strong class="strong-tb">` y convierte el formato venezolano
  (`1.001,18602288` → `1001.18602288`).
- Lee la **Fecha Valor** del atributo `content` (`dc:date`) y su texto visible.
- Si el HTML cambia, hay un respaldo por expresión regular; si aun así no
  encuentra un valor, lanza `ScrapingError` con un mensaje claro.
- El certificado TLS de `bcv.org.ve` suele estar incompleto: se intenta validar
  y, si falla, se repite la petición sin validar la cadena (la conexión sigue
  cifrada). Se avisa por log.

## Actualización automática (GitHub Actions)

`.github/workflows/actualizar-tasas.yml` corre **cada hora** (minuto 5, UTC)
y también a mano desde **Actions → Actualizar tasas BCV → Run workflow**:

1. Instala `requirements-tasas.txt` y ejecuta `python3 bcv_scraper.py > tasas.json`.
2. Valida que el JSON sea correcto.
3. Hace commit y push como `github-actions[bot]` solo si `tasas.json` cambió
   (mensaje `chore: actualizar tasas BCV`).
4. Si el BCV falla, la ejecución queda en rojo y **no** se commitea: se conserva
   la última tasa buena.

Requiere **Settings → Actions → General → Workflow permissions →
"Read and write permissions"** para que el robot pueda hacer push.

## Pruebas

```bash
python3 -m unittest test_bcv_scraper -v
```

Las pruebas no usan red: validan el parseo de números, el formato venezolano,
la extracción de USD/EUR, la fecha valor y el contexto de la plantilla.
