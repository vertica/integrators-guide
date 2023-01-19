---
title: "Vertica program extensions"
linkTitle: "Vertica program extensions"
weight: 30
---

## Validation Utilities

Vertica provides utilities that help you determine if your hosts and network can handle the processing and network traffic required by Vertica. The following tables contains the utilities required to use the Vertica validation scripts. Each directory is a subdirectory under `/home/dbadmin/vertica`:

| Directory           | Description      |
|:--------------------|:-----------------|
| `/bin`              | Add the following files:<ul><li>vcpuperf</li><li>vioperf</li><li>vnetperf_helper</li></ul> |
| `/config/logrotate` | Add the test file. |

## Vertica Language Drivers Files

Add the following language drivers to `/opt/vertica/bin`:

- VerticaSDK.jar
- VerticaUDxFence.jar
- vertica-udx-Java.jar
- vertica-udx-C++
- vertica-udx-Java
- vertica-udx-Python
- vertica-udx-zygote

## Vertica Java JDBC

In `/opt/vertica`, create new `/java` and a `/java/lib` subdirectory.
Add the following .jar files in each directory:

| Directory | Description           |
|:----------|:---------------------|
| `/java`     | <ul><li>vertica-javadoc.jar</li><li>vertica-jdbc-10.0.0-0.jar</li><li>vertica-jdbc.jar</li></ul> |
| `/java/lib` | <ul><li>vertica-jdbc-10.0.0-0.jar</li><li>vertica-jdbc.jar</li></ul> |

## Sample Database

Vertica provides a sample database, VMart, that includes schemas, tables, and scripts. Use VMart as sandbox environment to learn Vertica without loading or changing your organization's data.

Create the **/opt/vertica/examples** directory, and add the VMart_Schema.