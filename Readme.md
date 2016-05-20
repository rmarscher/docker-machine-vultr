<!--[metadata]>
+++
title = "Vultr"
description = "Vultr driver for docker machine"
keywords = ["machine, Vultr, driver, docker"]
[menu.main]
parent="smn_machine_drivers"
+++
<![end-metadata]-->

# Docker Machine driver plugin for Vultr

This plugin adds support for [Vultr](https://www.vultr.com/) cloud instances to the `docker-machine` command line tool.

[![CircleCI branch](https://img.shields.io/circleci/project/janeczku/docker-machine-vultr/master.svg)](https://circleci.com/gh/janeczku/docker-machine-vultr/)

## Installation

Requirement: [Docker Machine >= 0.5.1](https://github.com/docker/machine)

Download the `docker-machine-driver-vultr` binary from the release page.
Extract the archive and copy the binary to a folder located in your `PATH` and make sure it's executable (e.g. `chmod +x /usr/local/bin/docker-machine-driver-vultr`).

## Usage instructions

Grab your API key from the [Vultr control panel](https://my.vultr.com/settings/) and pass that to `docker-machine create` with the `--vultr-api-key` option.

**Example for creating a new machine running RancherOS:**

    docker-machine create --driver vultr --vultr-api-key=abc123 rancheros-machine

**Example for creating a new machine running Ubuntu 14.04:**

    docker-machine create --driver vultr --vultr-api-key=abc123 --vultr-os-id=160 ubuntu-machine

Command line flags:

 - `--vultr-api-key`: **required** Your Vultr API key.
 - `--vultr-ssh-user`: SSH username for the new VPS
 - `--vultr-os-id`: Operating system ID to use (OSID). See [available OS IDs](https://www.vultr.com/api/#os_os_list).
 - `--vultr-region-id`: Region the VPS will be created in (DCID). See [available Region IDs](https://www.vultr.com/api/#regions_region_list).
 - `--vultr-plan-id`: Plan to use for this VPS (VPSPLANID). See [available Plan IDs](https://www.vultr.com/api/#plans_plan_list).
 - `--vultr-pxe-script`: PXE boot script ID. Requires custom OS (vultr-os-id=159)
 - `--vultr-ipv6`: Enable IPv6 support for the VPS.
 - `--vultr-private-networking`: Enable private networking support for the VPS.
 - `--vultr-backups`: Enable automatic backups for the VPS.
 - `--vultr-userdata`: Path to file with cloud-init user-data for non-custom OS
 - `--vultr-cloud-config-template`: Path to file with custom cloud-config template for custom OS

When `--vultr-os-id` is not specified the current stable version of [RancherOS](http://rancher.com/rancher-os/) will be installed on the VPS.

### PXE deployment

You can boot a custom OS using a PXE boot script that you created in your Vultr account panel by supplying it's ID with the `--vultr-pxe-script` flag and setting `--vultr-os-id` to `159`.
The operating system must support Cloud-init and be configured to use the `ec2` datasource type.

Environment variables and default values:

| CLI option                      | Environment variable         | Default                     |
|---------------------------------|------------------------------|-----------------------------|
| **`--vultr-api-key`**           | `VULTR_API_KEY`              | -                           |
| `--vultr-ssh-user`              | `VULTR_SSH_USER`             | `root`                      |
| `--vultr-region-id`             | `VULTR_REGION`               | 1 (*New Jersey*)            |
| `--vultr-plan-id`               | `VULTR_PLAN`                 | 29 (*768 MB RAM,15 GB SSD*) |
| `--vultr-os-id`                 | `VULTR_OS`                   | -                           |
| `--vultr-pxe-script`            | `VULTR_PXE_SCRIPT`           | -                           |
| `--vultr-ipv6`                  | `VULTR_IPV6`                 | `false`                     |
| `--vultr-private-networking`    | `VULTR_PRIVATE_NETWORKING`   | `false`                     |
| `--vultr-backups`               | `VULTR_BACKUPS`              | `false`                     |
| `--vultr-userdata`              | `VULTR_USERDATA`             | -                           |
| `--vultr-cloud-config-template` | `VULTR_CLOUD_CONFIG_TEMPLATE`| -                           |

### Find available plans for all Vultr locations

Check out [vultr-status.appspot.com](http://vultr-status.appspot.com) for a live listing of the available plans per region. Get the applicable `--vultr-region-id` and `--vultr-plan-id` parameters with the click of a button.

[![vultr-status website](vultr-status-screenshot.png?raw=true)](http://vultr-status.appspot.com)

### Cloud-config Template

This only applies when a Custom OS is used for the instance or the default RancherOS is used.

A cloud-config file is automatically generated and placed into the instance's user-data. It sets the hostname to the name given to docker-machine and sets ssh_authorized_keys with the key that is automatically created for the instance.

Additionally, if a custom PXE script is not used, the generated cloud-config file includes writing a start script that makes sure the hard drive is automatically formatted for RancherOS and will setup private networking if the `--vultr-private-networking` option is given.

It may be desired to provide your own cloud-config user-data. You can pass a path to a template file with the `--vultr-cloud-config-template` option. The currently available variables are:

- `{{.HostName}}` - string with the docker-machine hostname
- `{{.SSHkey}}` - string with the public ssh key
- `{{.PrivateNet}}` - boolean whether `--vultr-private-networking` was enabled
- `{{.CustomScript}}` - boolean whether or not a custom PXE script was used or the default RancherOS PXE.

Here are the contents of the default template. The disk format check in the start.sh fixes a bug with mounting the primary hard drive - see [issue #9](https://github.com/janeczku/docker-machine-vultr/issues/9) and [commit 799693e](https://github.com/janeczku/docker-machine-vultr/commit/799693e).

```
#cloud-config
hostname: {{.HostName}}
ssh_authorized_keys:
- {{.SSHkey}}
write_files:
- path: /opt/rancher/bin/start.sh
permissions: "0755"
content: |
  #!/bin/sh
  mount | grep /dev/vda >/dev/null
  RETVAL=$?
  if [ $RETVAL -eq 0 ]; then
    exit 0
  fi
  sudo dd if=/dev/zero of=/dev/vda bs=1M count=1
  sudo reboot{{if not .CustomScript}}{{if .PrivateNet}}
rancher:
network:
interfaces:
  eth0:
    dhcp: true
  eth1:
    address: $private_ipv4/16
    mtu: 1450{{end}}{{end}}
```
