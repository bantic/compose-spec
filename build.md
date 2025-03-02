# The Compose Specification - Build support
{:.no_toc}

*Note:* Build is an OPTIONAL part of the Compose Specification

* ToC
{:toc}

## Introduction

Compose specification is a platform-neutral way to define multi-container applications. A Compose implementation
focussing on development use-case to run application on local machine will obviously also support (re)building
application from sources. The Compose Build specification allows to define the build process within a Compose file
in a portable way.

## Definitions

Compose Specification is extended to support an OPTIONAL `build` subsection on services. This section define the
build requirements for service container image. Only a subset of Compose file services MAY define such a Build
subsection, others being created based on `Image` attribute. When a Build subsection is present for a service, it
is *valid* for a Compose file to miss an `Image` attribute for corresponding service, as Compose implementation
can build image from source.

Build can be either specified as a single string defining a context path, or as a detailed build definition.

In the former case, the whole path is used as a Docker context to execute a docker build, looking for a canonical
`Dockerfile` at context root. Context path can be absolute or relative, and if so relative path MUST be resolved
from Compose file parent folder. As an absolute path prevent the Compose file to be portable, Compose implementation
SHOULD warn user accordingly.

In the later case, build arguments can be specified, including an alternate `Dockerfile` location. This one can be
absolute or relative path. If Dockerfile path is relative, it MUST be resolved from context path.  As an absolute
path prevent the Compose file to be portable, Compose implementation SHOULD warn user if an absolute alternate
Dockerfile path is used.

## Consistency with Image

When service definition do include both `Image` attribute and a `Build` section, Compose implementation can't
guarantee a pulled image is strictly equivalent to building the same image from sources. Without any explicit
user directives, Compose implementation with Build support MUST first try to pull Image, then build from source
if image was not found on registry. Compose implementation MAY offer options to customize this behaviour by user
request.

## Publishing built images

Compose implementation with Build support SHOULD offer an option to push built images to a registry. Doing so, it
MUST NOT try to push service images without an `Image` attribute. Compose implementation SHOULD warn user about
missing `Image` attribute which prevent image being pushed.

Compose implementation MAY offer a mechanism to compute an `Image` attribute for service when not explicitly
declared in yaml file. In such a case, the resulting Compose configuration is considered to have a valid `Image`
attribute, whenever the actual raw yaml file doesn't explicitly declare one.

## Illustrative sample

The following sample illustrates Compose specification concepts with a concrete sample application. The sample is non-normative.

```yaml
services:
  frontend:
    image: awesome/webapp
    build: ./webapp

  backend:
    image: awesome/database
    build:
      context: backend
      dockerfile: ../backend.Dockerfile

  custom:
    build: ~/custom
```

When used to build service images from source, such a Compose file will create three docker images:

* `awesome/webapp` docker image is build using `webapp` sub-directory within Compose file parent folder as docker build context. Lack of a `Dockerfile` within this folder will throw an error.
* `awesome/database` docker image is build using `backend` sub-directory within Compose file parent folder. `backend.Dockerfile` file is used to define build steps, this file is searched relative to context path, which means for this sample `..` will resolve to Compose file parent folder, so `backend.Dockerfile` is a sibling file.
* a docker image is build using `custom` directory within user's HOME as docker context. Compose implementation warn user about non-portable path used to build image.

On push, both `awesome/webapp` and `awesome/database` docker images are pushed to (default) registry. `custom` service image is skipped as no `Image` attribute is set and user is warned about this missing attribute.

## Build definition

The `build` element define configuration options that are applied by Compose implementations to build Docker image from source.
`build` can be specified either as a string containing a path to the build context or a detailed structure:

```yml
services:
  webapp:
    build: ./dir
```

Using this string syntax, only the build context can be configured as a relative path to the Compose file's parent folder.
This path MUST be a directory and contain a `Dockerfile`.

Alternatively `build` can be an object with fields defined as follow

### context (REQUIRED)

`context` defines either a path to a directory containing a Dockerfile, or a url to a git repository.

When the value supplied is a relative path, it MUST be interpreted as relative to the location of the Compose file.
Compose implementations MUST warn user about absolute path used to define build context as those prevent Compose file
for being portable.

```yml
build:
  context: ./dir
```

### dockerfile

`dockerfile` allows to set an alternate Dockerfile. A relative path MUST be resolved from the build context.
Compose implementations MUST warn user about absolute path used to define Dockerfile as those prevent Compose file
for being portable.

```yml
build:
  context: .
  dockerfile: webapp.Dockerfile
```

### args

`args` define build arguments, i.e. Dockerfile `ARG` values.

Using following Dockerfile:

```Dockerfile
ARG GIT_COMMIT
RUN echo "Based on commit: $GIT_COMMIT"
```

`args` can be set in Compose file under the `build` key to define `GIT_COMMIT`. `args` can be set a mapping or a list:

```yml
build:
  context: .
  args:
    GIT_COMMIT: cdc3b19
```

```yml
build:
  context: .
  args:
    - GIT_COMMIT=cdc3b19
```

Value can be omitted when specifying a build argument, in which case its value at build time MUST be obtained by user interaction,
otherwise build arg won't be set when building the Docker image.

```yml
args:
  - GIT_COMMIT
```

### cache_from

`cache_from` defines a list of images that the Image builder SHOULD uses for cache resolution.

```yml
build:
  context: .
  cache_from:
    - alpine:latest
    - corp/web_app:3.14
```

### extra_hosts

`extra_hosts` adds hostname mappings at build-time. Use the same syntax as [extra_hosts](spec.md#extra_hosts).

```yml
extra_hosts:
  - "somehost:162.242.195.82"
  - "otherhost:50.31.209.229"
```

Compose implementations MUST create matching entry with the IP address and hostname in the container's network
configuration, which means for Linux `/etc/hosts` will get extra lines:

```
162.242.195.82  somehost
50.31.209.229   otherhost
```

### isolation

`isolation` specifies a build’s container isolation technology. Like [isolation](spec.md#isolation) supported values
are platform-specific.

### labels

`labels` add metadata to the resulting image. `labels` can be set either as an array or a map.

reverse-DNS notation SHOULD be used to prevent labels from conflicting with those used by other software.

```yml
build:
  context: .
  labels:
    com.example.description: "Accounting webapp"
    com.example.department: "Finance"
    com.example.label-with-empty-value: ""
```

```yml
build:
  context: .
  labels:
    - "com.example.description=Accounting webapp"
    - "com.example.department=Finance"
    - "com.example.label-with-empty-value"
```

### shm_size

`shm_size` set the size of the shared memory (`/dev/shm` partition on Linux) allocated for building Docker image. Specify
as an integer value representing the number of bytes or as a string expressing a [byte value](spec.md#specifying-byte-values).

```yml
build:
  context: .
  shm_size: '2gb'
```

```yaml
build:
  context: .
  shm_size: 10000000
```

### target

`target` defines the stage to build as defined inside a multi-stage `Dockerfile`.

```yml
build:
  context: .
  target: prod
```

## Implementations

* [docker-compose](https://docs.docker.com/compose)
* [buildX bake](https://docs.docker.com/buildx/working-with-buildx/)
