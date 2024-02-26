### pfb-network-connectivity Docker image

Azavea provides the code to build the Docker image that is used to run an
analysis. There is no Image directly available at the time, thus it will be
necessary to build it manually, or pull it from an unofficial source.

#### Pull the image from an unofficial repository

There is no official `azavea/pfb-network-connectivity` docker repository (yet
🤞), but it is possible to pull the image from an unofficial one, and rename it
to the expected name. Please note that this image was built for the
`linux/amd64` platform.

```bash
docker pull rgreinho/pfb-network-connectivity:0.18.0
docker tag rgreinho/pfb-network-connectivity:0.18.0 azavea/pfb-network-connectivity:0.18.0
```

#### Build the Azavea docker image

Build the docker image from the latest tag.

```bash
git clone git@github.com:azavea/pfb-network-connectivity.git
cd pfb-network-connectivity
git checkout tags/0.18.0 -b 0.18.0
cd src/
docker buildx build -t azavea/pfb-network-connectivity:0.18.0 -f analysis/Dockerfile .
```

## Install

Install the tool from GitHub directly:

```bash
pip install git+https://github.com/PeopleForBikes/brokenspoke-analyzer
```

This will add a new command named `bna`.

## Quickstart

To run an analysis, the tools needs 2 parameters:

- The name of the country or dataset where the city is located.
- The name of the city.

Then simply run the tool, and all the steps will be performed automatically:

```bash
$ bna run arizona flagstaff
[17:00:55] Boundary files ready.
Downloaded Protobuf data 'arizona-latest.osm.pbf' (204.86 MB) to:
'data/arizona-latest.osm.pbf'
[17:07:21] OSM Region file downloaded.
           OSM file for flagstaff ready.
           Analysis for flagstaff complete.
```
