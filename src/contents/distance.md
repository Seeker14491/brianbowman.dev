---
author: Brian Bowman
datetime: 2022-10-29
title: Distance Projects
slug: distance-projects
featured: true
draft: false
tags:
  - videogames
ogImage: "distance_og.jpg"
description: A walkthrough of various Distance-related projects, and how they fit together.
---

![Distance screenshot](/distance_og.jpg)

The "Distance Projects" are a sprawling collection of interconnected projects related to the indie PC game [Distance](http://survivethedistance.com/). These projects primarily concern themselves with fetching, storing, processing, and presenting data from the game's leaderboards.

## Table of contents

## Intro

First, some context on Distance.

Distance is a "racing-platformer" game, where players compete by completing each level as quickly as possible. Each player's best time for each level is recorded in that level's public leaderboard. As of this writing, there are 100+ levels included in the game, ~4000 community made levels, and ~1.7 million total leaderboard entries.

Distance uses [Steamworks](https://partner.steamgames.com/) as the backend for its leaderboards, as well as using the Steam Workshop to host player-made levels.

## steamworks-rs

[code](https://github.com/Seeker14491/steamworks-rs)

The Steamworks API officially supports C++, and there are bindings available for various other languages as well as game engines. Distance, which is made in [Unity](https://unity.com/), uses the [Steamworks.NET](http://steamworks.github.io/) library. However, I wanted to access the Steamworks API using Rust, and altough I found [existing bindings](https://github.com/Noxime/steamworks-rs), they were incomplete and lacked the APIs I needed. Additionally, those bindings were callback-based, like the official C++ API as well as Steamworks.NET. I knew I could leverage Rust's async programming support to write my own callback-free bindings that would be nicer to use. And thus I wrote my own bindings to the Steamworks APIs I needed, namely querying Steam Workshop, fetching leaderboard entries, and resolving Steam IDs to player names.

Although the Steamworks API is written in C++, it also provides a C-compatible interface, which allowed me to use Rust's [bindgen](https://github.com/rust-lang/rust-bindgen) in a straightforward manner to automatically generate the low-level Rust FFI bindings to Steamworks. I then implemented async, idiomatic, high-level bindings on top.

## DistanceSteamDataServer

[code](https://github.com/Seeker14491/DistanceSteamDataServer)

The Steamworks API works by communicating with the running Steam client application through IPC. The Steam client will then forward requests to Steam's backend servers through gRPC. This has implications on performance and reliability when fetching large amounts of data, like millions of leaderboard entries. When fetching large amounts of data, the Steam client would often crash. While I may have been able to overcome this issue by throttling requests, doing so felt hacky and would impact the speed at which I could fetch the data. So I took a different approach.

[SteamKit](https://github.com/SteamRE/SteamKit) is a C# library that allows communicating directly with the Steam network without having to go through the Steam client application, in contrast with normal use of the Steamworks API. Using this library, I implemented a server in C# that provides Steamworks-querying capability through a simple gRPC interface. I also wrote a small Rust client library for connecting to the server (the code is hosted in the same repo).

With this new server, I could now perform most of the work without having to go through the Steamworks API and Steam client. This increased the data fetching throughput, and fixed the reliability issues I was experiencing with the previous approach. Unfortunately, SteamKit doesn't yet provide Steam workshop querying, so I still use the Steamworks API for that.

## distance-db-populator

[code](https://github.com/Seeker14491/distance-db-populator)

`distance-db-populator` is the application that uses my `steamworks-rs` library and `DistanceSteamDataServer` application to query all of Distance's Steam data, and inserts it into a Postgres database. I make good use of async programming techniques to efficiently perform the large amount of queries needed.

## Distance Leaderboard Filter

"Distance Leaderboard Filter", which I'll abbreviate as DLF, is a subproject whose purpose is allowing certain leaderboard entries to be tagged. For example, an entry might be tagged if was determined that the player had cheated in some way. This tagging is done manually, by inspecting player replays in-game.

The choice was made to keep this tag data in a second Postgres database instead of the main database populated by `distance-db-populator` for reasons of separation of concerns, and data integrity: All data in in the main database can be easily refetched (as is done continuously anyways to keep the data fresh), wherease the DLF data is 2000+ manually reviewed leaderboard entries.

## Hasura Layer

The main Distance DB as well as the Leaderboard Filter DB are both hooked up to a [Hasura](https://hasura.io/) instance, which exposes a GraphQL API for querying data from both databases. This allows fetching the database data from browser-based webapps, without having to write custom server code.

[API explorer](https://cloud.hasura.io/public/graphiql?endpoint=https%3A%2F%2Fdistance-db-hasura.seekr.pw%2Fv1%2Fgraphql)
