---
title: "Startup files and minimum requirements"
linkTitle: "Startup files and minimum requirements"
weight: 10
---

The Vertica server package RPM and the [vertica_install](https://www.vertica.com/docs/latest/HTML/Content/Authoring/InstallationGuide/InstallingVertica/RunTheInstallScript.htm) installation script include files that are not required to run a Vertica
instance (e.g. an example database). The following sections provide information about the minimum requirements to manually configure Vertica
on your machine.

## System User Requirements

Vertica has a number of user-level requirements, including username and group requirements. For detailed information, see [System User
Requirements](https://www.vertica.com/docs/latest/HTML/Content/Authoring/InstallationGuide/BeforeYouInstall/systemuserreqts.htm).

## Operating System Configurations

To install Vertica on your machine, you must configure your operating system with the required packages, settings, and tools. The following links provide information about distribution-specific prerequisites needed to run Vertica:

- [Disk
  Readahead](https://www.vertica.com/docs/latest/HTML/Content/Authoring/InstallationGuide/BeforeYouInstall/DiskReadahead.htm)
- [Transparent   Hugepages](https://www.vertica.com/docs/latest/HTML/Content/Authoring/InstallationGuide/BeforeYouInstall/transparenthugepages.htm)
- [Swappiness](https://www.vertica.com/docs/latest/HTML/Content/Authoring/InstallationGuide/BeforeYouInstall/CheckforSwappiness.htm)
- [Network Time Protocol   (NTP)](https://www.vertica.com/docs/latest/HTML/Content/Authoring/InstallationGuide/BeforeYouInstall/ntp.htm)
- [Defragmentation](https://www.vertica.com/docs/latest/HTML/Content/Authoring/InstallationGuide/BeforeYouInstall/defrag.htm)
- [Maximum Memory Map   Configuration](https://www.vertica.com/docs/latest/HTML/Content/Authoring/InstallationGuide/BeforeYouInstall/maxmapcount.htm)
- [Support Tools for   Troubleshooting](https://www.vertica.com/docs/latest/HTML/Content/Authoring/InstallationGuide/BeforeYouInstall/supporttools.htm?cshid=S0041)

Configuration requirements might vary depending on your environment. If your operating system is not properly configured during installation, the installation process stops and the installer displays an error message and an identifier that indicates why the installation failed. 
See [General Operating System Configuration - Manual Configuration](https://www.vertica.com/docs/latest/HTML/Content/Authoring/InstallationGuide/BeforeYouInstall/GeneralOSConfigure.htm) in the [Vertica documentation](https://www.vertica.com/docs/latest/HTML/Content/Home.htm) for additional information about configuring your operating system.

## Vertica Directory Structure

The `/vertica` directory is the top-level directory that stores subdirectories containing important utilities, binaries, and files that are required to run Vertica on your system. Create the `/vertica` directory in the `/opt` directory:

`mkdir /opt/vertica`

The following table contains the minimum files, libraries, and binaries that are required to run Vertica. Each directory is a subdirectory of the `/vertica` folder.

| Directory | Contents |
|:----------|:------------|
| `/sbin` | <ul><li>ssh_config</li><li>vconf</li><li>verify_libraries_test</li><li>verticad</li><li>verticad.service</li></ul> |
|`/bin` | <ul><li>vertica</li><li>vsql</li><li>bootstrap-catalog (symbolic link to vertica)</li><li>vertica-download-file (symbolic link to vertica)</li><li>check-auth-config (symbolic link to vertica)</li><li>extract-snapshot (symbolic link to vertica)</li><li>check-extra-sal-files (symbolic link to vertica)</li></ul> |
|`/lib` | <ul><li>libAutopassCrypto64.so</li><li>libcom_err.so.3.0</li><li>liblmx64.so</li><li>libvmalloc.so</li></ul> |
|`/config` | Contains the `/share`  directory. |
|`/config/share` | <ul><li>agent.cert</li><li>agent.key</li><li>agent.pem</li><li>license.key</li></ul> |
|`/log`  | Contains the all `/all-local-verify-<timestamp>`  and `/local-coerce-<timestamp>`  log directories. |
|`/log/all-local-verify-<timestamp>` | <ul><li>verify-`<host-address>`.xml</li></ul> |
|`/log/local-coerce-<timestamp>` | <ul><li>verify-latest.xml</li></ul> |
|`/spread` | Contains the `/lib`  and `/sbin` directories. |
|`/spread/lib` | <ul><li>libspread-core-so.3.0.0 shared object</li></ul> |
|`/spread/sbin` | <ul><li>spread</li></ul> |

## Creating the man Files (Optional)

Add the man files to access man pages that provide information for Vertica commands and functions.

1.  Navigate into the `/usr/share/man/man1` directory:
    ```bash
    $ cd /usr/share/man/man1
    ```
2.  In the `man1` folder, add the vsql.1 file.

## Setting the PATH

Add the `/vertica/bin` directory to your path so that you can access Vertica binaries from any directory.

Open the .bashrc file with a text editor, and then add the `export` statement to the bottom of the file:

```bash
export PATH=$PATH:/opt/vertica/bin
```