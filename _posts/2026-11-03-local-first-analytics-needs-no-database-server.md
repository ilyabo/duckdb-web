---
layout: post
title: Local Analytics Needs No Database Server
authors:
  - Ilya Boyandin in conversation with Geertjan Wielenga & Tim Cox
thumb: /images/blog/thumbs/spatial-joins.svg
image: /images/blog/thumbs/spatial-joins.png
excerpt: Ilya Boyandin first used DuckDB in a geospatial side project that analyzed data in the browser without a database server. He later turned that setup into SQLRooms, an open-source toolkit for local analytics applications. Foursquare Spatial Desktop and Cosmograph use SQLRooms for geospatial and graph analysis.
tag: using-duckdb
---

[Ilya Boyandin](https://www.linkedin.com/in/ilyabo/) is a software engineer, the creator of [SQLRooms](https://sqlrooms.org), [FlowmapBlue](https://flowmap.blue), and [Flowmap City](https://flowmap.city/). He worked on geospatial data analytics at Unfolded, the company founded by the team behind the open-source data visualization projects [deck.gl](https://deck.gl) and [kepler.gl](https://kepler.gl), which was later acquired by Foursquare.

## How DuckDB Entered the Picture

Ilya first used DuckDB in [Flowmap City](https://flowmap.city/), a side project for [visualizing mobility flows](https://ilya.boyandin.me/p/flowmap-city) that he still maintains.
Working with a small budget, Ilya wanted to keep Flowmap City easy to operate with minimal infrastructure. DuckDB's [WebAssembly build]({% link docs/2.0/clients/wasm/overview.md %}) lets the app run analytical queries in the user's browser without a database server. Uploaded datasets are stored as Parquet files in S3. When someone opens a project, the app loads the data into DuckDB-Wasm through [presigned HTTPS URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html).

As users pan, zoom, or change filters in Flowmap City, DuckDB-Wasm runs SQL queries in the browser to update the map. A single interaction can trigger a number of queries preparing the data for the current viewport:

<video src="https://ilya.boyandin.me/media/projects/flowmap-city/flowmap-city.mp4" controls autoplay muted loop playsinline style="width: 100%; height: auto;"></video>

**“We can do much of the data analytics and visualization in the browser,”** Ilya says, **“which is very convenient: it has very low infrastructure requirements, and can often be faster because it saves round trips to the server for each query.”**

## From a Side Project to SQLRooms

Much of the core functionality and UI of Flowmap City was not specific to mobility data and could be reused in other projects. So Ilya decided to extract these pieces into a standalone framework called SQLRooms, which he now continues to develop as an open source project.

[SQLRooms](https://sqlrooms.org/) is a modular toolkit for building DuckDB-powered analytics applications in React. It provides building blocks for loading data, writing SQL, displaying tables, creating visualizations, and integrating AI assistants. Developers can combine the pieces they need with their own components to build applications such as a map explorer or a graph analysis tool.

These components work together within a _Room_: a workspace containing datasets, analysis artifacts such as saved queries, charts, maps, and documents, and the application state that connects them. SQLRooms handles query execution through a shared DuckDB connection, so developers can focus on how users explore their data and build on their work.

DuckDB can run directly in the browser through WebAssembly or natively on the user's machine in a desktop application, allowing data analysis without a remote database server. Optional AI assistants can write SQL and use query results to answer questions, while the queries themselves run against the application's DuckDB instance.

Ilya's interest in local-first software shaped the framework. **“You should own your data. It can live on your computer, not everything has to live in the Cloud. It can be synced via the cloud, but you decide when to keep it local and when not.”**

The quickest way in is the [minimal example](https://sqlrooms.org/examples), a small Vite and React app that loads a CSV file and runs SQL queries against it.

## Putting SQLRooms to Work at Foursquare

At Foursquare, Ilya's team has been developing [Foursquare Spatial Desktop](https://ilya.boyandin.me/p/fsq-spatial-desktop), a desktop application for exploring geospatial data built with SQLRooms. It combines DuckDB's analytical engine with [kepler.gl](https://kepler.gl/)'s visualization tools. The app builds on the team's experience with kepler.gl and [Foursquare Studio](https://studio.foursquare.com), its cloud-hosted geospatial analytics platform.

**“For this particular product, DuckDB embedded into the desktop app is just the perfect fit,”** he says. **“It enables analysis workflows which users can execute locally.”** SQLite is embeddable too, he says, but it is not column-oriented and was not designed for analytics on larger datasets.

![Foursquare Spatial Desktop](https://ilya.boyandin.me/media/projects/fsq-spatial-desktop/fsq-spatial-desktop-earthquakes.webp)

With DuckDB embedded in the app, queries against local data run on the user's machine without a round trip to a remote database server. Modern laptops are powerful, Ilya adds, and their GPUs are good at visualization. **“I’m seeing renewed interest in applications that keep data and computation on the user’s own machine.”**

### Dynamic Tiling in Spatial Desktop

Spatial Desktop uses [deck.gl](https://deck.gl/) to render maps, but drawing millions of features at once can slow things down. To keep large datasets interactive, the app uses DuckDB’s [spatial extension]({% link docs/2.0/core_extensions/spatial/overview.md %}) to divide the data into tiles and reduce the detail shown at each zoom level.

Map tiles are often prepared in advance for fast loading. Spatial Desktop can also generate tiles from users’ own datasets on demand, using the local DuckDB database, so users can start exploring without preparing a tileset first. For each requested tile, it selects nearby features and takes a random sample to keep dense tiles manageable. It also simplifies lines and polygons according to the zoom level, retaining more detail as the user zooms in.

<video src="https://ilya.boyandin.me/media/projects/fsq-spatial-desktop/fsq-spatial-desktop-dynamic-tiling-4-1500.mp4" controls autoplay muted loop playsinline style="width: 100%; height: auto;"></video>

The following simplified query shows the main steps. It assumes the source geometries are already in Web Mercator coordinates (`EPSG:3857`). The tile width determines both a small buffer around the search area and a simplification tolerance equivalent to roughly half a pixel:

```sql
WITH tile AS (
    SELECT
        ST_TileEnvelope(⟨z⟩, ⟨x⟩, ⟨y⟩) AS bounds,
        (ST_XMax(bounds) - ST_XMin(bounds)) / 256.0 AS meters_per_pixel
),
candidates AS (
    SELECT
        f.id, f.geometry, tile.bounds,
        GREATEST(meters_per_pixel * 0.5, 0.1) AS tolerance
    FROM features f, tile
    WHERE ST_Intersects_Extent(
        f.geometry,
        ST_Buffer(tile.bounds, meters_per_pixel * 16)
    )
),
sampled AS (
    SELECT *
    FROM candidates
    USING SAMPLE reservoir(50000 ROWS)
),
encoded AS (
    SELECT
        id,
        ST_AsMVTGeom(
            ST_SimplifyPreserveTopology(geometry, tolerance),
            ST_Extent(bounds)
        ) AS geom
    FROM sampled
)
SELECT ST_AsMVT(encoded, 'features') AS tile
FROM encoded
WHERE geom IS NOT NULL;
```

`ST_AsMVTGeom` converts the simplified geometries to tile coordinates and clips them; [`ST_AsMVT`]({% link docs/2.0/core_extensions/spatial/functions.md %}#st_asmvt) packages them and their attributes into a Mapbox Vector Tile (MVT). The production implementation also handles invalid geometries, filters out very small lines and polygons, and caches generated tiles.

An HTTP server running locally as part of the Electron desktop app serves tiles by their zoom level and tile coordinates (`z/x/y`). As the user pans and zooms, the map requests the tiles it needs, showing a sampled, simplified view of datasets too large to render in full.

## Graph Analytics on the Fly with Cosmograph

[Cosmograph](https://ilya.boyandin.me/p/cosmograph) is another project built on the SQLRooms framework. It's a tool for graph analytics, developed by [Nikita Rokotyan](https://rokotyan.com/) and Thor Whalen; Ilya helped with the SQLRooms integration. It allows users to visualize graphs with millions of edges directly in the browser and use SQLRooms features, such as an AI assistant that generates SQL queries, to analyze their data.

<video src="https://ilya.boyandin.me/media/projects/cosmograph/cosmograph-2.0.mp4"
       controls autoplay muted loop playsinline style="width: 100%; height: auto;"></video>

Cosmograph has been tested with datasets ranging from tens of thousands of points to several million. DuckDB-Wasm handles data loading and SQL queries in a background worker, helping keep the interface responsive. It lets the app filter and summarize data as users explore, while columnar query results reduce the conversion work needed to prepare data for visualization.

## When the Agent Writes the SQL

The SQLRooms integration also brought AI-assisted analysis to Cosmograph. **“This combination of SQL and AI agents is actually quite powerful,”** Ilya says. **“SQL has such a long history, so many documents on the web explaining how to use it, and LLMs are pretty good at generating it.”** Users ask a question and the agent writes and runs the DuckDB queries, so they don't need to know SQL.

<video src="https://ilya.boyandin.me/media/projects/sqlrooms/sqlrooms-cars-document.mp4"
       controls autoplay muted loop playsinline style="width: 100%; height: auto;"></video>

The SQLRooms-powered assistant generates SQL queries and chart specifications, which the application executes and renders locally. This allows the source dataset to stay on the user's machine, while database schemas and selected query results can be shared with the AI provider to help answer questions.

## One Database File, One Analytical Project

In Cosmograph, Ilya explains, **“The project file you create is actually a DuckDB database file, in which we store both the data users add to the project and all the artifacts they create within it.”**

A whole project then lives in one file that a user can move around like any other document.

## Where SQLRooms Goes Next

The project file in Cosmograph points to a broader idea that Ilya is developing in SQLRooms, where the DuckDB file carries the analysis as well as the data. He is building a Canvas module with SQL query cells and Vega-Lite chart cells, alongside a Notebooks module. A dependency engine tracks how cells depend on each other and reruns downstream cells when the data changes.

At [DuckCon #7]({% link _events/2026-06-24-duckcon7.html %}) in Amsterdam, Ilya showed a SQLRooms command-line tool that works with a DuckDB database and presents the analysis in a notebook-like style. The direction is a single DuckDB file that holds an entire analytical workspace, which someone can open elsewhere and continue. The design is still changing.

## Watch the DuckCon Talk

Ilya's five-minute talk from DuckCon #7 gives a short tour of this direction.

<div class="video-container">
<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/fV1LXnPl4ng" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
</div>

SQLRooms is open source and available at [sqlrooms.org](https://sqlrooms.org), where the example apps are a good place to start. [Flowmap City](https://flowmap.city/), [Foursquare Spatial Desktop](https://foursquare.com/products/spatial-desktop/), [Cosmograph](https://cosmograph.app) are free to try.

We would like to thank Ilya for the conversation. You can follow his writing on DuckDB and SQLRooms on [his blog](https://ilya.boyandin.me/). And if you are reading this, are using DuckDB, and would like to share your story, drop us a line at [quack@duckdb.org](mailto:quack@duckdb.org).
