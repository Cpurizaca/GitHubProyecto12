# GitHubProyecto12 - Movies Smart Data

Proyecto de smart data relacionado a películas, incluyendo información de directores, votos, géneros, presupuesto y ganancias. Elaborado bajo la arquitectura Medallion (Bronze, Silver, Golden) en Azure Databricks con Unity Catalog, utilizando 3 fuentes de datos (CSV), creando funciones para categorizar cada película por votos y por resultado en taquilla, para finalmente mostrar información relevante para el negocio en un dashboard.

## Arquitectura

```
ADLS Gen2 (adlproyecto12/raw)          Unity Catalog: catalog_movies
  Movies.csv        ──┐
  FilmDetails.csv   ──┼──> bronze ──> silver ──> golden ──> Dashboard / Delta Sharing
  MoreInfo.csv      ──┘
```

* **Storage:** Azure Data Lake Storage Gen2 (`adlproyecto12`), contenedores `raw`, `bronze`, `silver`, `golden`.
* **Gobierno:** Unity Catalog con Storage Credential y External Locations; catálogo `catalog_movies`.
* **Cómputo:** cluster `ClusterProyecto12`.
* **Orquestación:** Lakeflow Job `WF_ADB`, desplegado automáticamente con GitHub Actions.

## Fuentes de datos

| Archivo | Contenido |
| --- | --- |
| `Movies.csv` | Título, géneros, idioma, puntuación, duración, fecha de estreno, cantidad de votos |
| `FilmDetails.csv` | Director, reparto principal, presupuesto y recaudación (USD) |
| `MoreInfo.csv` | Duración, presupuesto y recaudación en formato texto (`$25,000,000`) |

## Estructura del repositorio

| Carpeta | Contenido |
| --- | --- |
| `PrepAmb/` y `proceso/1.Preparacion_Ambiente` | Creación del catálogo, esquemas, external locations y tablas |
| `proceso/` | Notebooks del pipeline ETL (ingesta, transformación, carga y permisos) |
| `seguridad/` | Grants sobre las capas Medallion |
| `reversion/` | `1. Drop-Medallion`: elimina tablas y datos de cada capa para revertir el ambiente |
| `delta-sharing/` | Publicación de tablas con Delta Sharing (`MOVIES_SHARE`) |
| `dashboard/` | Dashboard AI/BI "Movie Dashboard" e imagen de referencia |
| `datasets/` | Archivos CSV de origen |
| `evidencias/` | Capturas de recursos Azure, metastore, credenciales y external locations |
| `.github/workflows/` | `deploy-notebook.yml`: CI/CD hacia el workspace destino |

## Pipeline ETL

| Paso | Notebook | Origen | Destino |
| --- | --- | --- | --- |
| 1 | `1.Preparacion_Ambiente` | - | Catálogo, esquemas y tablas |
| 2 | `2.Ingest_Movie_data` | `raw/Movies.csv` | `catalog_movies.bronze.movies` |
| 2 | `2.Ingest_MovieDetails_data` | `raw/FilmDetails.csv` | `catalog_movies.bronze.moviedetails` |
| 2 | `2.Ingest_MovieMoreInfo_data` | `raw/MoreInfo.csv` | `catalog_movies.bronze.moviemoreinfo` |
| 3 | `3.Transform` | `bronze.movies`, `bronze.moviedetails` | `catalog_movies.silver.movies_transformed` |
| 4 | `4.Load` | `silver.movies_transformed` | `catalog_movies.golden.golden_movies` |
| 5 | `5.Grants_Medallion` | - | Permisos por capa |

Los notebooks son parametrizados con widgets (`catalogo`, `esquema`, `container`, `storageName`).

### Transformaciones (Silver)

* Limpieza de nulos y `trim` de título y géneros.
* Cálculo de `release_year`, `total_minutos` (horas * 60 + minutos) y `profit` (revenue - budget).
* Conversión de presupuesto y recaudación a `DECIMAL(36,2)`.
* Join entre películas y detalles por `movie_id`.

### Reglas de negocio

**Categoría por votos (`vote_category`)** - UDF `categorizar_votos`:

| Votos | Categoría |
| --- | --- |
| >= 100,000 | Blockbuster / Mass-market phenomenon |
| >= 10,000 | Popular / Commercial success |
| >= 1,001 | Moderately well-known |
| >= 100 | Independent / Niche film |
| > 0 | Very obscure / No audience |
| 0 | 0 votes |

**Resultado en taquilla (`status_ticket`)**:

| Condición | Estado |
| --- | --- |
| revenue >= 2.5 x budget | Commercial Success 🔥 |
| revenue < budget | Failure / Loss 📉 |
| Otro caso | Break-even Point ⚖️ |
| Sin presupuesto | NULL |

## Dashboard

"Movie Dashboard" (AI/BI) sobre `catalog_movies.golden.golden_movies`, con filtro por año de estreno:

* KPIs: Total Movies, Total Directors, Total Genres.
* Top 10 Genres by Number of Movies.
* Top 10 Directors by Profit.
* Top 20 Movies by Profit, incluyendo Ticket Status, duración, presupuesto, recaudación y votos.

![Dashboard](dashboard/Dashboard_image.png)

## CI/CD

El workflow `.github/workflows/deploy-notebook.yml` se ejecuta con cada push a `main`:

1. Exporta los notebooks de `proceso/` desde el workspace origen.
2. Los importa en `/Proyecto/Movies/` del workspace destino.
3. Elimina el job `WF_ADB` si existe y lo recrea con las tareas encadenadas sobre `ClusterProyecto12`.

Secrets requeridos en GitHub: `DATABRICKS_ORIGIN_HOST`, `DATABRICKS_ORIGIN_TOKEN`, `DATABRICKS_DEST_HOST`, `DATABRICKS_DEST_TOKEN`.
