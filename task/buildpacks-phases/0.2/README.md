# Cloud Native Buildpacks

This build template builds source into a container image using [Cloud Native
Buildpacks](https://buildpacks.io). To do that, it uses [builders](https://buildpacks.io/docs/concepts/components/builder/#what-is-a-builder) to run buildpacks against your application.

Cloud Native Buildpacks are pluggable, modular tools that transform application source code into OCI images. They replace Dockerfiles in the app development lifecycle, and enable for swift rebasing of images and modular control over images (through the use of builders), among other benefits. This command uses a builder to construct the image, and pushes it to the registry provided.

The lifecycle phases are run in separate containers to enable better security for untrusted builders. Specifically, registry credentials are hidden from the detect and build phases of the lifecycle, and the analyze, restore, and export phases (which require credentials) are run in the lifecycle image published by the [Cloud Native Buildpacks project]( https://hub.docker.com/u/buildpacksio).

See also [`buildpacks`](../buildpacks) for the combined version of this task, which uses the [creator binary](https://github.com/buildpacks/spec/blob/platform/0.3/platform.md#operations), to run all of the [lifecycle phases](https://buildpacks.io/docs/concepts/components/lifecycle/#phases). This task, in contrast, runs all of the phases separately.

## Compatibility

- **Tekton** v0.11.0 and above
- **[Platform API][platform-api]**  0.3
    - For other versions, see [previous versions](#previous-versions).

## Install the Task

```
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/master/task/buildpacks-phases/0.2/buildpacks-phases.yaml
```

## Parameters

* **`BUILDER_IMAGE`**: The image on which builds will run. (must include lifecycle and compatible buildpacks; _required_)

* **`CACHE`**: The name of the persistent app cache volume. (_default:_ an empty directory -- effectively no cache)

* **`PLATFORM_DIR`**: A directory containing platform provided configuration, such as environment variables.
  Files of the format `/platform/env/MY_VAR` with content `my-value` will be translated by the lifecycle into environment variables provided to buildpacks. For more information, see the [buildpacks spec](https://github.com/buildpacks/spec/blob/master/buildpack.md#provided-by-the-platform). (_default:_ an empty directory)

* **`USER_ID`**: The user ID of the builder image user, as a string value. (_default:_ `"1000"`)

* **`GROUP_ID`**: The group ID of the builder image user, as a string value. (_default:_ `"1000"`)

* **`PROCESS_TYPE`**: The default process type to set on the image. (_default:_ `"web"`)

* **`SOURCE_SUBPATH`**: A subpath within the `source` input where the source to build is located. (_default:_ `""`)

### Outputs

* **`image`**: An `image`-type `PipelineResource` specifying the image that should
  be built.

## Workspaces

The `source` workspace holds the source to build. See `SOURCE_SUBPATH` above if source is located within a subpath of this input.

## Usage

This `TaskRun` will use the `buildpacks` task to build the source code, then publish a container image.

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: example-run
spec:
  taskRef:
    name: buildpacks-phases
  podTemplate:
    volumes:
# Uncomment the lines below to use an existing cache
#    - name: my-cache
#      persistentVolumeClaim:
#        claimName: my-cache-pvc
# Uncomment the lines below to provide a platform directory
#    - name: my-platform-dir
#      persistentVolumeClaim:
#        claimName: my-platform-dir-pvc
  params:
  - name: SOURCE_SUBPATH
    value: <optional subpath within your source repo, e.g. "apps/java-maven">
  - name: BUILDER_IMAGE
    value: <your builder image tag, see below for suggestions, e.g. "builder-repo/builder-image:builder-tag">
# Uncomment the lines below to use an existing cache
#  - name: CACHE
#    value: my-cache
# Uncomment the lines below to provide a platform directory
#  - name: PLATFORM_DIR
#    value: my-platform-dir
  resources:
    outputs:
    - name: image
      resourceSpec:
        type: image
        params:
        - name: url
          value: <your output image tag, e.g. "gcr.io/app-repo/app-image:app-tag">
  workspaces:
  - name: source
    persistentVolumeClaim:
      claimName: my-source-pvc
```

### Example builders
Paketo:
- `paketobuildpacks/builder:base` &rarr; Ubuntu bionic base image with buildpacks for Java, NodeJS and Golang
- `paketobuildpacks/builder:tiny` &rarr; Tiny base image (bionic build image, distroless run image) with buildpacks for Golang
- `paketobuildpacks/builder:full-cf` &rarr; cflinuxfs3 base image with buildpacks for Java, .NET, NodeJS, Golang, PHP, HTTPD and NGINX
> NOTE: The `paketobuildpacks/builder:full-cf` requires setting the USER_ID and GROUP_ID parameters to 2000, in order to work.

Heroku:
 - `heroku/buildpacks:18` &rarr; heroku-18 base image with buildpacks for Ruby, Java, Node.js, Python, Golang, & PHP

Google:
 - `gcr.io/buildpacks/builder:v1` &rarr; Ubuntu 18 base image with buildpacks for .NET, Go, Java, Node.js, and Python

## Previous Versions

For support of previous [Platform API][platform-api]s use a previous version of this task.

> Be sure to also supply a compatible builder image (`BUILDER_IMAGE` input) when running the task (i.e. one that has a lifecycle that supports the platform API).

| Version        | Platform APIs
|----            |-----
| [0.1](../0.1/) | [0.3][platform-api-0.3]

[platform-api]: https://buildpacks.io/docs/reference/spec/platform-api/
[platform-api-0.3]: https://github.com/buildpacks/spec/blob/platform/0.3/platform.md