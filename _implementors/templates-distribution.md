---
layout: implementors
title:  "Dev Container Templates distribution and discovery [proposal]"
shortTitle: "Templates distribution"
author: Microsoft
index: 10
---

This specification defines a pattern where community members and organizations can author and self-publish [Dev Container Templates](/implementors/templates). 

Goals include:

- For Template authors, create a "self-service" way to publish a Template, either publicly or privately, that is not centrally controlled.
- Provide the ability to standardize publishing such that supporting tools may implement their own mechanism to aid Template discoverability as they see fit.

## <a href="#source-code" name="source-code" class="anchor"> Source code </a>

A Template's source code is stored in a git repository.

For ease of authorship and maintenance, [1..n] Templates can share a single git repository. This set of Templates is referred to as a "collection," and will share the same [`devcontainer-collection.json`](#devcontainer-collection.json) file and "namespace" (eg. `<owner>/<repo>`).

> **Note:** Templates and [Features](/implementors/features) should be placed in different git repositories. 

Source code for a set of Templates follows the example file structure below:

```
.
├── README.md
├── src
│   ├── dotnet
│   │   ├── devcontainer-template.json
│   │   ├── .devcontainer.json
│   │   ├── ...
|   ├
│   ├── docker-from-docker
│   │   ├── devcontainer-template.json
│   │   ├── .devcontainer
│   │       ├── devcontainer.json
│   │       ├── Dockerfile
│   │       └── ...
│   │   ├── ...
|   ├
│   ├── go-postgres
│   │   ├── devcontainer-template.json
│   │   ├── .devcontainer
│   │       ├── devcontainer.json
│   │       ├── docker-compose.yml
│   │       ├── Dockerfile
│   │       └── ...
│   │   ├── ...
```

...where `src` is a directory containing a sub-folder with the name of the Template (e.g. `src/dotnet` or `src/docker-from-docker`) with at least a file named `devcontainer-template.json` that contains the Template metadata, and a `.devcontainer.json` (or `.devcontainer/devcontainer.json`) that the supporting tools will drop into an existing project or folder.

Each sub-directory should be named such that it matches the `id` field of the `devcontainer-template.json`.  Other files can also be included in the Templates's sub-directory, and will be included during the [packaging step](#packaging) alongside the two required files.  Any files that are not part of the Templates's sub-directory (e.g. outside of `src/dotnet`) will not included in the [packaging step](#packaging).

## <a href="#versioning" name="versioning" class="anchor">Versioning </a>


Each Template is individually [versioned according to the semver specification](https://semver.org/). The `version` property in the respective `devcontainer-template.json` file is parsed to determine if the Template should be republished.

Tooling that handles publishing Templates will not republish Templates if that exact version has already been published; however, tooling must republish major and minor versions in accordance with the semver specification.

## <a href="#packaging" name="packaging" class="anchor"> Packaging </a>


Templates are distributed as tarballs. The tarball contains the entire contents of the Template sub-directory, including the `devcontainer-template.json`, `.devcontainer.json` (or `.devcontainer/devcontainer.json`), and any other files in the directory.

The tarball is named `devcontainer-template-<id>.tgz`, where `<id>` is the Templates's `id` field.

A reference implementation for packaging and distributing Templates is provided as a GitHub Action (https://github.com/devcontainers/action).


<div id="devcontainer-collection.json"></div>
### devcontainer-collection.json

The `devcontainer-collection.json` is an auto-generated metadata file.

| Property | Type | Description |
| :--- | :--- | :--- |
| `sourceInformation` | object | Metadata from the implementing packaging tool. |
| `templates` | array | The list of Templates that are contained in this collection.|

Each Template's `devcontainer-template.json` metadata file is appended into the `templates` top-level array.

## <a href="#distribution" name="distribution" class="anchor"> Distribution </a>


There are several supported ways to distribute Templates.  Distribution is handled by the implementing packaging tool.

A user can add a Template in to their projects as defined by the [supporting tools](/supporting#supporting-tools-and-services).

### <a href="#oci-registry" name="oci-registry" class="anchor">OCI Registry</a>


An OCI registry that implements the [OCI Artifact Distribution Specification](https://github.com/opencontainers/distribution-spec) serves as the primary distribution mechanism for Templates.

Each packaged Template is pushed to the registry following the naming convention `<registry>/<namespace>/<id>[:version]`, where version is the major, minor, and patch version of the Template, according to the semver specification.

> **Note:** The `namespace` is a unique identifier for the collection of Templates and must be different than the collection of [Features](/implementors/features). There are no strict rules for the `namespace`; however, one pattern is to set `namespace` equal to source repository's `<owner>/<repo>`. 

A custom media type `application/vnd.devcontainers` and `application/vnd.devcontainers.layer.v1+tar` are used as demonstrated below.

For example, the `go` Template in the `devcontainers/templates` namespace at version `1.2.3` would be pushed to the ghcr.io OCI registry.

> **Note:** The example below uses [`oras`](https://oras.land/) for demonstration purposes.  A supporting tool should directly implement the required functionality from the aforementioned OCI artifact distribution specification.

```bash
# ghcr.io/devcontainers/templates/go:1
REGISTRY=ghcr.io
NAMESPACE=devcontainers/templates
TEMPLATE=go

ARTIFACT_PATH=devcontainer-template-go.tgz

for VERSION in 1  1.2  1.2.3  latest
do
        oras push ${REGISTRY}/${NAMESPACE}/${TEMPLATE}:${VERSION} \
                --manifest-config /dev/null:application/vnd.devcontainers \
                        ./${ARTIFACT_PATH}:application/vnd.devcontainers.layer.v1+tar
done

```

The "namespace" is the globally identifiable name for the collection of Templates. (eg: `owner/repo` for the source code's git repository).

The auto-generated `devcontainer-collection.json` is pushed to the registry with the same `namespace` as above and no accompanying `template` name. The collection file is always tagged as `latest`.

```bash
# ghcr.io/devcontainers/templates
REGISTRY=ghcr.io
NAMESPACE=devcontainers/templates

oras push ${REGISTRY}/${NAMESPACE}:latest \
        --manifest-config /dev/null:application/vnd.devcontainers \
                            ./devcontainer-collection.json:application/vnd.devcontainers.collection.layer.v1+json
```