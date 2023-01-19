---
title: "Creating a Vertica image"
linkTitle: "Creating a Vertica image"
weight: 10
---

[Vertica on Kubernetes](https://www.vertica.com/docs/latest/HTML/Content/Authoring/Containers/Kubernetes/ContainerizedVerticaWithK8s.htm) deploys an [Eon Mode database](https://www.vertica.com/docs/latest/HTML/Content/Authoring/Eon/EonOverview.htm) in a Kubernetes StatefulSet. The [Vertica server image](https://hub.docker.com/r/vertica/vertica-k8s) is optimized for Kubernetes, using the minimum tools and libraries requried to containerize Vertica.

This tutorial describes the components of our minimal Vertica image so you can build a custom Vertica image for development or production purposes. The Dockerfile reduces image size with a multistage build, and orders instructions for the most efficient cache and build.

For additional guidance, refer to the [Dockerfile](https://github.com/vertica/vertica-kubernetes/blob/main/docker-vertica/Dockerfile) hosted in the [vertica-kubernetes GitHub repository](https://github.com/vertica/vertica-kubernetes).

## Prerequisites

- Hardware or virtual machine with a container engine
- Vertica 11.0.0 or higher RPM
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

The `ADD` instruction copies files from the host filesystem into the container filesystem.

The following `ADD` instruction copies your Vertica RPM to the container's `/tmp` folder. This is used to install Vertica in the container:

```
ADD ./packages/$VERTICA_RPM /tmp/
```

The following `ADD` instructions add bash scripts that clean up your image and reduce its size. They are available in the `vertica-kubernetes/docker-vertica/packages` folder:

- **cleanup.sh** strips debugging symbols from the libraries in   `/opt/vertica/packages` directory, decreasing the image size. If you   set `ARG MINIMAL=YES`, this script removes any packages that are not   installed automatically, such as Tensorflow.
- **package-checksum-patcher.py** patches the library installers to use   new checksums that **cleanup.sh** created when removing the debugging   symbols.

```
ADD ./packages/cleanup.sh /tmp/  
ADD ./packages/package-checksum-patcher.py /tmp/
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

## Prepare the Entrypoint Script

The `vertica-kubernetes/docker-vertica/docker-entrypoint.sh` scriptrequires the following files in the container to execute properly:

```
COPY dbadmin/.bash_profile /home/dbadmin/  
COPY dbadmin/.ssh /home/dbadmin/.ssh  
COPY ./docker-entrypoint.sh /opt/vertica/bin/
```
In the previous command:

- **.bash_profile**: Add the .bash_profile to /home/dbadmin
- **dbadmin/.ssh**: Add static SSH keys to /home/dbadmin. Static keys  are requires so that if the environment runs multiple versions of the  image, then all nodes can communicate through SSH.
- **docker-entrypoint.sh**: Add the entrypoint script to vertica/bin

## Configure Container Network and Access Privileges

### Begin the RUN Instruction

Add `RUN set -x` to log each command to the console as it is executed:

```
RUN set -x \
```

### Configuring Container Network and Access Privileges

Use **&&** to chain instructions to the `RUN` command. Be sure to indent the following lines 2 spaces further than the `RUN set -x` command:

```
  && chmod a+x /opt/vertica/bin/docker-entrypoint.sh \  
  && chown dbadmin:verticadba /home/dbadmin/.bash_profile \  
  && chmod 600 /home/dbadmin/.bash_profile \
```

The commands in the previous example:

- Make the docker-entrypoint.sh script executable
- Provide the dbadmin ownership of the files created by the image
- Restrict access to the dbadmin's .bash_profile

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
### Ensure proper ownership

Ensure that everything under /home/dbadmin has the correct ownership:

```
  && chown -R dbadmin:verticadba /home/dbadmin/ \
```
On Docker versions 19 and earlier, ownership of the opt/vertica is not preserved in the COPY statement in the second stage of the image build. The following command makes this directory and its files are read and write ownership:

```
  && chmod 777 -R /opt/vertica
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

Inherit the minimal setting used in the previous build stage:

```
ARG MINIMAL
```
### COPY Artifacts from the First Stage

The `COPY` command adds files into the image as a new layer. The `--from=builder `<source-location>` `<container-location> option copies build artifacts from the first builder stage of the Dockerfile without the tools or files required to build them:

```
COPY --from=builder /opt/vertica /opt/vertica  
COPY --from=builder /home/dbadmin /home/dbadmin  
COPY --from=builder /root/.ssh /root/.ssh  
COPY --from=builder /var/spool/cron /var/spool/cron  
COPY --from=builder /usr/local/bin/docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
```

## Setting Environment Variables

Values set with the `ENV` instruction set environment variables available in the running container:

```
ENV LANG en_US.UTF-8  
ENV TZ UTC  
ENV DEBIAN_FRONTEND noninteractive
```

In the previous example:

- **ENV LANG** sets the character set to US English.
- **ENV TZ** sets the time zone. The default time zone is UTC (Coordinated Universal Time, previously called Greenwich Mean Time   (GMT)).
- **ENV PATH** sets the \$PATH in the container to include the Vertica binaries and system binaries.
- **DEBIAN_FRONTEND** set to `noninteractive` ensures that there is zero interaction while installing or upgrading the system with `apt`.

## Install Daemon Scripts

Because the container does not run systemd by default, we provide functions to enable the `vertica_agent` script to function properly:

```
ADD ./packages/init.d.functions /etc/rc.d/init.d/functions
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

### Address File Permissions Not Preserved in COPY Statement

For Docker versions 19.03.0 and earlier, recursively apply file permissions that were not preserved from the first build stage:

```
  && chown -R $DBADMIN_UID:$DBADMIN_GID /home/dbadmin \
```

### Update the Packages

Update the package cache so that you can install packages:

```
  && apt-get -y update \
```
### Vertica and Admintools Required Packages

Vertica and [Admintools](https://www.vertica.com/docs/latest/HTML/Content/Authoring/AdministratorsGuide/AdminTools/AdministrationToolsReference.htm) require the following packages to function properly:

```
  && apt-get install -y --no-install-recommends \  
  ca-certificates \  
  cron \  
  dialog \  
  gdb \  
  iproute2 \  
  krb5-user \  
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
  && update-alternatives --install /usr/bin/python python /opt/vertica/oss/python3/bin/python3 1
```

## The Entrypoint Script

The entrypoint script is what executes to create a container from your
image:

```
ENTRYPOINT ["/opt/vertica/bin/docker-entrypoint.sh"]
```

## Exposing Ports

Expose port 5433 for Vertica, and 5444 for the Vertica agent:

```
EXPOSE 5433  
EXPOSE 5444
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