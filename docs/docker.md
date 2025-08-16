# Docker

## Create an Image without Compose

The following commands are equivalent and can be used to build a docker image

- `docker image build`
- `docker build`
- `docker buildx build`
- `docker builder build`

For example, assuming the directory structure:

```bash
src
└── analysis
    └── Dockerfile
```

Inside `src` run

```bash
docker buildx build -t azavea/pfb-network-connectivity:0.16.1 -f analysis/Dockerfile .
```

to create an image called `azavea/pfb-network-connectivity:0.16.1` using the `Dockerfile` in the `analysis` directory.

## Create an Image with Compose

The following command can be used to build a docker image using Compose

- `docker compose`

For example, assuming the directory structure:

```bash
src
└── analysis
    └── Dockerfile
└── compose.yml
```

and a `compose.yml` given by:

```yaml
name: project-name
version: "1.0"

services:
  service1:
    build:
      context: .
      dockerfile: analysis/Dockerfile
    image: image_name:latest
```

Inside `src` run

```bash
docker compose build service1
```

to create an image called `image_name:latest` for the `service1` service using the `Dockerfile` in the `analysis` directory.
