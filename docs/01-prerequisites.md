# Prerequisites

## Hetzner Cloud

This tutorial leverages the [Hetzner Cloud](https://cloud.hetzner.de) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up.

TBD [Estimated cost](https://cloud.google.com/products/calculator#id=873932bc-0840-4176-b0fa-a8cfd4ca61ae) to run this tutorial: $0.23 per hour ($5.50 per day).

## Hetzner Cloud API

Hetzner is providing a REST-API and a CLI-Tool.

### Install hcloud CLI

Follow the [README](https://github.com/hetznercloud/cli/blob/main/README.md) to install and configure the `hcloud` command line utility.

Arch Linux users are able to install `community/hcloud`, Debian/Ubuntu users the `hcloud-cli` package.

This tutorial was tested with version 1.29.0.

### Configure `hcloud` for the first time

You'll need to create an API-Token via Cloud Console. Inside your Project you need to go to Security -> API TOKENS.

```
hcloud context create kubernetes-the-hard-way
```

Test if login and access to the API is working:

```
hcloud server list
```

## Running Commands in Parallel with tmux

[tmux](https://github.com/tmux/tmux/wiki) can be used to run commands on multiple compute instances at the same time. Labs in this tutorial may require running the same commands across multiple compute instances, in those cases consider using tmux and splitting a window into multiple panes with synchronize-panes enabled to speed up the provisioning process.

> The use of tmux is optional and not required to complete this tutorial.

![tmux screenshot](images/tmux-screenshot.png)

> Enable synchronize-panes by pressing `ctrl+b` followed by `shift+:`. Next type `set synchronize-panes on` at the prompt. To disable synchronization: `set synchronize-panes off`.

Next: [Installing the Client Tools](02-client-tools.md)
