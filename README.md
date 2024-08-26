# Bomctl Playground

A Gitpod based playground to understand `bomctl`.

### Use a Playground Environment

[Launch the Playground](https://gitpod.io/?autostart=true#https://github.com/bomctl/bomctl-playground) in gitpod. 

### Install Locally

Follow one of the [installation methods](https://github.com/bomctl/bomctl?tab=readme-ov-file#installation).

## Concepts

`bomctl` uses several key concepts that are important to understand to effectively use the tool.

### Caches

`bomctl` uses a sqlite document storage cache to manage documents. Similar to how docker image storage is managed on a system, users should not directly interact with the cache.

The cache is located in one of these locations:

- Unix:    `$HOME/.cache/bomctl`
- Darwin:  `$HOME/Library/Caches/bomctl`
- Windows: `%LocalAppData%\bomctl`

Software Bill of Materials (SBOMs) are imported into the cache, manipulated, and then exported.

### Multiple SBOM Files

Rarely are systems represented by a single application, and therefore multiple components in a system need are probably going to be individual SBOMs. For example a helm chart deployment has several container images. Each container image is going to have its own SBOM.

`bomctl` is designed to work with "trees of SBOMs" where linkages exist between components and lower level SBOMs.

## Exercises

### Download and List SBOMs in the Cache Exercise

bomctl has two commands for adding SBOMs to the cache

- `bomctl fetch` will download SBOMs from different sources
- `bomctl import` will import SBOM files or from stdin

#### __Step 1__

Add SBOMs to cache:

``` bash
bomctl fetch https://raw.githubusercontent.com/bomctl/bomctl-playground/main/examples/bomctl-container-image/bomctl_bomctl_v0.3.0.cdx.json
```

You will notice two important `bomctl` objectives in the command above. 

1. `bomctl` is designed to handle collections or sets of SBOM documents. While only one url is listed in the coomand a second SBOM document. This happens because `bomctl` will recursively fetch any SBOMs that are external references of components. We believe this is a foundational concept for representing systems. 
1. `bomctl` supports multiple SBOM formats. The first SBOM fetched is a cyclonedx 1.5 SBOM Document, the second externally referenced SBOM is an SPDX 2.3 SBOM Document.

#### __Step 2__

List SBOMs in cache:

``` bash
bomctl list
```
