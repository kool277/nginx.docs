---
title: Deploying NGINX and NGINX Plus with Docker
description: Run NGINX Open Source and F5 NGINX Plus in Docker containers from official images or your own image, and manage their files, logs, and signals.
toc: true
weight: 600
f5-content-type: how-to
f5-product: F5 NGINX Plus
f5-keywords: "nginx docker, nginx plus docker image, private-registry.nginx.com, docker run, docker build, bind mount, container logs, reload"
f5-summary: >
  This page shows how to run NGINX Open Source and F5 NGINX Plus in Docker containers.
  It covers the official NGINX Plus images from the F5 registry, the NGINX Open Source image from Docker Hub, and building your own NGINX Plus image.
  It also shows how to manage configuration files, logs, and NGINX signals in containers.
f5-audience: operator
---

## Overview

You can run [F5 NGINX Plus](https://www.f5.com/products/nginx/nginx-plus) and NGINX Open Source in Docker containers. For NGINX Plus, pull an official image from the F5 registry, or build your own image. For NGINX Open Source, run the official image from Docker Hub. When you finish this page, NGINX runs in a container, and you know how to give it configuration files, read its logs, and reload it.

---

<span id="prereq"></span>

## Before you begin

Before you begin, make sure you have:

- **Docker Engine**: You need the [Docker Engine](https://docs.docker.com/engine/install/) and its command-line tool.
- **For NGINX Plus**:
    - The JSON Web Token (JWT) license file and the `nginx-repo.crt` and `nginx-repo.key` files of your subscription, from the [MyF5](https://my.f5.com) customer portal.
    - A private Docker registry that runs and that you can push images to.
- **For NGINX Open Source**: Nothing else. You don't need a [Docker Hub](https://hub.docker.com/) account to pull the image, but Docker Hub allows fewer pulls without one.

---

<span id="nginx_plus_official_images"></span>

## Use the official NGINX Plus images

Since NGINX Plus [Release 31]({{< ref "nginx/releases.md#r31" >}}), you can get NGINX Plus images from the F5 registry at `private-registry.nginx.com` and upload them to your private registry.

The registry has these image types:

{{< table >}}

| Image type | Contents |
|---|---|
| `nginx-plus/base` | [NGINX Plus]({{< ref "/nginx/releases.md" >}}) |
| `nginx-plus/rootless-base` | NGINX Plus that runs as the `nginx` user, like an [unprivileged installation]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md#unpriv_install" >}}) |
| `nginx-plus/agent` | NGINX Plus with [NGINX Agent]({{< ref "/agent/" >}}) version 2 |
| `nginx-plus/agentv3` | NGINX Plus with NGINX Agent version 3 |
| `nginx-plus/rootless-agent` | NGINX Plus with NGINX Agent version 2, both running as the `nginx` user |
| `nginx-plus/rootless-agentv3` | NGINX Plus with NGINX Agent version 3, both running as the `nginx` user |
| `nginx-plus/modules` | [NGINX Plus dynamic modules]({{< ref "/nginx/admin-guide/dynamic-modules/dynamic-modules.md" >}}) |

{{< /table >}}

Tags select the operating system and the NGINX Plus release of an image.

### Tags for operating systems

{{< table >}}

| Operating system | Basic OS tag | Tag examples |
|---|---|---|
| Alpine (x86_64, aarch64) | `alpine` | `r36-alpine`, `r36-alpine-3.20` |
| Debian (x86_64, aarch64) | `debian` | `r36-debian`, `r36-debian-bookworm` |
| Red Hat Enterprise Linux (x86_64, aarch64) | `ubi` | `r36-ubi`, `r36-ubi-9`, `r36-ubi-9-20251201` |

{{< /table >}}

### Tags for NGINX Plus versions

The registry has images of the two most recent NGINX Plus versions. The basic operating system tag gives you the latest NGINX Plus version, built for the latest version of that operating system. Add the release to get a specific version, for example `r36-ubi-9` or `r36-alpine-3.22`.

### List all tags

To list the tags of an image type, run:

```shell
curl https://private-registry.nginx.com/v2/nginx-plus/<NGINXPLUS_IMAGE_TYPE>/tags/list --key <NGINX_REPO.KEY> --cert <NGINX_REPO.CRT> | jq
```

Where:

- `<NGINXPLUS_IMAGE_TYPE>` is an image type from the table above without `nginx-plus/`, for example `base` or `rootless-agent`.
- `<NGINX_REPO.KEY>` is the path to the client key from MyF5, for example `/etc/ssl/nginx/nginx-repo-x12345.key`.
- `<NGINX_REPO.CRT>` is the path to the client certificate from MyF5, for example `/etc/ssl/nginx/nginx-repo-x12345.crt`.
- `jq` formats the JSON output for easier reading. It needs the [jq](https://jqlang.github.io/jq/) JSON processor.

### Download your subscription credential files

{{< include "use-cases/credential-download-instructions.md" >}}

### Set up Docker for the F5 Container Registry

{{< include "use-cases/docker-registry-instructions.md" >}}

### Pull the image

Pull the image that you need from `private-registry.nginx.com`. Replace `<VERSION_TAG>` with a tag for the NGINX Plus version, or for the NGINX Plus version and the operating system, that you need.

- NGINX Plus:

    ```shell
    docker pull private-registry.nginx.com/nginx-plus/base:<VERSION_TAG>
    ```

- NGINX Plus with NGINX Agent version 2:

    ```shell
    docker pull private-registry.nginx.com/nginx-plus/agent:<VERSION_TAG>
    ```

- NGINX Plus with NGINX Agent version 3:

    ```shell
    docker pull private-registry.nginx.com/nginx-plus/agentv3:<VERSION_TAG>
    ```

- NGINX Plus running as the `nginx` user:

    ```shell
    docker pull private-registry.nginx.com/nginx-plus/rootless-base:<VERSION_TAG>
    ```

- NGINX Plus with NGINX Agent version 2, running as the `nginx` user:

    ```shell
    docker pull private-registry.nginx.com/nginx-plus/rootless-agent:<VERSION_TAG>
    ```

- NGINX Plus with NGINX Agent version 3, running as the `nginx` user:

    ```shell
    docker pull private-registry.nginx.com/nginx-plus/rootless-agentv3:<VERSION_TAG>
    ```

- NGINX Plus dynamic modules:

    ```shell
    docker pull private-registry.nginx.com/nginx-plus/modules:<VERSION_TAG>
    ```

To run the images that run as the `nginx` user on Kubernetes or OpenShift, see [Deploy unprivileged NGINX Plus on Kubernetes and OpenShift]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus-unprivileged-kubernetes-openshift.md" >}}).

{{< include "security/jwt-password-note.md" >}}

### Push the image to your private registry

After you pull the image, tag it and upload it to your private registry.

{{< call-out class="caution" >}} Never upload NGINX Plus images to a public repository such as Docker Hub. This violates your license agreement. {{< /call-out >}}

1. Log in to your private registry:

    ```shell
    docker login <YOUR_REGISTRY>
    ```

1. Tag and push the image. Replace `<YOUR_REGISTRY>` with the path of your registry, and `<VERSION_TAG>` with the tag of the image:

    ```shell
    docker tag private-registry.nginx.com/nginx-plus/base:<VERSION_TAG> <YOUR_REGISTRY>/nginx-plus/base:<VERSION_TAG>
    docker push <YOUR_REGISTRY>/nginx-plus/base:<VERSION_TAG>
    ```

### Run the NGINX Plus container

{{< call-out class="note" >}} Since [NGINX Plus Release 33]({{< ref "nginx/releases.md#r33" >}}), every NGINX Plus instance needs the JWT license file. For more information, see [About Subscription Licenses]({{< ref "/solutions/about-subscription-licenses.md" >}}). {{< /call-out >}}

Pass the contents of your JWT license file, named `license.jwt`, to the container in the `NGINX_LICENSE_JWT` environment variable. If the license file must be in another place than the default, `/etc/nginx/license.jwt`, set its full path in the `NGINX_LICENSE_PATH` variable.

To start a container with NGINX Plus only, run:

```shell
sudo docker run \
    --name nginxplus \
    --env=NGINX_LICENSE_JWT=$(cat license.jwt) \
    --restart=always \
    --runtime=runc \
    -p 80:80 \
    -d <YOUR_REGISTRY>/nginx-plus/base:<VERSION_TAG>
```

Replace `<YOUR_REGISTRY>` with the path of your private registry. The `-p 80:80` option maps port 80 of the Docker host to port 80 in the container.

To check that the container runs, list it. The `STATUS` column shows `Up`:

```shell
sudo docker ps --filter name=nginxplus
```

To start a container with NGINX Plus and NGINX Agent, also pass the NGINX One data plane key in the `NGINX_AGENT_SERVER_TOKEN` variable. For more information, see [Create and manage data plane keys]({{< ref "/nginx-one-console/connect-instances/create-manage-data-plane-keys.md" >}}).

```shell
sudo docker run \
    --name nginxplus \
    --env=NGINX_LICENSE_JWT=$(cat license.jwt) \
    --env=NGINX_AGENT_SERVER_GRPCPORT=443 \
    --env=NGINX_AGENT_SERVER_HOST=agent.connect.nginx.com \
    --env=NGINX_AGENT_SERVER_TOKEN="YOUR_NGINX_ONE_DATA_PLANE_KEY" \
    --env=NGINX_AGENT_TLS_ENABLE=true \
    --restart=always \
    --runtime=runc \
    -p 80:80 \
    -d <YOUR_REGISTRY>/nginx-plus/agent:<VERSION_TAG>
```

Where:

- `NGINX_LICENSE_JWT` is the contents of your JWT license file from MyF5, `license.jwt`.
- `NGINX_AGENT_SERVER_GRPCPORT` is the gRPC port of the server that NGINX Agent connects to.
- `NGINX_AGENT_SERVER_HOST` is the domain name or IP address of that server: `agent.connect.nginx.com` for NGINX One Console, or your NGINX Instance Manager. Don't expose NGINX Instance Manager to public networks in production.
- `NGINX_AGENT_SERVER_TOKEN` is your NGINX One data plane key. For details, see [Create and manage data plane keys]({{< ref "/nginx-one-console/connect-instances/create-manage-data-plane-keys.md" >}}).
- `NGINX_AGENT_TLS_ENABLE` turns on mutual TLS, server-side TLS, or insecure mode, which isn't recommended for production. For details, see [Encrypt communication]({{< ref "agent/configuration/encrypt-communication.md" >}}).
- `<YOUR_REGISTRY>` is the path of your private registry.
- `<VERSION_TAG>` is the tag that you used when you pushed the image to your registry.

For example, to start a container with NGINX Plus and NGINX Agent on Debian, directly from the F5 registry:

```shell
sudo docker run \
    --name nginxplus \
    --env=NGINX_LICENSE_JWT=$(cat license.jwt) \
    --env=NGINX_AGENT_SERVER_GRPCPORT=443 \
    --env=NGINX_AGENT_SERVER_HOST=agent.connect.nginx.com \
    --env=NGINX_AGENT_SERVER_TOKEN="YOUR_NGINX_ONE_DATA_PLANE_KEY" \
    --env=NGINX_AGENT_TLS_ENABLE=true \
    --restart=always \
    --runtime=runc \
    -p 80:80 \
    -d private-registry.nginx.com/nginx-plus/agent:debian
```

---

<span id="docker_oss"></span>

## Use the NGINX Open Source image

The official NGINX Open Source image on Docker Hub is built from the [nginx.org packages]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-open-source.md#prebuilt" >}}). It runs NGINX with the default configuration.

1. Start a container from the image:

    ```shell
    docker run --name mynginx1 -p 80:80 -d nginx
    ```

    Where:

    - `mynginx1` is the name of the new container.
    - `-p 80:80` maps port 80 of the Docker host to port 80 in the container. The first number is the port on the host.
    - `-d` runs the container in the background and prints its ID.

    The command prints the long form of the container ID, for example `fcd1fb01b14557c7c9d991238f2558ae2704d129cf9fb97bb4fadf673a58580d`. Docker uses this ID in the names of the container's log files.

1. Check that the container runs:

    ```shell
    docker ps
    ```

    The output lists the container. The `PORTS` column shows that port 80 of the host, for IPv4 and IPv6, maps to port 80 in the container:

    ```none
    CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                                 NAMES
    fcd1fb01b145   nginx     "/docker-entrypoint.…"   16 seconds ago   Up 15 seconds   0.0.0.0:80->80/tcp, [::]:80->80/tcp   mynginx1
    ```

---

<span id="docker_plus_image"></span>
<span id="docker_plus"></span>

## Build your own NGINX Plus image

NGINX Plus is a commercial product, so Docker Hub has no NGINX Plus images. Instead of the [official images](#nginx_plus_official_images), you can build your own.

{{< call-out class="caution" >}} Never upload NGINX Plus images to a public repository such as Docker Hub. This violates your license agreement. {{< /call-out >}}

1. Create a Dockerfile in an empty directory, the build context. For example, start from the [example Dockerfile](https://gist.github.com/nginx-gists/36e97fc87efb5cf0039978c8e41a34b5) for Debian. It installs NGINX Plus from the NGINX Plus repository with your subscription files.

    Like the NGINX Open Source image, this image sends the access and error logs to the Docker log collector, and it defines no volumes. To define volumes, add them to the Dockerfile, or to a Dockerfile that builds on this image:

    ```dockerfile
    VOLUME /usr/share/nginx/html
    VOLUME /etc/nginx
    ```

    The image doesn't copy files from the Docker host either. To add files, add `COPY` instructions.

1. Log in to the [MyF5 Customer Portal](https://account.f5.com/myf5), and download the `nginx-repo.crt`, `nginx-repo.key`, and JSON Web Token files, as described in [Before you begin](#prereq). A trial of NGINX Plus comes with these files.

1. Copy the files to the directory of the Dockerfile, and rename them to `nginx-repo.crt`, `nginx-repo.key`, and `license.jwt`. The build command reads these names.

1. Build the image, for example `nginxplus`. The final period sets the current directory as the build context.

    ```shell
    docker build --no-cache --secret id=nginx-key,src=nginx-repo.key --secret id=nginx-crt,src=nginx-repo.crt --secret id=nginx-jwt,src=license.jwt -t nginxplus .
    ```

    The `--secret` options give the build access to the files. The certificate and the key don't stay in the image. The JWT license does, because NGINX Plus needs it at runtime. The `--no-cache` option makes Docker build the image from scratch, so that it installs the latest version of NGINX Plus. Without it, Docker can reuse the NGINX Plus version of an image that it built before.

1. Check that Docker created the image:

    ```shell
    docker images nginxplus
    ```

    The output lists the image `nginxplus` with the tag `latest`.

1. Start a container from the image, for example `mynginxplus`:

    ```shell
    docker run --name mynginxplus -p 80:80 -d nginxplus
    ```

1. Check that the container runs:

    ```shell
    docker ps
    ```

You manage NGINX Plus containers the same way as NGINX Open Source containers.

---

<span id="manage"></span>

## Manage content and configuration files

You can manage the content and the configuration files of NGINX in three ways:

- [Keep the files on the Docker host](#manage_host).
- [Copy the files into the image](#manage_copy).
- [Change the files in a running container](#manage_container).

<span id="manage_host"></span>

### Keep the files on the Docker host

When you create a container, you can mount directories of the Docker host into it. The image's default configuration serves content from `/usr/share/nginx/html`, and its `/etc/nginx/nginx.conf` reads every `.conf` file in `/etc/nginx/conf.d`. For content in `/var/www` and server configuration files in `/var/nginx/conf.d` on the host, run:

```shell
docker run --name mynginx2 \
    --mount type=bind,source=/var/www,target=/usr/share/nginx/html,readonly \
    --mount type=bind,source=/var/nginx/conf.d,target=/etc/nginx/conf.d,readonly \
    -p 80:80 \
    -d nginx
```

Changes to the files in `/var/www` and `/var/nginx/conf.d` on the host appear in `/usr/share/nginx/html` and `/etc/nginx/conf.d` in the container. With `readonly`, the container can't change these files. After you change a configuration file, [reload NGINX](#control).

<span id="manage_copy"></span>

### Copy the files into the image

You can copy the files into a new image when you build it. In this example, the content is in the `content` directory, and the configuration files are in the `conf` directory, both next to the Dockerfile. `COPY` paths are relative to the build context. To use only your own configuration files, the `RUN` instruction deletes the image's default server, `default.conf`:

```dockerfile
FROM nginx
RUN rm /etc/nginx/conf.d/default.conf
COPY content /usr/share/nginx/html
COPY conf /etc/nginx
```

1. Build the image from the directory of the Dockerfile:

    ```shell
    docker build -t mynginx_image1 .
    ```

1. Start a container from the image:

    ```shell
    docker run --name mynginx3 -p 80:80 -d mynginx_image1
    ```

To change the files, build a new image and start a new container from it, or [change the files in the running container](#manage_container).

<span id="manage_container"></span>

### Change files in a running container

To change files inside a running container, open a shell in it with `docker exec`:

```shell
docker exec -it mynginx3 bash
```

The NGINX Open Source image is based on Debian and has `bash`. Images based on Alpine Linux have `sh` instead. To leave the shell, run `exit`. The container keeps running.

Changes inside a container are lost when you remove the container. To keep them for the next container, store the files in named volumes. Docker creates a volume the first time that a container mounts it. If the volume is empty, Docker copies the files of the image into it. For example, to keep the content and the configuration in the volumes `nginx-content` and `nginx-conf`:

```shell
docker run --name mynginx4 \
    --mount source=nginx-content,target=/usr/share/nginx/html \
    --mount source=nginx-conf,target=/etc/nginx \
    -p 80:80 \
    -d nginx
```

The changes that you make in `mynginx4` stay in the volumes. A new container that mounts the same volumes gets them. To keep the files in directories of the Docker host instead, see [Keep the files on the Docker host](#manage_host).

Other containers can then use these volumes. For example, to work on the files with the tools of another image, start a helper container with the volumes of `mynginx4`:

```shell
docker run -it --volumes-from mynginx4 --name mynginx4_files debian /bin/bash
```

---

<span id="log"></span>

## Manage logging

<span id="log_default"></span>

### Use the default logging

The NGINX image sends the [access log](https://nginx.org/en/docs/http/ngx_http_log_module.html#access_log) and the [error log](https://nginx.org/en/docs/ngx_core_module.html#error_log) to the Docker log collector, because it links them to `stdout` and `stderr`. With the default logging driver, Docker writes both logs to the file `/var/lib/docker/containers/CONTAINER_ID/CONTAINER_ID-json.log` on the host. `CONTAINER_ID` is the long form of the ID that `docker run` prints when you [create a container](#docker_oss). To show it, run:

```shell
docker inspect --format '{{ .Id }}' CONTAINER_NAME
```

To read the log messages with the Docker command line, run:

```shell
docker logs CONTAINER_NAME
```

To read them with the Docker Engine API, send a `GET` request to the Docker UNIX socket:

```shell
curl --unix-socket /var/run/docker.sock --output - "http://localhost/containers/CONTAINER_NAME/logs?stdout=1&stderr=1"
```

Quote the URL, because the shell treats `&` as a special character. Without `--output -`, curl doesn't print the logs to a terminal, because they contain binary data. For a container without a terminal, the API puts a binary header of 8 bytes before each log message. For only the access log, use only `stdout=1`. For only the error log, use only `stderr=1`. For other options, see [Get container logs](https://docs.docker.com/reference/api/engine/version/v1.45/#tag/Container/operation/ContainerLogs) in the [Docker Engine API](https://docs.docker.com/reference/api/engine/version/v1.45/) documentation.

<span id="log_custom"></span>

### Use custom logging

To log differently in some configuration blocks, such as `server {}` and `location {}`, write the logs to files in a volume. Then read them from a helper container with any logging tools. For example, to keep log files in `/var/log/nginx/log`, add a `VOLUME` instruction for this directory to the Dockerfile:

```dockerfile
FROM nginx
COPY content /usr/share/nginx/html
COPY conf /etc/nginx
VOLUME /var/log/nginx/log
```

Then [build an image](#manage_copy), and create an NGINX container and a helper container that both use this volume.

---

<span id="control"></span>

## Control NGINX

You can run NGINX commands in the container with `docker exec`. To test the configuration and reload it, run:

```shell
docker exec CONTAINER_NAME nginx -t
docker exec CONTAINER_NAME nginx -s reload
```

You can also send [signals](https://nginx.org/en/docs/control.html) to NGINX, which runs as the main process of the container. To reload the configuration, send the `HUP` signal:

```shell
docker kill -s HUP CONTAINER_NAME
```

To restart NGINX, restart the container:

```shell
docker restart CONTAINER_NAME
```

---

## Troubleshooting

### The container doesn't start because the port is in use

**Symptom**: `docker run` prints an error, and the container doesn't start. If another container uses the port, the error ends like this:

```none
Bind for 0.0.0.0:80 failed: port is already allocated
```

If another process on the Docker host uses the port, the error ends like this:

```none
failed to bind host port 0.0.0.0:80/tcp: address already in use
```

**Cause**: Another container or process on the Docker host already uses that host port.

**Fix**:

1. Remove the container that didn't start. It keeps its name, so another `docker run` with that name fails:

    ```shell
    docker rm CONTAINER_NAME
    ```

1. Stop the other container or process, or start the container again with another host port, for example `-p 8080:80`.

---

## References

- [Installing NGINX Open Source]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-open-source.md" >}})
- [Installing NGINX Plus]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus.md" >}})
- [Deploy unprivileged NGINX Plus on Kubernetes and OpenShift]({{< ref "/nginx/admin-guide/installing-nginx/installing-nginx-plus-unprivileged-kubernetes-openshift.md" >}})
- [About Subscription Licenses]({{< ref "/solutions/about-subscription-licenses.md" >}})
- [NGINX image on Docker Hub](https://hub.docker.com/_/nginx)
