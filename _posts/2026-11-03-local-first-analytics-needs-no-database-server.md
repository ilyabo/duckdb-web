---
layout: post
title: Local Analytics Needs No Database Server
authors:
  - Ilya Boyandin in conversation with Geertjan Wielenga & Tim Cox
thumb: /images/blog/thumbs/spatial-joins.svg
image: /images/blog/thumbs/spatial-joins.png
excerpt: Ilya Boyandin first used DuckDB in a geospatial side project that analyzed data in the browser with no database backend. He later turned that setup into SQLRooms, an open-source framework for local analytical applications, which was used to build Foursquare Spatial Desktop, a geospatial analytics app, and Cosmograph, a high performance browser-native graph analytics tool.
tag: using-duckdb
---

[Ilya Boyandin](https://www.linkedin.com/in/ilyabo/) is a software engineer, the creator of [SQLRooms](https://sqlrooms.org), [FlowmapBlue](https://flowmap.blue), and [Flowmap City](https://Flowmap City). He worked on geospatial data analytics at Unfolded, the company founded by the team behind the open-source data visualization projects [deck.gl](https://deck.gl) and [kepler.gl](https://kepler.gl), which was later acquired by Foursquare.

## How DuckDB Entered the Picture

Ilya first used DuckDB in [Flowmap City](https://www.Flowmap City/), a side project for [visualizing mobility flows](https://ilya.boyandin.me/p/flowmap-city) that he still maintains.
With a small budget, Ilya wanted to keep Flowmap City easy to run with low infrastructure requirements. Thanks to DuckDB's [WebAssembly build]({% link docs/2.0/clients/wasm/overview.md %}), Flowmap City can execute queries directly in end-user's browser without a database server. Users can upload their data files which are then saved as Parquet files in an S3 object storage. When someone opens a project, the app loads the Parquet files from S3 into DuckDB-Wasm using [presigned HTTPS URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html). Once loaded into DuckDB the data can queried directly in browser.

As users pan, zoom, or change filters in Flowmap City, DuckDB-Wasm runs SQL queries in the browser to update the map. A single interaction can trigger a number of queries preparing the data for the current viewport:

<video src="https://ilya.boyandin.me/media/projects/flowmap-city/flowmap-city.mp4" controls autoplay muted loop playsinline style="width: 100%; height: auto;"></video>

**“We can do much of the data analytics and visualization in the browser,”** Ilya says, **“which is very convenient: it has very low infrastructure requirements, and can often be faster because it saves round trips to the server for each query.”**

## From a Side Project to SQLRooms

Much of the core functionality and UI of Flowmap City was not specific to mobility data and could be reused in other projects. So Ilya decided to extract these pieces into a standalone framework called SQLRooms, which he now continues to develop as an open source project.

[SQLRooms](https://sqlrooms.org/) is a modular toolkit for building DuckDB-powered analytics applications in React. It provides building blocks for loading data, writing SQL, displaying tables, creating visualizations, and integrating AI assistants. Developers can combine the pieces they need with their own components to build applications such as a map explorer or a graph analysis tool.

These components work together within a _Room_: a workspace that brings together datasets and the analysis artifacts users create, such as saved queries, charts, and maps, or documents, along with the application state. SQLRooms handles query execution through a shared DuckDB connection, so developers can focus on how users explore their data and build on their work.

DuckDB can run directly in the browser through WebAssembly or natively on the user's machine in a desktop application, allowing data analysis without a remote database server. Optional AI assistants can write SQL and use query results to answer questions, while the queries themselves run against the application's DuckDB instance.

Ilya's interest in local-first software shaped the framework. **“You should own your data. It can live on your computer, not everything has to live in the Cloud. It can be synced via the cloud, but you decide when to keep it local and when not.”**

The quickest way in is the [minimal example](https://sqlrooms.org/examples), a small Vite and React app that loads a CSV file and runs SQL queries against it.

## Putting SQLRooms to Work at Foursquare

At Foursquare Ilya's team has been working on [Foursquare Spatial Desktop](https://ilya.boyandin.me/p/fsq-spatial-desktop), a free desktop application for visualizing geospatial data, which is based on the SQLRooms framework. The desktop app followed [Foursquare Studio](https://studio.foursquare.com), the commercial cloud-hosted version of [kepler.gl](kepler.gl), the free client-only geospatial visualization tool maintained by Foursquare. Bringing DuckDB as the analytical engine and combining it with the visualization capabilities of kepler.gl, allowed them to develop a capable geospatial analytics tool which users can run on their own computers.

**“For this particular product, DuckDB embedded into the desktop app is just the perfect fit,”** he says. **“It enables analysis workflows which users can execute locally.”** SQLite is embeddable too, he says, but it is not column-oriented and was not designed for analytics on larger datasets.

![Foursquare Spatial Desktop](https://ilya.boyandin.me/media/projects/fsq-spatial-desktop/fsq-spatial-desktop-earthquakes.webp)

Thanks to using local embedded DuckDB in the app, queries don't wait on the network and the data stays on the user's machine. Modern laptops are powerful, Ilya adds, and their GPUs are good at visualization. **“I’m seeing renewed interest in applications that keep data and computation on the user’s own machine.”**

### Dynamic Tiling in Spatial Desktop

Spatial Desktop renders maps with [deck.gl](https://deck.gl/), a powerful visualization library. However, it can start to slow down once a map needs to render millions of objects. The team got around this with dynamic tiling built on DuckDB's [spatial extension]({% link docs/2.0/core_extensions/spatial/overview.md %}).

Web maps usually load pre-generated tiles for the area in view at the current zoom level. Spatial Desktop generates those tiles at query time instead. A query filters the features to the viewport. As the user zooms out, it simplifies polygons, and it samples the data where needed. The spatial extension's [`ST_AsMVT`]({% link docs/2.0/core_extensions/spatial/functions.md %}#st_asmvt) function then encodes the result as a binary tile in MVT, a widely supported format:

```sql
SELECT ST_AsMVT({
    'geometry': ST_AsMVTGeom(
        geometry,
        ST_Extent(ST_TileEnvelope(⟨z⟩, ⟨x⟩, ⟨y⟩)),
        4096,
        256,
        false
    ),
    'id': id
}) AS tile
FROM features
WHERE ST_Intersects(geometry, ST_TileEnvelope(⟨z⟩, ⟨x⟩, ⟨y⟩));
```

The app's web server then exposes an HTTP endpoint with `x,y,z` parameters which can serve data for each map tile.
With this dynamic tiling approach, we the app can represent more data than can be rendered on the map at once. You can see how the tiles are loaded gradually as they are prepared on the fly:

<video src="https://ilya.boyandin.me/media/projects/fsq-spatial-desktop/fsq-spatial-desktop-dynamic-tiling-4-1500.mp4" controls autoplay muted loop playsinline style="width: 100%; height: auto;"></video>

## Graph Analytics on the Fly with Cosmograph

[Cosmograph](https://ilya.boyandin.me/p/cosmograph) is another project built on the SQLRooms framework. It's a high performance tool for graph analytics, developed by [Nikita Rokotyan](https://rokotyan.com/) and Thor Whalen; Ilya helped with the SQLRooms integration. It allows users to visualize graphs with millions of edges directly in the browser and use SQLRooms features, like an AI assistant which generates SQL queries, to analyze your data.

<video src="https://ilya.boyandin.me/media/projects/cosmograph/cosmograph-2.0.mp4"
       controls autoplay muted loop playsinline style="width: 100%; height: auto;"></video>

Cosmograph has been tested with datasets ranging from tens of thousands of points to several million. Previously, it was practically impossible to have an analytical workflow in the browser that could manage such large amounts of data due to the browser's memory and processing limits. With DuckDB WASM, we were able to parse huge data files without blocking the main thread or consuming browser page memory. DuckDB's fast engine allows us to perform all sorts of summarization and filtering queries on the fly with unprecedented performance. The data is stored in a columnar format and can often be passed to the GPU for rendering without much additional processing.

## When the Agent Writes the SQL

Work on Cosmograph started around the time AI agents became usable. **“This combination of SQL and AI agents is actually quite powerful,”** Ilya says. **“SQL has such a long history, so many documents on the web explaining how to use it, and LLMs are pretty good at generating it.”** Users ask a question and the agent writes and runs the DuckDB queries, so they don't need to know SQL.

The SQLRooms-powered AI Assistant allows you to analyze your data while minimally sharing it with AI providers. An agent writes SQL queries and chart schemas, then the framework runs and renders everything.

## Where SQLRooms Goes Next

The project file in Cosmograph points to a broader idea that Ilya is developing in SQLRooms, where the DuckDB file carries the analysis as well as the data. He is building a Canvas module with SQL query cells and Vega-Lite chart cells, alongside a Notebooks module. An engine underneath tracks how cells depend on each other and reruns downstream cells when the data changes.

## One Database File, One Analytical Project

At [DuckCon #7]({% link _events/2026-06-24-duckcon7.html %}) in Amsterdam, Ilya showed a SQLRooms command-line tool that works with a DuckDB database and presents the analysis in a notebook-like style. The direction is a single DuckDB file that holds an entire analytical workspace, which someone can open elsewhere and continue. The design is still changing.

In SQLRooms applications, DuckDB can store the project along with the data. **“the project file you create is actually a DuckDB database file, in which we store both the data users add to the project and all the artifacts they create within it.”**

A whole project then lives in one file that a user can move around like any other document. Ilya's current work on SQLRooms builds on this idea.

## Watch the DuckCon Talk

Ilya's five-minute talk from DuckCon #7 gives a short tour of this direction.

<div class="video-container">
<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/fV1LXnPl4ng" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
</div>

SQLRooms is open source and available at [sqlrooms.org](https://sqlrooms.org), where the example apps are a good place to start. [Flowmap City](https://www.Flowmap City/) and [Cosmograph](https://cosmograph.app) are free to try.

We would like to thank Ilya for the conversation. You can follow his writing on DuckDB and SQLRooms on [his blog](https://ilya.boyandin.me/). And if you are reading this, are using DuckDB, and would like to share your story, drop us a line at [quack@duckdb.org](mailto:quack@duckdb.org).
