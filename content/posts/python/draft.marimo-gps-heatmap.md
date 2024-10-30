---
title: Creating a heatmap from GPS coordinates in Nextcloud + HomeAssistant # in any language you want
cover:
  image: "images/posts/marimo-gps-heatmap/cover.jpg"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
description: >
    A marimo data exploration and visualization notebook displaying a heatmap of a person's whereabouts!
showToc: true
tocOpen: true
author: LM. Garret
UseHugoToc:
---

This is a pet project I've wanted to do for a long time. Since 2020 to be precise, when I started keeping track of my phone's location using the FOSS Android app [PhoneTrack](https://gitlab.com/eneiluj/phonetrack-android), storing the GPS points in Nextcloud. I've since realized that I did not need an extra app since I was already using HomeAssistant on my Android, which can register the phone's coordinates as a sensor. Coupled with the [InfluxDB integration](https://www.home-assistant.io/integrations/influxdb/), I could then keep track of all my GPS points in a suitable time series database.

Why record all this? To build a heatmap of course! Data visualization is fun, maps are awesome, so let's get the best of both!

The first part of this post may be a bit specific to my setup: I will explain the steps needed to prepare the data for the visualization. Then in the second part I will build a notebook using [marimo](https://marimo.io/), rendering our heatmap. Having used Jupyter notebooks extensively and developed libraries leveraging the most out of it, I wanted to explore this new notebook library for a while.

The project is available in its own repo at https://github.com/lmgarret/marimo-gps-heatmap. 

## Requirements
First, clone the notebook repository:
```command
git clone git@github.com:lmgarret/marimo-gps-heatmap.git
```

The notebook requires [`marimo`](https://docs.marimo.io/getting_started/index.html) to run, and [`uv`](https://docs.astral.sh/uv/getting-started/installation/) is recommended. I'd recommend installing the latter first and then installing `marimo` with:
```command
uv tool install marimo --upgrade
```

## Data preparation
Due to the historical reasons explained before, my GPS coordinates reside in several places:
 - in an InfluxDB, as recorded by the HomeAssistant Android companion app and pushed by the HomeAssistant InfluxDB integration
 - in a Nextcloud Maps' Postgres table, as recorded and pushed by PhoneTrack
 - and an older phone with a broken screen, in PhoneTrack's app storage. For some reasons, it wasn't pushed or persisted on Nextcloud
 - a few GPX tracks prior to 2020, recorded while traveling abroad/hiking etc...

We basically have it split into InfluxDB, Postgres and GPX files, and we will want to consolidate all this later.

### InfluxDB
I want the notebook to directly query the time serie, so I will need to get access to the DB. First, we need to find out our bucket id: (replace `homelab` with the name of your organization)
```command
influx bucket ls -o homelab
ID			Name		Retention	Shard group duration	Organization ID		Schema Type
63bf176a32e55582	_monitoring	168h0m0s	24h0m0s			61c3f28fcd4c8755	implicit
1c08c2a014f250f4	_tasks		72h0m0s		24h0m0s			61c3f28fcd4c8755	implicit
0c0fcfcafb5cf8a0	hass		infinite	168h0m0s		61c3f28fcd4c8755	implicit
```

Our bucket ID here is `0c0fcfcafb5cf8a0`, we will now create an API token with `read` access:

```command
influx auth create --org homelab --read-bucket 0c0fcfcafb5cf8a0
ID			Description	Token			User Name	User ID			Permissions
611a0fb903b33073			<influxdb_api_token>	admin		611a0fb903b33073	[read:orgs/61c3f28fcd4c8755/buckets/0c0fcfcafb5cf8a0]
```

We can now store the token into a conf file like the `.conf.template.json` one:

```json {title=".conf.json"}
{
    "influxdb": {
        "token": "<influxdb_api_token>",
        "url": "<influxdb_url",
        "org": "homelab",
        "database": "hass"
    }
}
```


### Nextcloud Maps
Here I could directly query the PostgreSQL database. But since I already know that I will be working with GPX files, and I don't want to create new credentials for my Nextcloud PostgreSQL instance, I will be downloading a GPX file directly from the UI.
