---
title: "Creating a Vertica image"
linkTitle: "Creating a Vertica image"
weight: 1
---

[Vertica on Kubernetes](https://www.vertica.com/docs/latest/HTML/Content/Authoring/Containers/Kubernetes/ContainerizedVerticaWithK8s.htm) deploys an [Eon Mode database](https://www.vertica.com/docs/latest/HTML/Content/Authoring/Eon/EonOverview.htm) in a Kubernetes StatefulSet. The [Vertica server image](https://hub.docker.com/r/vertica/vertica-k8s) is optimized for Kubernetes, using the minimum tools and libraries requried to containerize Vertica.

This tutorial describes the components of our minimal Vertica image so you can build a custom Vertica image for development or production purposes. The Dockerfile reduces image size with a multistage build, and orders instructions for the most efficient cache and build.

For additional guidance, refer to the [Dockerfile](https://github.com/vertica/vertica-kubernetes/blob/main/docker-vertica/Dockerfile) hosted in the [vertica-kubernetes GitHub repository](https://github.com/vertica/vertica-kubernetes).

## Prerequisites

- Hardware or virtual machine with a container engine
- Vertica 11.0.1 or higher RPM
- Local clone of the [vertica-kubernetes GitHub
  repository](https://github.com/vertica/vertica-kubernetes)

To build a container image with the GitHub repo, you must store the Vertica RPM in your local clone repo, in the `docker-vertica/packages` directory, with the name `vertica-x86_64.RHEL6.latest.rpm`.

## Security Considerations

Vertica recommends the following open-source security scanners to protect against malware, and identify critical vulnerabilities that prevent unauthorized users from accessing sensitive information in your container runtime:

- [Anchore Engine](https://engine.anchore.io/docs/quickstart/): A static   analysis and policy-based compliance tool that evaluates images with   user-defined security policies. Anchore identifies vulnerabilities and   returns links to associated [NIST](https://www.nist.gov/) reports.
- [Trivy](https://aquasecurity.github.io/trivy/latest/): A vulnerability   scanner that you can add to your continuous integration (CI)   toolchain.

For additional container file guidance, refer to [Best practices for writing Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/).

# Multistage Build

Vertica uses a multistage build to reduce the size of the final Docker image. Each `RUN` command adds a new layer to the Docker image. Each layer performs work using the the previous layer, resulting in leftover build artifacts that increase the image size. Using a multistage build takes the final result of the first stage and use it as a single layer in the second stage, removing any unnecessary artifacts created by intermediary layers during any previous stages

# Setting Global ARG Variables

Before the build stages begin, we must define ARG variables that are scoped globally across the two stages. To make an ARG variable available in a particular stage, you must define it after the FROM tag.

The Vertica image uses two different operating systems during the entire build process:

- [CentOS 7](https://hub.docker.com/_/centos) as the base image for the   **builder** stage
- [Ubuntu](https://hub.docker.com/_/ubuntu) as the base image for the
  final image 
Assign your operating system selection to the `BUILD_OS_VERSION` or `BASE_OS_VERSION` variables:

```
ARG BASE_OS_VERSION="focal-20220801"
ARG BUILDER_OS_VERSION="7.9.2009"
```
Both versions that you select must correspond to a docker tag for the OS images.

When the MINIMAL argument is set to YES, the Dockerfile builds a smaller image. The smaller image omits large packages like Tensorflow and the Java runtime, which is required only to run Java UDx's. This will result in over 300MB (uncompressed) of savings. By default, we build the full image.

```
ARG MINIMAL=""
```

We use [s6](https://github.com/just-containers/s6-overlay) as the init program. This argument allows you to choose the version of that program. This version refers to one of the GitHub releases on the s6 [GitHub repository](https://github.com/just-containers/s6-overlay).
```
ARG S6_OVERLAY_VERSION=3.1.2.1
```

# First Stage

The first stage is named **builder**. The **builder** stage generates the /opt/vertica directory structure by downloading the required packages and dependencies, then running the Vertica installer. The `FROM` command sets the base image for the **builder** and initiates the build of that stage. The `BUILDER_OS_VERSION` was previously selected as the image version to use for this stage. The following `FROM` command assigns the name **builder** to the first stage in the multistage build:

```
FROM centos:centos${BUILDER_OS_VERSION} as builder
```

## Setting ARG Variables

Container files use the `ARG` instruction to define build process variables. **VERTICA_RPM** stores the name of the Vertica RPM file. You must store the RPM in the `/packages` directory to build a container image. If you do not have a Vertica license, use the free trial [Community Edition](https://www.vertica.com/register/) RPM.

```
ARG VERTICA_RPM="vertica-x86_64.RHEL6.latest.rpm"
```

The MINIMAL argument is already globally defined--the following line makes the variable available in this stage:

```
ARG MINIMAL
```

The next two variables define the default UID and GID of the [dbadmin](https://www.vertica.com/docs/latest/HTML/Content/Authoring/AdministratorsGuide/DBUsersAndPrivileges/Roles/DBADMINRole.htm) user account in the container:

```
ARG DBADMIN_GID=5000  
ARG DBADMIN_UID=5000
```

## Adding Files

The `COPY` instruction copies files from the host filesystem into the container filesystem.

The following `COPY` instruction copies your Vertica RPM to the container's `/tmp` folder. This is used to install Vertica in the container:

```
COPY ./packages/$VERTICA_RPM /tmp/
```

The following `COPY` instructions add bash scripts that clean up your image and reduce its size. They are available in the `vertica-kubernetes/docker-vertica/packages` folder:

- **cleanup.sh** strips debugging symbols from the libraries in   `/opt/vertica/packages` directory, decreasing the image size. If you   set `ARG MINIMAL=YES`, this script removes any packages that are not   installed automatically, such as Tensorflow.
- **package-checksum-patcher.py** patches the library installers to use   new checksums that **cleanup.sh** created when removing the debugging   symbols.

```
COPY ./packages/cleanup.sh /tmp/  
COPY ./packages/package-checksum-patcher.py /tmp/
```

The following `COPY` instructions add config files for sshd and ssh. These config files are used to ensure all environment variables are passed and accepted from the ssh client to the ssh server. This is needed so that when we start vertica, environment variables that are set in the pod are picked up by the server.

```
COPY ./packages/10-vertica-sshd.conf /etc/ssh/sshd_config.d/10-vertica-sshd.conf
COPY ./packages/10-vertica-ssh.conf /etc/ssh/ssh_config.d/10-vertica-ssh.conf
```

## Installing Dependencies

This section incrementally builds a single `RUN` instruction. The `RUN` instruction executes Bash commands that persist in your container.

Each `RUN` instruction adds a layer to the final image. To limit the number of `RUN` instructions, use the Bash **&&** operator to chain multiple `RUN` commands into a single command. To chain commands that span multiple lines into a single command, enter the backslash ( **\\** ) character at the end of the line.

### Begin the RUN Instruction

Add `RUN set -x` to log each command to the console as it is executed:

```
RUN set -x \
```

Indent the following commands 2 spaces further than the `RUN set -x \` command.

### Update the Packages

To begin, update all packages:

```
  && yum -q -y update \
```
### Vertica and Admintools Required Packages

Vertica and [Admintools](https://www.vertica.com/docs/latest/HTML/Content/Authoring/AdministratorsGuide/AdminTools/AdministrationToolsReference.htm) require the following packages to function properly:

```
  && yum install -y \  
     cronie \  
     dialog \  
     iproute \  
     mcelog \  
     openssh-server \  
     openssh-clients \  
     openssl \  
     sudo \  
     which \  
     zlib-devel \
```

### Configure the dbadmin Role and Group

Create the required **verticadba** group and add the **dbadmin** user:

```
  && /usr/sbin/groupadd -r verticadba --gid ${DBADMIN_GID} \  
  && /usr/sbin/useradd -r -m -s /bin/bash -g verticadba --uid ${DBADMIN_UID} dbadmin \
```
### Install the Locally-Sourced RPM

Install the RPM from the `docker-vertica/packages` directory in the container `/tmp` directory:

```
  && yum localinstall -q -y /tmp/${VERTICA_RPM} \
```
### Run install_vertica Script

To prepare the Vertica environment, run the [install_vertica script](https://www.vertica.com/docs/11.0.x/HTML/Content/Authoring/InstallationGuide/InstallingVertica/InstallVerticaScript.htm):

```
  && /opt/vertica/sbin/install_vertica \  
  --accept-eula \  
  --debug \  
  --dba-user-password-disabled \  
  --failure-threshold NONE \  
  --license CE \  
  --hosts 127.0.0.1 \  
  --no-system-configuration \  
  --ignore-install-config \  
  -U \  
  --data-dir /home/dbadmin \
```
### Add License Files

If you used the [Community Edition license](https://www.vertica.com/landing-page/start-your-free-trial-today/), create a directory to install the Community Edition license key:

```
  && mkdir -p /home/dbadmin/licensing/ce \  
  && cp -r /opt/vertica/config/licensing/* /home/dbadmin/licensing/ce/ \
```
Configure [logrotate](https://www.vertica.com/docs/11.0.x/HTML/Content/Authoring/AdministratorsGuide/Monitoring/Vertica/RotatingLogFiles.htm) to simplify log file administration:

```
  && mkdir -p /home/dbadmin/logrotate \  
  && cp -r /opt/vertica/config/logrotate /home/dbadmin/logrotate/  \  
  && cp /opt/vertica/config/logrotate_base.conf /home/dbadmin/logrotate/ \
```

Provide the dbadmin user ownership of the Vertica files:

```
  && chown -R dbadmin:verticadba /opt/vertica \
```
### Clean Up Install Files

Run the `cleanup.sh` script to reduce the size of the final image:

```
  && rm -rf /opt/vertica/lib64  \  
  && yum clean all \  
  && sh /tmp/cleanup.sh
```

## Prepare the static ssh keys

We use static SSH key for the dbadmin id. This is required so that if the environment runs multiple versions of the image, then all nodes can communicate through SSH.
```
COPY dbadmin/.ssh /home/dbadmin/.ssh  
```

## Configure Container Network and Access Privileges

### Begin the RUN Instruction

Add `RUN set -x` to log each command to the console as it is executed:

```
RUN set -x \
```

### Copy SSH Keys

The following commands copy the static SSH key to use for root and ensures all keys have the proper permissions:

```
  && mkdir -p /root/.ssh \  
  && cp -r /home/dbadmin/.ssh /root \  
  && chmod 700 /root/.ssh \  
  && chmod 600 /root/.ssh/* \  
  && mkdir -p /home/dbadmin/.ssh \  
  && chmod 600 /home/dbadmin/.ssh/* \
```
### Ensure proper ownership and permissions

Ensure that everything under /home/dbadmin has the correct ownership and the ssh config files have the correct permissions:

```
  && chown -R dbadmin:verticadba /home/dbadmin/ \
  && chmod go-w /etc/ssh/sshd_config.d/* /etc/ssh/ssh_config.d/*
```

# Second Stage

The second and final build stage:

- copies only the necessary build artifacts from the first stage to   reduce the number of layers in the final image.
- creates environment variables.
- exposes ports for networking.
- adds metadata to the image.

The beginning of the second stage is indicated by a row of **\#** characters in the Dockerfile.

### Choosing Your Operating System

As previously mentioned, the second stage uses Ubuntu as the operating system. You can set the OS version with the `BASE_OS_VERSION` variable that you set earlier:

```
FROM ubuntu:${BASE_OS_VERSION}
```

### Additional ARG Variables

Reuse the ARG variables from the first stage that define Vertica license and user information:

```
ARG DBADMIN_GID=5000  
ARG DBADMIN_UID=5000
```

In the previous example:

- **DBADMIN_GID** is the default GID for the [dbadmin](https://www.vertica.com/docs/latest/HTML/Content/Authoring/AdministratorsGuide/DBUsersAndPrivileges/Roles/DBADMINRole.htm) account in the container.
- **DBADMIN_UID** is the default UID for the [dbadmin](https://www.vertica.com/docs/latest/HTML/Content/Authoring/AdministratorsGuide/DBUsersAndPrivileges/Roles/DBADMINRole.htm) account in the container.

Select the Java runtime version to install. This must correspond with a package name found in the Ubuntu distribution:

```
ARG JRE_PKG=openjdk-8-jre-headless
```

Inherit the arguments in the previous build stage:

```
ARG MINIMAL
ARG S6_OVERLAY_VERSION
```

### COPY Artifacts from the First Stage

The `COPY` command adds files into the image as a new layer. The `--from=builder `<source-location>` `<container-location> option copies build artifacts from the first builder stage of the Dockerfile without the tools or files required to build them:

```
COPY --from=builder /opt/vertica /opt/vertica  
COPY --from=builder --chown=$DBADMIN_UID:$DBADMIN_GID /home/dbadmin /home/dbadmin  
COPY --from=builder /root/.ssh /root/.ssh  
COPY --from=builder /var/spool/cron /var/spool/cron  
COPY --from=builder /etc/ssh/sshd_config.d/* /etc/ssh/sshd_config.d/
COPY --from=builder /etc/ssh/ssh_config.d/* /etc/ssh/ssh_config.d/
```

## Setting Environment Variables

Values set with the `ENV` instruction set environment variables available in the running container:

```
ENV PATH "$PATH:/opt/vertica/bin:/opt/vertica/sbin"
ENV DEBIAN_FRONTEND noninteractive
```

In the previous example:

- **PATH** sets the \$PATH in the container to include the Vertica binaries and system binaries.
- **DEBIAN_FRONTEND** set to `noninteractive` ensures that there is zero interaction while installing or upgrading the system with `apt`.

## Install Daemon Scripts

Because the container does not run systemd by default, we provide functions to enable the `vertica_agent` script to function properly:

```
ADD ./packages/init.d.functions /etc/rc.d/init.d/functions
```

## Install init program

The init program that we use in the container is called [s6](https://github.com/just-containers/s6-overlay). It is like systemd, but is designed for containers. It behaves like a true init program (PID 1): reaping zombie processes, passing signals down to child process, restart long-running services. Both cron and sshd are setup as long-running services. If any of those two services stop running, s6 will restart them.

The following commands will copy over scripts and binaries needed to run s6:

```
ADD https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-noarch.tar.xz /tmp
ADD https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-x86_64.tar.xz /tmp
```

The following command will copy our custom config for s6 so that sshd and cron are created as long-running services:
```
COPY s6-rc.d/ /etc/s6-overlay/s6-rc.d/
```

## Installing Dependencies

This section incrementally builds a single `RUN` instruction. The `RUN` instruction executes Bash commands that persist in your container.

Each `RUN` instruction adds a layer to the final image. To limit the number of `RUN` instructions, use the Bash **&&** operator to chain multiple `RUN` commands into a single command.

Many of the following commands are similar to those added in the previous build stage.

### Setup the SHELL

Setup the shell so that commands fail if a command flowing through a `|` fails.

```
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
```

### Begin the RUN Instruction

Add `RUN set -x` to log each command to the console as it is executed:

```
RUN set -x \
```
Be sure to indent all of the following commands 2 spaces further than the `RUN set -x \` command.

### Update the Packages

Update the package cache so that you can install packages:

```
  && apt-get -y update \
```
### Vertica and Admintools Required Packages

Vertica and [Admintools](https://www.vertica.com/docs/latest/HTML/Content/Authoring/AdministratorsGuide/AdminTools/AdministrationToolsReference.htm) require the following packages to function properly. There are also some additional packages included to make it easier when running `kubectl exec` for the container.

```
  && apt-get install -y --no-install-recommends \  
  ca-certificates \  
  cron \  
  dialog \  
  gdb \  
  iproute2 \  
  krb5-user \  
  less \
  libkeyutils1\  
  libz-dev \  
  locales \  
  logrotate \  
  ntp \  
  openssh-client \  
  openssh-server \  
  openssl \  
  procps \  
  sysstat \  
  sudo \   
  vim-tiny \
```
### Install Java

Add the following only if you are building a minimal image:

```
  && if ($MINIMAL != "YES" && $MINIMAL != "yes")  ; then \
    apt-get install -y --no-install-recommends $JRE_PKG; \
  fi \
```

### Cleanup package manager

Remove any cached data brought in from the package manager:

```
  && rm -rf /var/lib/apt/lists/* \  
```

### Setup the locale

Make the `en_US.UTF-8` locale so that Vertica will use utf-8 by default:

```
  && localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8 \
```
### SSH Setup

This ensures the ssh daemon can start:

```
  && mkdir -p /run/sshd \  
  && ssh-keygen -q -A \
```

### Configure the dbadmin Role and Group

Create the required **verticadba** group and add the **dbadmin** user:

```
  && /usr/sbin/groupadd -r verticadba --gid ${DBADMIN_GID} \  
  && /usr/sbin/useradd -r -m -s /bin/bash -g verticadba --uid ${DBADMIN_UID} dbadmin \
```
### Allow Passwordless sudo Access for dbadmin

```
  && echo "dbadmin ALL=(ALL) NOPASSWD: ALL" | tee -a /etc/sudoers \
```

### Setup limits for dbadmin

```
  && echo "dbadmin -       nofile  65536" >> /etc/security/limits.conf \
```

### Setup Java

We set the `JAVA_HOME` environment variable if we included Java in this image. This is used by Vertica to know where to find the Java runtime:

```
  && if $MINIMAL != "YES" && $MINIMAL != "yes" ; then \
    echo "JAVA_HOME=/usr" >> /etc/environment; \
  fi \
```
### Set Python Path

This step allows you to call Python from anywhere in the system. This is only required to allow us to run the UDx samples, as some samples use a Python script to generate data for ingestion:

```
  && update-alternatives --install /usr/bin/python python /opt/vertica/oss/python3/bin/python3 1 \
```

### Make cron a setuid program

This step changes cron so that it's setuid. This is done so that s6 doesn't t have to run `sudo cron ...` to start it.

```
  && chmod u+s /usr/sbin/cron \
```

### Unpack s6

We copied s6 tar files in an earlier step. This will extract them into the root of the file system.

```
  && tar -C / -Jxpf /tmp/s6-overlay-x86_64.tar.xz \
  && tar -C / -Jxpf /tmp/s6-overlay-noarch.tar.xz
```

## The Entrypoint Script

The entrypoint script is what executes to create a container from your image. We call the s6 init program and let it supervise the start of other processes.

```
ENTRYPOINT ["/init"]
```

## Exposing Ports

Expose port 5433 for Vertica, and 8443 for Vertica's HTTP server:

```
EXPOSE 5433  
EXPOSE 8443
```

## Configuring Image Access

Set the default user that runs the image to dbadmin:

```
USER dbadmin
```

## Adding Labels to the Image

Labels enable you to add metadata to your image, which is helpful when storing images in repositories and tracking build information. Vertica uses the following `LABELS`:

```
LABEL os-family="ubuntu"  
LABEL image-name="vertica_k8s"  
LABEL maintainer="K8s Team"  
LABEL org.opencontainers.image.source=[https://github.com/vertica/vertica-kubernetes/tree/main/docker-vertica](https://github.com/vertica/vertica-kubernetes/tree/main/docker-vertica) \  
      org.opencontainers.image.title='Vertica Server' \  
      org.opencontainers.image.description='Runs the Vertica server that is optimized for use with the VerticaDB operator' \  
      org.opencontainers.image.url=[https://github.com/vertica/vertica-kubernetes/](https://github.com/vertica/vertica-kubernetes/) \  
      org.opencontainers.image.documentation=[https://www.vertica.com/docs/latest/HTML/Content/Authoring/Containers/ContainerizedVertica.htm](https://www.vertica.com/docs/latest/HTML/Content/Authoring/Containers/ContainerizedVertica.htm)
```
