---
author: Brian Bowman
datetime: 2022-11-22
title: Opener
slug: opener
featured: true
draft: false
tags:
  - rust
ogImage: ""
description: >
  opener: a Rust crate for opening files and links
---

`opener` is a small Rust crate (library) for opening a file or link with the configured default application. I developed this library while working on a contribution to [rustup](https://github.com/rust-lang/rustup) and [Cargo](https://github.com/rust-lang/cargo). rustup supports opening a local copy of Rust’s documentation in the user’s browser, and Cargo supports generating documentation for a crate and also opening it in the user’s browser. This browser-opening functionality was duplicated in these two tools, with slightly different behavior, so I saw an opportunity to extract the functionality into a crate that each tool could depend on. And so, `opener` was born.

`opener` supports the major desktop operating systems: Windows, Mac, and Linux. Each has its own implementation; Windows and Mac have fairly straightforward system-provided means of implementing the functionality, but Linux is much trickier due to the wide variety of Linux distributions and desktop environments. To deal with this, the crate embeds the `xdg-open` script from [xdg-utils](<[https://www.freedesktop.org/wiki/Software/xdg-utils/](https://www.freedesktop.org/wiki/Software/xdg-utils/)>), and delegates the task to that script.
