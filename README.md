# Helm Charts

## Add

```bash
$ helm repo add dreamcodefactory https://thedreamcodefactory.github.io/helm-charts
```

## Update

```bash
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "dreamcodefactory" chart repository
```

## Search

```bash
$ helm search repo dreamcodefactory
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
dreamcodefactory/vaultwarden    0.2.1           1.36.0-alpine   A Helm chart for vaultwarden
```

## Install

```bash
$ helm upgrade --install vaultwarden dreamcodefactory/vaultwarden --version 0.2.1
Release "vaultwarden" does not exist. Installing it now.
NAME: vaultwarden
LAST DEPLOYED: Wed May 27 18:47:47 2026
NAMESPACE: default
STATUS: deployed
REVISION: 1
...
```
