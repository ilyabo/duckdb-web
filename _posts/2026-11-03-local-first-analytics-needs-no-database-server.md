---
layout: post
title: Local-First Analytics Needs No Database Server
authors:
  - Ilya Boyandin in conversation with Geertjan Wielenga & Tim Cox
thumb: /images/blog/thumbs/spatial-joins.svg
image: /images/blog/thumbs/spatial-joins.png
excerpt: Ilya Boyandin first used DuckDB in a geospatial side project that analyzed data in the browser with no database backend. He later turned that setup into SQLRooms, an open-source framework for local-first analytical applications, and Cosmograph, a browser-native graph analytics tool.
tag: using-duckdb
---

[Ilya Boyandin](https://www.linkedin.com/in/ilyabo/) is a software engineer, the creator of [SQLRooms](https://sqlrooms.org) and co-creator of [Cosmograph](https://cosmograph.app). He has a PhD in geospatial data visualization. He worked on urban mobility analytics at Teralytics before joining Unfolded, the company founded by the Uber visualization team behind the open-source projects [deck.gl](https://deck.gl) and [kepler.gl](https://kepler.gl). ~~Foursquare later acquired Unfolded, and he has worked there since.~~

## How DuckDB Entered the Picture

At Teralytics, Ilya's team used ClickHouse. Like DuckDB, it is columnar and built for analytics, but it usually runs on provisioned servers in a cluster. **“ClickHouse can be used locally, but it's not really meant for that, unlike DuckDB.”**

He first used DuckDB in [Flowmap.city](https://www.flowmap.city/), a side project for [visualizing mobility flows](https://ilya.boyandin.me/p/flowmap-city) that he still maintains.
He had little budget for infrastructure, and DuckDB already had a [WebAssembly build]({% link docs/current/clients/wasm/overview.md %}) that runs in the browser.

<video src="https://ilya.boyandin.me/media/projects/flowmap-city/flowmap-city.mp4"
       controls autoplay muted loop playsinline style="width: 100%; height: auto;"></video>
In Flowmap.city, users upload files to object storage, which is cheap or often free. When someone opens a project, DuckDB-Wasm starts in the browser and loads the files from the bucket over the [`s3://`]({% link docs/current/core_extensions/httpfs/s3api.md %}) protocol:

```sql
SELECT *
FROM 's3://⟨bucket⟩/⟨project⟩/flows.parquet';
```

**“They can do all the analytics and visualization in the browser,”** Ilya says, **“which is very convenient and has very low requirements on the infrastructure.”**

Flowmap.city showed him that an analytical application could work with no database server behind it.

## From a Side Project to SQLRooms

For Flowmap.city, Ilya had built a table view, a SQL editor, a schema tree and query execution in the browser. None of it was specific to mobility data. He extracted these pieces into a standalone framework called [SQLRooms](https://ilya.boyandin.me/p/sqlrooms), which he continues to develop as open source.

SQLRooms is a set of building blocks for creating DuckDB-backed analytics applications in React. Developers combine only the packages and state slices their app needs. At its center is the Room, a self-contained workspace that holds the data along with the analysis done on it. Each Room is connected to its own DuckDB instance, running natively on a device or as WebAssembly in a browser. The framework's DuckDB wrapper handles file loading, so data can be queried as soon as it is added. All computation happens on the user's own machine, whether that is a browser tab or a desktop app. The framework also has built-in support for AI agents that write and run SQL.

The quickest way in is the [minimal example](https://sqlrooms.org/examples), a small Vite and React app that loads a CSV file and runs SQL queries against it:

```sql
SELECT *
FROM 'sales.csv'
LIMIT 10;
```


## Graph Analytics on the Fly with Cosmograph

[Cosmograph](https://ilya.boyandin.me/p/cosmograph), a tool for graph analytics, Ilya has been building alongside with [Nikita Rokotyan](https://rokotyan.com/) and Thor Whalen, is another application on top of the framework. It allows you to visualize graphs with millions of edges directly in the browser and use SQL Rooms features, like an AI assistant, to analyze your data.

<video src="https://ilya.boyandin.me/media/projects/cosmograph/cosmograph.mp4"
       controls autoplay muted loop playsinline style="width: 100%; height: auto;"></video>
Other developers have built on the framework as well. Amar Harolikar customized SQLRooms to analyze an 11-million-row, 1.6 GB CSV file in natural language, entirely in the browser.

Ilya's interest in local-first software shaped the framework. **“You should own your data. It should live on your computer, not in the Cloud. It can be synced via the cloud, but you decide when to keep it local and when not.”**

Cosmograph has been tested with datasets ranging from tens of thousands of points to several million. Previously, it was practically impossible to have an analytical workflow in the browser that could manage such large amounts of data due to the browser's memory and processing limits. With DuckDB WASM, we were able to parse huge data files without blocking the main thread or consuming browser page memory. Moreover, DuckDB's fast engine allows us to perform all sorts of summarization and filtering queries on the fly with unprecedented performance. Furthermore, the data is stored in a columnar format and can be passed to the GPU for rendering without additional processing.

Ilya says this lets the app **“work with even more data than we can render at once.”**

## One Database File, One Analytical Project

In SQLRooms applications, DuckDB can store the project along with the data. In Cosmograph, **“the project file you create is actually a DuckDB database file, in which we store both the data users add to the project and all the artifacts they create within it.”** Opening a project is a single `ATTACH`:

```sql
ATTACH 'project.duckdb';
```

A whole project then lives in one file that a user can move around like any other document. Ilya's current work on SQLRooms builds on this idea.

## When the Agent Writes the SQL

Work on Cosmograph started around the time AI agents became usable. **“This combination of SQL and AI agents is actually quite powerful,”** Ilya says. **“SQL has such a long history, so many documents on the web explaining how to use it, and LLMs are pretty good at generating it.”** Users ask a question and the agent writes and runs the DuckDB queries, so they don't need to know SQL.

The SQLRooms-powered AI Assistant allows you to analyze your data while minimally sharing it with AI providers. An agent writes SQL queries and chart schemas, then the framework runs and renders everything.

## Putting SQLRooms to Work at Foursquare

Another application built on SQLRooms is [Foursquare Spatial Desktop](https://ilya.boyandin.me/p/fsq-spatial-desktop), a free desktop application for visualizing geospatial data. ~~Anyone can register and use it with their own data. Only the embedded AI agent is paid. Foursquare also uses SQLRooms for internal analysis tools.~~

Spatial Desktop followed [kepler.gl](kepler.gl), the free client-only tool, and Foursquare Studio, the commercial version hosted in the cloud. It was the first of these to use DuckDB, which Ilya proposed. **“For this particular product, I think DuckDB was just the perfect fit,”** he says. **“I wouldn't say we considered alternatives.”** SQLite is embeddable too, he says, but it is not column-oriented and was not designed for analytics on larger datasets.

DuckDB ships with the application, so users install nothing extra. Ilya acknowledges that a desktop application makes it harder to share projects and edit them together. In exchange, queries don't wait on the network and the data stays on the user's machine. Foursquare also has no database servers to run. Modern laptops are powerful, he adds, and their GPUs are good at visualization. **“There is a bit of a resurrection of desktop applications right now.”**

## Where SQLRooms Goes Next

The project file in Cosmograph points to a broader idea that Ilya is developing in SQLRooms, where the DuckDB file carries the analysis as well as the data. He is building a Canvas module with SQL query cells and Vega-Lite chart cells, alongside a Notebooks module. An engine underneath tracks how cells depend on each other and reruns downstream cells when the data changes.

He is also adding real-time collaboration based on the [Loro CRDT](https://loro.dev/) library. Because DuckDB treats the data as read-only, only the interface state needs to be synchronized between collaborators. That state can itself be stored in DuckDB, either on a shared server or on a local machine for offline work.

At [DuckCon #7]({% link _events/2026-06-24-duckcon7.html %}) in Amsterdam, Ilya showed a SQLRooms command-line tool that works with a DuckDB database and presents the analysis in a notebook-like style. The direction is a single DuckDB file that holds an entire analytical workspace, which someone can open elsewhere and continue. The design is still changing.

## Watch the DuckCon Talk

Ilya's five-minute talk from DuckCon #7 gives a short tour of this direction.

<div class="video-container">
<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/fV1LXnPl4ng" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
</div>

SQLRooms is open source and available at [sqlrooms.org](https://sqlrooms.org), where the example apps are a good place to start. [Flowmap.city](https://www.flowmap.city/) and [Cosmograph](https://cosmograph.app) are free to try.

We would like to thank Ilya for the conversation. You can follow his writing on DuckDB and SQLRooms on [his blog](https://ilya.boyandin.me/). And if you are reading this, are using DuckDB, and would like to share your story, drop us a line at [quack@duckdb.org](mailto:quack@duckdb.org).
