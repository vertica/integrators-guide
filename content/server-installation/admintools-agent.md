---
title: "Admintools, agent, and daemon files"
linkTitle: "Admintools, agent, and daemon files"
weight: 2
---

## Adding the Python Library

Python is required to run AdminTools and the AdminTools agent.

1. Create the `/oss` directory and add the python library to that directory:
    ```bash
    $ mkdir /home/dbadmin/opt/vertica/oss
    ```

2. Add the python3 binary to the `/sbin` directory:
    ```bash
    $ mv python3 /opt/vertica/sbin
    ```

## AdminTools Requirements

To run Admintools on your machine, you must create a sub-directory structure that contains important files, libraries, and binaries that are required to run Admintools on your system.

The following table contains the files required to run Admintools on your machine. Each directory is a subdirectory under `/opt/vertica`:

| Directory           | Description       |
|:--------------------|:------------------|
| `/bin`              | Create a symbolic link to adminTools:<br>`$ ln -s admintools /home/dbadmin/opt/vertica/bin/adminTools` |
| `/config`           | <ul><li>admintools.conf</li><li>admintools.conf.bak.1597862525.319460</li><li>admintools.conf.bak.1597862189.882214</li></ul> |
| `/config/logrotate` | admintool.logrotate |
| `/config/log`       | <ul><li>adminTools.errors</li><li>adminTools.log</li></ul> |


## Admintools Agent Requirements

The Vertica agent is required to use Admintools. The following table contains the files and directories required to run the Vertica agent. Each directory is a subdirectory of `/opt/vertica`:


| Directory         | Description    |
|:------------------|:---------------|
| `/agent`            | agent.sh |
| `/config/logrotate` | agent.logrotate |
| `/config/share`     | agent.cert agent.key agent.pem  |
| `/log`              | <ul><li>agent_dbadmin.log</li><li>agentStdMsg.log</li><li>agent.log</li><li>agent_dbadmin.err</li><li>agent.pid</li></ul> |


## Admintools Daemon Requirements

Add the following files to the `/opt/vertica/sbin` folder:

- daemonize
- verticad
- verticad.service

## Creating the man Files (Optional)

To access man pages that provide information about Vertica Admintools commands and functions, add the `admintools.1` file to the
`/usr/share/man/man1` directory.