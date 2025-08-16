# Create an Image with Compose

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
