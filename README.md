# domain-squatter

Filtro en tres fases para detectar dominios sospechosos —cybersquatting,
typosquatting y phishing— relacionados con marcas de empresas y sectores
españoles: banca, energía, telecomunicaciones, distribución, transporte,
seguros, industria y medios.

Cada fase es un script independiente. Se ejecutan por separado, leen ficheros de
texto plano y no se llaman entre sí.

## Requisitos

- **Python 3.9 o superior.** Comprueba tu versión con:

  ```bash
  python3 --version
  ```

- **No hay que instalar ninguna librería.** Los tres scripts usan únicamente
  módulos que vienen incluidos en Python: `argparse`, `csv`, `os`, `sys` y
  `unicodedata`.
- No hace falta `pip`, ni un entorno virtual, ni permisos de administrador.

> Los ficheros de datos deben estar en **UTF-8**. Es importante porque la fase 2
> detecta caracteres homóglifos, como una «е» cirílica que se ve igual que una
> «e» latina.

La fase 3 necesita el volcado público de [TweetFeed](https://github.com/0xDanielLopez/TweetFeed).
No se incluye en el repositorio porque pesa varios megas y cambia a diario;
descárgalo antes de ejecutarla:

```bash
curl -sL -o data/year.csv https://raw.githubusercontent.com/0xDanielLopez/TweetFeed/master/year.csv
```

## Estructura del proyecto

```
domain-squatter/
├── keywords-filtering.py      # fase 1: coincidencia por palabras clave
├── levenshtein-filtering.py   # fase 2: distancia de edición
├── phishing-filtering.py      # fase 3: comparación con una lista real
└── data/
    ├── domains.txt         # dominios a analizar (el "feed")
    ├── keywords.txt        # marcas que usa la fase 1
    ├── legitimate.txt      # dominios legítimos que usa la fase 2
    └── year.csv            # lista de phishing de TweetFeed (descarga necesaria)
```

`data/domains.txt` incluye 120 dominios de ejemplo que cubren varias técnicas de cybersquatting.


## Las tres fases

### Fase 1 — Coincidencia por palabras clave

Busca literalmente las marcas de `data/keywords.txt` dentro de los dominios. Es
la fase más rápida y no deja escapar las variantes con guiones o palabras
añadidas, pero es ciega a los errores tipográficos.

```bash
python3 keywords-filtering.py --domains data/domains.txt --keywords data/keywords.txt
```

La salida es una tabla con el dominio, la marca encontrada y el tipo de
coincidencia (`exact`, `contains` o `token-all`). Al terminar, y por la salida de
error, aparece el resumen:

```text
80 of 120 domains flagged
```

Es decir: 80 de los 120 dominios han coincidido con alguna marca, con 82 filas en
la tabla porque un mismo dominio puede coincidir con varias. Los 40 restantes
son las erratas de un carácter y el ruido; ahí es donde entra la fase 2.

| Opción | Por defecto | Para qué sirve |
|---|---|---|
| `--domains` | obligatorio | lista de dominios a analizar (`-` para leer de la entrada estándar) |
| `--keywords` | obligatorio | lista de marcas (`-` para leer de la entrada estándar) |
| `--min-length` | `4` | ignora las palabras clave más cortas que este valor |
| `--show-unmatched` | desactivado | añade al final los dominios que no coincidieron con nada |
| `--csv RUTA` | — | exporta el análisis completo a un CSV |

#### **DEMO** -> [Filtrado por Keywords](/domain-squatter/demo/fase-1-keywords.mov)

### Fase 2 — Distancia de Levenshtein

Compara cada dominio con los dominios legítimos de `data/legitimate.txt` y mide
cuántas ediciones (inserciones, borrados o sustituciones) hacen falta para pasar
de uno a otro. Así detecta lo que la fase 1 no ve: `bbv4.es`, `santandcr.es` o
una «е» cirílica en lugar de una «e».

```bash
python3 levenshtein-filtering.py --domains data/domains.txt --legitimate data/legitimate.txt
```

Salida: una tabla con el dominio, el dominio legítimo al que se parece y la
distancia. Resumen final:

```text
58 of 120 domains matched (35 exact/fuzzy, 23 near miss)
```

Son 35 coincidencias exactas o difusas, y 23 parecidos que se han marcado por
caer dentro de `--watch-distance`.

| Opción | Por defecto | Para qué sirve |
|---|---|---|
| `--domains` | obligatorio | lista de dominios a analizar |
| `--legitimate` | obligatorio | lista de dominios legítimos |
| `--min-length` | `3` | ignora las marcas más cortas que este valor |
| `--max-distance` | `1` | número máximo de ediciones que cuenta como coincidencia |
| `--watch-distance` | `6` | además, marca como `near-miss` lo que esté a esa distancia o menos |
| `--show-unmatched` | desactivado | añade al final los dominios que no se parecen a nada |
| `--csv RUTA` | — | exporta el análisis completo a un CSV |

Hay dos detalles que conviene conocer:

- **El límite real es más estricto que `--max-distance`.** El presupuesto de cada
  marca es `min(--max-distance, longitud de la marca ÷ 2)`. Por eso una marca
  corta como `seur` (4 letras) nunca pasa de una edición, aunque subas
  `--max-distance` a 10. Gracias a eso, `mercadona.es` no se marca contra
  `seur.com` por mucho que subas el umbral: están a distancia 8.
- **`--watch-distance` sirve para no perder análisis.** Los dominios que caen
  justo por encima del presupuesto se marcan igualmente, con el tipo
  `near-miss`, para que puedas revisarlos sin subir el umbral de las
  coincidencias normales. Baja `--watch-distance` si quieres un informe más
  estricto.

Tipos de coincidencia: `identical` (distancia 0), `fuzzy` (dentro del
presupuesto) y `near-miss` (dentro de `--watch-distance`).

#### **DEMO** -> [Filtrado por Distancia de Levenshtein](/domain-squatter/demo/fase-2-levenshtein.mov)


### Fase 3 — Comparación con una lista real de phishing

Comprueba si alguno de nuestros dominios aparece en la lista de dominios que ya
se han visto en campañas reales de phishing. No hay nada difuso aquí: es una
comparación exacta, así que una coincidencia significa que ese dominio existe de
verdad.

```bash
python3 phishing-filtering.py --domains data/domains.txt --phishing data/year.csv
```

Antes de comparar, el script filtra la lista original: se queda solo con las
filas cuyo tipo es `domain` y cuya etiqueta incluye `#phishing`. Del volcado
completo (unas 98.000 líneas) salen 14.813 dominios únicos.

Resumen esperado con los datos de ejemplo:

```text
4 of 120 crafted domains matched against 14813 phishing domains
```

Cada fila de la tabla incluye también el dominio del volcado que ha coincidido:

```text
| movistarz.shop            | phishing | movistarz.shop            |
| santander-seguridad.com   | phishing | santander-seguridad.com   |
| transaccionesmovistar.com | phishing | transaccionesmovistar.com |
| renfe.my.id               | phishing | renfe.my.id               |
```

En este caso las coincidencias son cuatro, porque `data/domains.txt` incluye
cuatro dominios que el volcado real tiene registrados: `renfe.my.id`,
`movistarz.shop`, `santander-seguridad.com` y `transaccionesmovistar.com`. Los
tres primeros también los detectan las fases 1 y 2 (`movistarz.shop`, por
ejemplo, está a una edición de `movistar.es`); la fase 3 no mide parecido, sino
si el dominio ya se ha usado.

| Opción | Por defecto | Para qué sirve |
|---|---|---|
| `--domains` | obligatorio | lista de dominios propios a comprobar |
| `--phishing` | obligatorio | CSV de phishing a leer |
| `--type` | `domain` | valor de la columna *type* que se conserva |
| `--tag` | `phishing` | etiqueta que se conserva (sin la `#`) |
| `--show-unmatched` | desactivado | añade al final los dominios que no están en la lista |
| `--csv RUTA` | — | exporta el veredicto y el dominio de phishing coincidente a un CSV |

Con `--tag scam`, por ejemplo, se comparan las filas etiquetadas como estafa en
lugar de phishing:

```bash
python3 phishing-filtering.py --domains data/domains.txt --phishing data/year.csv --tag scam
```

#### **DEMO** -> [Filtrado por Lista de Phishing](/domain-squatter/demo/fase-3-phishing.mov)


## Cómo leer el CSV

Con `--csv` cada fase exporta un informe completo. La columna `match` vale `Y`
cuando el dominio se ha marcado y `N` cuando no.

| Fase | Columnas |
|---|---|
| 1 | `analyzed-domain, match, keyword, match-type` |
| 2 | `analyzed-domain, match, legitimate, distance, match-type` |
| 3 | `analyzed-domain, match, phishing-domain` |



## Ejemplo completo

Copia y pega los cuatro comandos para ejecutar todo de principio a fin:

```bash
# 0) Descarga el volcado de phishing (solo la primera vez)
curl -sL -o data/year.csv https://raw.githubusercontent.com/0xDanielLopez/TweetFeed/master/year.csv

# 1) Coincidencias literales por marca
python3 keywords-filtering.py --domains data/domains.txt --keywords data/keywords.txt --csv fase1.csv

# 2) Parecidos por distancia de levenshtein
python3 levenshtein-filtering.py --domains data/domains.txt --legitimate data/legitimate.txt --csv fase2.csv

# 3) Presencia en la lista real de phishing
python3 phishing-filtering.py --domains data/domains.txt --phishing data/year.csv --csv fase3.csv
```

Resultado de referencia con los ficheros incluidos en el repositorio:

| Fase | Salida |
|---|---|
| 1 | `80 of 120 domains flagged` |
| 2 | `58 of 120 domains matched (35 exact/fuzzy, 23 near miss)` |
| 3 | `4 of 120 crafted domains matched against 14813 phishing domains` |

Si tus números coinciden, el entorno está bien montado. El volcado de phishing
cambia a diario, así que la cifra de la fase 3 puede variar; las de las fases 1
y 2 solo dependen de los ficheros del repositorio.

## Personalizar las listas

Todos los ficheros de `data/` son texto plano, con un elemento por línea:

- Las líneas en blanco y las que empiezan por `#` se ignoran, así que puedes
  comentar y agrupar entradas.
- Se admite un dominio por línea, o varios separados por comas.
- Se normaliza todo automáticamente: mayúsculas, `http://`, `*.`, rutas y el
  punto final. Así, `HTTP://WWW.Bbva.Es.` se convierte en `www.bbva.es`.
  El prefijo `www.` **no** se elimina: `www.bbva.es` y `bbva.es` son dos
  entradas distintas (en la fase 2 da igual, porque compara cada etiqueta por
  separado).
- Para analizar tus propios dominios, sustitúyelos en `data/domains.txt` o pasa
  otro fichero con `--domains ruta/al/fichero.txt`.


## Solución de problemas

| Mensaje | Causa y solución |
|---|---|
| `error: no usable domain in ...` | Solo en la fase 3: el fichero no tiene ningún dominio válido o solo contiene comentarios. |
| `error: no usable brand in ... at --min-length 3` | Todas las marcas de `legitimate.txt` son más cortas que `--min-length`. Baja el valor o revisa el fichero. |
| `error: no domain row tagged #phishing in ...` | El CSV no es el esperado, o la etiqueta es otra. Comprueba la ruta y usa `--tag` con el valor correcto. |
| `error: only one of --domains or --legitimate can read from stdin` | Solo puedes usar `-` en una de las dos opciones, porque la entrada estándar no se puede repartir. |
| `error: cannot write ...` | No hay permisos para escribir el CSV en esa ruta. |
| La tabla sale vacía | No hay coincidencias. Prueba `--show-unmatched` para ver los dominios no marcados, o sube `--watch-distance` en la fase 2. |

Las fases 1 y 2 son más tolerantes con las listas vacías: no dan error, solo
muestran `0 of 0 ...`. Si aparece un cero donde esperabas resultados, revisa la
ruta de `--domains` y la de la lista de marcas.

Cualquier script muestra todas sus opciones con `--help`:

```bash
python3 levenshtein-filtering.py --help
```
