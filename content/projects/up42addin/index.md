---
title: Sole Developer Commercial ArcGIS Pro Add-in for UP42
date: 2022-11-01
authors:
- andreshernandez
links:
  - type: site
    url: https://docs.up42.com/help/arcgis-addin
tags:
  - Product Design
  - ArcGIS Pro SDK
  - .NET (C#)
  - MVVM
  - WPF
---

The satellite data ecosystem is fragmented, forcing users to jump between multiple platforms. My challenge was to bridge this gap by bringing the entire UP42 "single access point" directly into the Esri ecosystem, the world's most-used desktop GIS.

As the sole developer, I single-handedly designed, built, and shipped the UP42 ArcGIS Pro Add-in from concept to launch.

**V1.0 (Storage Access)**: I engineered the initial release, which allowed users to log into their UP42 account, browse their existing storage of purchased imagery, and download/add data directly to their map layers.

**V2.0 (Full Catalog & Ordering)**: I then led the V2.0 upgrade, which delivered the full marketplace experience. Users can now draw an Area of Interest (AOI), search the entire UP42 catalog, filter by date or cloud cover, and place new commercial orders—all without ever leaving ArcGIS Pro.

This required building a multithreaded application using the .NET SDK, implementing the MVVM design pattern, and managing a CI/CD pipeline for automated testing and versioned releases.

In recognition of this work, I was selected to present the product's architecture and technical challenges at the 2022 Esri European Developer Summit.

<!--more-->
