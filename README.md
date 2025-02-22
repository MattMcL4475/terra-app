# Terra Applications

This repo contains tooling for launching custom approved apps in [Terra]([https://app.terra.bio/).

**Note this project is under active development, and may not be fully functional yet.**

## Table of Contents

- [Architecture](#architecture)
- [App Schema](#app-schema)
- [Supported Apps](#supported-apps)
- [Launching an App Locally](#launching-an-app-locally)
- [Launching an App on Terra](#launching-an-app-on-terra)
- [Helm Chart](#helm-chart)
- [Publishing `terra-app-charts` Helm Chart](#publishing-terra-app-charts-helm-chart)

# Architecture

Apps are defined by a YAML [schema](#app-schema). An "app launcher" (such as Terra) receives a request to launch an app, parses the app YAML file, and invokes the [helm chart](#helm-chart) with the appropriate values to launch the app on a Kubernetes cluster. Once the app is launched, it may be accessed by the end user in a browser via a proxy URL defined by the app launcher.

Diagram of the app launch flow:

![Leo - Custom Apps](Leo%20-%20Custom%20Apps.png)

[Lucidchart source](https://lucid.app/lucidchart/invitations/accept/35f8ee43-e1d7-43e2-94d2-65076c08c84d)

# App Schema

*Note: the schema is under evolution!*

Here is a description of the app YAML schema:

```
# Short name of the app, for example 'jupyter'. This is used to name some Kubernetes resources,
# for example the namespace is generated as 'jupyter-ns'.
name: jupyter

# Maintainer of the app. Informational only.
author: workbench-interactive-analysis@broadinstitute.org

# Longer description of the app. Informational only.
description: |
  Jupyter Notebook, Python and R kernels, GATK packages
  
# Version of the app. Informational only.
version: 1.0.14

# List of services the app exposes.
# NOTE: currently only one service is supported by the helm chart. In the future we may explore multi-service apps.
services:
  jupyter:
    # The app docker image. Required.
    image: "us.gcr.io/broad-dsp-gcr-public/terra-jupyter-gatk:1.0.14"
    
    # Port the app listens on. Required.
    port: 8000
    
    # Optional. If the app is configured to listen on a base URL path, specify it here.
    # If absent, we assume the app listens on / and we will set up an nginx reverse proxy based on
    # the service name. However, not all apps are guaranteed to behave correctly under reverse proxies;
    # so if possible, configure your app with a base URL.
    baseUrl: "/jupyter/"
    
    # Optional. Specifies a startup command and arguments for the app. This overrides the Dockerfile
    # ENTRYPOINT if present.
    command:  ["/usr/local/bin/jupyter"]
    args: ["notebook"]
    
    # Optional. Specifies the mount point for an attached PD and the access mode (ReadWriteOnce or ReadOnlyMany
    # for read-write or read-only disks respectively). If absent no persistent volumes will be created.
    pdMountPath: "/data"
    pdAccessMode: "ReadWriteOnce"
    
    # Optional. Specifies environment variables to be present in the container.
    environment:
      WORKSPACE_NAME: "my-ws"
      WORKSPACE_NAMESPACE: "my-proj"
```

# Supported Apps

Example apps can be found in this repo under [apps](/apps). View the following for more details:

If an app is in this list, it should have a working smoke test run on PRs. See the testing section for details 

[cellxgene](apps/cellxgene)

[cirrocumulus](apps/cirrocumulus)

[jupyter](apps/jupyter)

[rstudio](apps/rstudio)

[ucsc_genome_browser](apps/ucsc_genome_browser)

# Launching an App Locally

This repo contains a `terra-app-local.sh` script which can launch apps on a locally running [minikube](https://minikube.sigs.k8s.io/docs/) cluster. This can be much more convenient for development than launching apps against Terra.

Before running the script:
1. Install `minikube` according to instructions [here](https://minikube.sigs.k8s.io/docs/start/).
2. Start `minikube` service.
   a. On Mac (Catalina) I start it with the command (when NOT connected to a VPN):
     ```
     minikube start --vm=true --mount --mount-string="~/data:/data"
     ```
     Replace the data directory with the directory you want to mount into your persistent volume (or omit if not using persistence).
    
Once `minikube` is running, you can run the script. Here is the usage:
```
$ ./terra-app-local.sh 
terra-app-local.sh deploys an app on a local minikube cluster.

 Find more information at: https://github.com/DataBiosphere/terra-app

Usage:
  terra-app-local.sh [command]

Available commands:
  install       installs an app
  uninstall     deletes an app
  status        displays stats about an app
  list          lists all apps
  show-kubectl  prints kubectl commands to interact with an app

Flags:
  -h, --help    display help

Use "terra-app-local.sh [command] --help" for more information about a command.
```

Example:
```
$ ./terra-app-local.sh install -f /apps/jupyter/app.yaml
```

# Testing
The script detailed in the local launching section, `terra-app-local.sh`, is also used to smoke-test supported apps on merges to this repo. If you are adding an app, and want it to be automatically tested, be sure to add it to the `app` `matrix` in the github action file. 

These smoke-tests DO NOT test against Terra/Leonardo. It may be desirable to have a dedicated test in leonardo pegged to a specific version of an app.yaml. This is not possible until we begin publishing and versioning these app yaml. See https://broadworkbench.atlassian.net/browse/IA-2495.  

# Launching an App on Terra
 
Leonardo has an endpoint that takes an app descriptor and various arguments. See the [swagger](https://leonardo.dsde-dev.broadinstitute.org/#/apps/createApp) documentation for the endpoint, and the associated `CreateAppRequest` schema (located near bottom of swagger specification). 

# Helm Chart

There are two helm charts inside this repo:

1. [terra-app-setup-chart](terra-app-setup-chart): this chart sets up common infrastructure used by Galaxy and Terra custom apps.
2. [terra-app-chart](terra-app-chart): this chart deploys Terra custom apps.

The charts are deployed to the following repos:
```
helm repo add terra-app https://terra-app-charts.storage.googleapis.com
helm repo add terra-app-setup https://terra-app-setup-chart.storage.googleapis.com
helm repo update
```
```
$ helm show chart terra-app/terra-app
apiVersion: v2
description: Chart for deploying Terra applications
name: terra-app
type: application
version: 0.3.0

$ helm show chart terra-app-setup/terra-app-setup
apiVersion: v2
description: Chart for set up entities for deploying Terra applications
name: terra-app-setup
type: application
version: 0.0.1
```
