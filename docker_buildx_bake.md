The build context is the files/dirs that are included as resources to the docker build process.

I didn't realize it, but the context you pass to docker doesn't have to be local. For example, you can pass a URL as the build context and docker will clone down the repo and use it as the build context. 

```bash
docker build \
  --file=Dockerfile.webapp \
  --tag=docker.io/username/webapp:latest \
  https://github.com/username/webapp
```
is equivalent to

```hcl
target "webapp" {
  dockerfile = "Dockerfile.webapp"
  tags = ["docker.io/username/webapp:latest"]
  context = "https://github.com/username/webapp"
}
```

For each target, you can pass a single context (a URL or directory path) to the context attribute or you can pass multiple contexts (local directories, Git URLs, and even other Bake targets) using a map and the contexts attribute.

| Context type    | Example                                 |
| --------------- | --------------------------------------- |
| Container image |	docker-image://alpine@sha256:0123456789 |
| Git URL	      | https://github.com/user/proj.git        |
| HTTP URL        | https://example.com/files               |
| Local directory |	../path/to/src                          |
| Bake target	  | target:base                             |

Build arguments may be passed in to the target using the args attribute as a map.

You pass in the dockerfile with the dockerfile attribute.

Bake supports inheritance and the way you specify this is with the inherits attribute which takes a list.

You can target multiple platforms passing a list of platforms to the platforms attribute.
```hcl
target "default" {
  platforms = ["linux/amd64", "linux/arm64", "linux/arm/v7"]
}
```

You can use the matrix attributes to define a map of variables that forks one target into multiples. Uses the name attribute.
```hcl
target "app" {
  name = "app-${tgt}"
  matrix = {
    tgt = ["foo", "bar"]
  }
  target = tgt
}

# OR

target "app" {
  name = "app-${tgt}-${replace(version, ".", "-")}"
  matrix = {
    tgt = ["foo", "bar"]
    version = ["1.0", "2.0"]
  }
  target = tgt
  args = {
    VERSION = version
  }
}

# OR

target "app" {
  name = "app-${item.tgt}-${replace(item.version, ".", "-")}"
  matrix = {
    item = [
      {
        tgt = "foo"
        version = "1.0"
      },
      {
        tgt = "bar"
        version = "2.0"
      }
    ]
  }
  target = item.tgt
  args = {
    VERSION = item.version
  }
}

```

You can tag a target with one or more tags using the tags attribute. You do so with a list of strings.

```hcl
target "default" {
  tags = [
    "org/repo:latest",
    "myregistry.azurecr.io/team/image:v1"
  ]
}
```

Using the dockerfile-inline attribute with groups looks super powerful. Note that groups always take presidence over an identically named target.

```hcl
target "default" {
  dockerfile-inline = "FROM ubuntu"
}

group "default" {
  targets = ["alpine", "debian"]
}
target "alpine" {
  dockerfile-inline = "FROM alpine"
}
target "debian" {
  dockerfile-inline = "FROM debian"
}
```

You can specify secrets to be passed in.
```hcl
variable "HOME" {
  default = null
}

target "default" {
  secret = [
    "type=env,id=KUBECONFIG",
    "type=file,id=aws,src=${HOME}/.aws/credentials"
  ]
}
```
And you can then use them in your Dockerfiles as follows:

```Dockerfile
RUN --mount=type=secret,id=aws,target=/root/.aws/credentials \
    aws cloudfront create-invalidation ...
RUN --mount=type=secret,id=KUBECONFIG \
    KUBECONFIG=$(cat /run/secrets/KUBECONFIG) helm upgrade --install
```

You can also pass in a target build stage to the target attribute. I really don't undertand this...



Variables are cool. The HCL file format supports variable block definitions. You can use variables as build arguments in your Dockerfile, or interpolate them in attribute values in your Bake file.

variable "TAG" {
  default = "latest"
}

target "webapp-dev" {
  dockerfile = "Dockerfile.webapp"
  tags = ["docker.io/username/webapp:${TAG}"]
}
You can assign a default value for a variable in the Bake file, or assign a null value to it. If you assign a null value, Buildx uses the default value from the Dockerfile instead.

You can override variable defaults set in the Bake file using environment variables. The following example sets the TAG variable to dev, overriding the default latest value shown in the previous example.

content_copy
$ TAG=dev docker buildx bake webapp-dev

You can set a Bake variable to use the value of an environment variable as a default value:

content_copy
variable "HOME" {
  default = "$HOME"
}

To interpolate a variable into an attribute string value, you must use curly brackets

variable "HOME" {
  default = "$HOME"
}

target "default" {
  ssh = ["default=${HOME}/.ssh/id_rsa"]
}

You can use build in function from the [cty]("https://github.com/zclconf/go-cty/tree/main/cty/function/stdlib") (pronounced see-tie) library.

```hcl
# docker-bake.hcl
target "webapp-dev" {
  dockerfile = "Dockerfile.webapp"
  tags = ["docker.io/username/webapp:latest"]
  args = {
    buildno = "${add(123, 1)}"
  }
}
```

Advanced usage including interpolation, functions and ternary operators can be found [here]("https://docs.docker.com/build/bake/advanced/")

Ternary example

```hcl
# docker-bake.hcl
variable "TAG" {default="" }

group "default" {
  targets = [
    "webapp",
  ]
}

target "webapp" {
  context="."
  dockerfile="Dockerfile"
  tags = [
    "my-image:latest",
    notequal("",TAG) ? "my-image:${TAG}": "",
  ]
}
```