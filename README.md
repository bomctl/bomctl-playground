# Bomctl Playground

A Gitpod based playground to understand `bomctl`.

## Use a Playground Environment

[Launch the Playground](https://gitpod.io/?autostart=true#https://github.com/bomctl/bomctl-playground) in gitpod.

## Install Locally

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

### Format Agnostic

Generating SBOMs for complex systems usually involves [multiple SBOM files](#multiple-sbom-files), and sometimes the
format of SBOMs provided for sub-components is outside of a user's control. Internally, `bomctl` leverages
[protobom](https://github.com/protobom/protobom) to parse and store SBOMs in a format-agnostic way. `bomctl` can
import SBOMs in various formats and easily perform operations on them without having to first be converted,
potentially losing information in the process. In addition, this allows for easy conversion of provided SBOMs
and generation of SBOMs in all supported formats.

For more information on which formats `bomctl` supports, run:

```bash
bomctl export --help
```

The supported formats and encodings are available in the help info.

### Multiple SBOM Files

Rarely are systems represented by a single application, and therefore multiple components in a system need are probably going to be individual SBOMs. For example a helm chart deployment has several container images. Each container image is going to have its own SBOM.

`bomctl` is designed to work with "trees of SBOMs" where linkages exist between components and lower level SBOMs.

## Exercises

### Download and List SBOMs in the Cache Exercise

bomctl has two commands for adding SBOMs to the cache

- `bomctl fetch` will download SBOMs from different sources
- `bomctl import` will import local SBOM files or SBOM content piped from stdin

#### __Step 1: Fetch__

Add SBOMs to cache:

``` bash
bomctl fetch https://raw.githubusercontent.com/bomctl/bomctl-playground/main/examples/bomctl-container-image/bomctl_bomctl_v0.3.0.cdx.json
```

Notice two important `bomctl` objectives in the command above.

1. `bomctl` is designed to handle collections or sets of SBOM documents. While only one url is listed in the command a second SBOM document was fetched along with the first. This happens because `bomctl` will recursively fetch any SBOMs that are external references of components. This is a foundational concept for representing modern software systems.
1. `bomctl` supports multiple SBOM formats. The first SBOM fetched is a CycloneDX 1.5 SBOM Document, the second externally referenced SBOM is an SPDX 2.3 SBOM Document.

#### __Step 2: List__

List SBOMs in cache:

``` bash
bomctl list
```

### Import SBOMs from an SBOM generation tool or file

#### __Step 1: Import from standard input__

Using [Trivy](https://trivy.dev/latest/), scan a remote image and pipe the output into `bomctl`

```bash
trivy image --format spdx-json chainguard/cosign:latest | bomctl import -
```

#### __Step 2: Import from file__

``` bash
bomctl import examples/bomctl_0.3.0_darwin_arm64.tar.gz.spdx.json
```

and/or

``` bash
bomctl import examples/bomctl_0.1.3_darwin_amd64.tar.gz.cdx.json
```

#### __Step 3: List__

List SBOMs in cache:

``` bash
bomctl list
```

### Deliver SBOMs to the local file system and another tool

#### __Step 1: Understanding Export Flags__

``` bash
bomctl export --help
```

Notice there are two flags that control how the document is exported:

- __Encoding__: Controls what encoding the resulting document will be, either json or xml. Default: `json`  

> [!NOTE]
> The format choice may change the encoding options.

- __Format__: Controls the format and format version. Default: `cyclonedx`  

> [!NOTE]  
  > The generic `spdx` and `cyclonedx` format options will use the latest version of that format supported by `bomctl`.
  > There are more explicit format options for each format version if an older version is desired.

#### __Step 2: Export to a file__

Export the an SBOM to a local directory.

This step assumes the CycloneDX SBOM from [exercise](#step-1-fetch) was previously fetched.

``` bash
bomctl export urn:uuid:f360ad8b-dc41-4256-afed-337a04dff5db -f spdx -o test.spdx.json
```

Note that the document that was initally fetched was a CycloneDX SBOM and the above command is exporting an SPDX SBOM.

#### __Step 3: Export to another tool__

[Grype](https://github.com/anchore/grype) is a tool that can scan SBOMs for vulnerabilities and accepts them as input from stdin.`bomctl`can facilitate this.

```bash
bomctl export urn:uuid:f360ad8b-dc41-4256-afed-337a04dff5db | grype
```

### Deliver SBOMs to a remote OCI registry

#### __Step 1: Understanding Push Flags__

``` bash
bomctl push --help
```

Similar to `export`, there are options to specify the format and encoding of the pushed SBOM(s).
Review flags for encoding and format [here](#step-1-understanding-export-flags)

#### __Step 2: Push__

Setup:

Push the original SBOM to a "remote" oci registry. In this exercise, the env variable used `BOMCTL_PORT_URL` is already set in the environment, so the commands  
can be copied/pasted as is. Based on [this](https://oras.land/docs/quickstart/) example from Oras.  

For a more interactive view, switch to the ports tab of the open terminal and select the globe icon next to the `Zot Registry` row.
This will open the deployed Zot Registry we are using for the OCI registry endpoint. There is another preloaded tag in this registry,
simply for illustration and initialization purposes.

> [!NOTE]  
  >
  > - Using this example registry is only necessary to make these exercises standalone, a personal or test oci registry can also be used.  
  > - For authentication to those registries, a `.netrc` file needs to be created and a flag `--netrc` should be added to the `push` command.  

``` bash
bomctl push -f spdx urn:uuid:f360ad8b-dc41-4256-afed-337a04dff5db oci://${BOMCTL_PORT_URL}/hello-bomctl:latest
```

#### __Step 3: Verify (Optional)__

Fetch the manifest from the local test registry

```bash
oras manifest fetch ${BOMCTL_PORT_URL}/hello-bomctl:latest | jq
```

Identify the appropriate blob digest from the manifest output from previous cmd:

For SPDX, look for a layer with `"mediaType": "application/spdx+json"`  
For CycloneDX, look for a layer with `"mediaType": "application/vnd.cyclonedx+json;version=<version#>"`. Note the `<version#>` at the end would have the CycloneDX version of the SBOM.

Once the layer has been identified, copy the digest.  
In the example below, the digest for the CycloneDX SBOM layer would be `"sha256:bd309293a0e79962aa08abf2e5e4e636d254accf9a455bdf17c6ca22bc3ae7a0"`

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "artifactType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.empty.v1+json",
    "digest": "sha256:44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a",
    "size": 2,
    "data": "e30="
  },
  "layers": [
    {
      "mediaType": "application/vnd.cyclonedx+json;version=1.5",
      "digest": "sha256:bd309293a0e79962aa08abf2e5e4e636d254accf9a455bdf17c6ca22bc3ae7a0",
      "size": 2470,
      "data": "super cool data"
    }
  ]
}
```

Fetch the blob contents using the digest located in the previous step:

```bash
oras blob fetch --output - ${BOMCTL_PORT_URL}/hello-bomctl@sha256:<digest of SBOM layer in manifest>
```

### Merge SBOMs

The `merge` command will merge two or more documents that exist in local storage. This will leave the original SBOMs unaltered in  
the cache and will generate a new SBOM with the combined, deduplicated contents.

More info regarding the `merge` command and merging strategy can be found in the help menu for the command:

```bash
bomctl merge --help
```

Next merge some simple test boms in the db to illustrate how it works.

``` bash
bomctl import examples/bomctl_merge_A.cdx.json examples/bomctl_merge_B.cdx.json
```

View the imported SBOMs in the database:

``` bash
bomctl list
```

These are intentionally small, manageable SBOMs. Use `cat` to view contents if desired, or simply open them in the editor to view.  
To give the merged SBOM a specific name, add a `--name` or `-n` flag; otherwise a UUID will be generated and used as the name.  

To merge:

```bash
bomctl merge urn:uuid:0cd5c64f-318a-40cd-a2a9-a93301beff5d urn:uuid:3de02d44-f9c6-4a94-bf48-eb92730dc3b5
```

> [!Important]  
> Look for the last line in the output, for a line that start with `INFO  merge: Adding merged document sbomID=`.  
> Copy the string with `urn:uid` (Hint: it's everything after the `=`), __Needed in the next step__.  

Do another `list` to see the new third SBOM that has been generated, while both previous SBOMs are still available.

``` bash
bomctl list
```

Now export the merged SBOM to see the before and after:  
Replace the `<UUID>` with the uuid copied in the previous step.

``` bash
bomctl export <UUID> -f spdx -o merged-sbom.spdx.json
```

These SBOMs were originally CycloneDX, but can be exported as SPDX if desired. Or, to inspect the merge behavior more easily
export in the original CycloneDX and use the original files as reference to understand the result (remember the default export  
format is CycloneDX so the `-f` isn't needed).

``` bash
bomctl export <UUID> -o merged-sbom.cdx.json
```

### SBOM Alias

Working with multiple SBOMs, it can be tricky to keep documents straight, as different formats have different expectations for naming. `bomctl`  
typically relies on document IDs for storing documents; however, these can be taxing to use as an ID when performing operations. The `alias` command  
gives the option to pick a more human readable name for a document for ease of use.

To get more information about `alias` and its' subcommands, run:

```bash
bomctl alias --help
```

#### __Step 1: Set Alias__

This step assumes the CycloneDX SBOM from [exercise](#step-1-fetch) was previously fetched.

```bash
bomctl alias set urn:uuid:f360ad8b-dc41-4256-afed-337a04dff5db 0_3_0
```

#### __Step 2: View Alias__

Displays top level info about everything in the cache, including aliases

```bash
bomctl list
```

or this command displays only the alias information

```bash
bomctl alias list
```

#### __Step 3: Remove Alias__

This commmand removes the existing alias for the provided SBOM ID.

```bash
bomctl alias remove urn:uuid:f360ad8b-dc41-4256-afed-337a04dff5db
```

### SBOM Tags

Working with multiple SBOMs, it can be hard to keep documents organized. There may be groups of SBOMs within your system that are related and should  
be handled together, or at least linked in a way that makes it easy to know what goes with what. The `tag` command allows a user to specify a 'tag'  
to associate with an SBOM to make searching for and organizing your SBOMs easier.

To get more information about `tag` and its' subcommands, run:

```bash
bomctl tag --help
```

#### __Step 1: Add Tag__

This step assumes the CycloneDX SBOM from [exercise](#step-1-fetch) was previously fetched.
Feel free to use any previously fetched SBOM in the local database instead by updating the SBOM ID in the command.

Since this SBOM originated from a`bomctl`container image, lets add those tags. Additional SBOM IDs can be specified separated by spaces.

```bash
bomctl tag add urn:uuid:f360ad8b-dc41-4256-afed-337a04dff5db bomctl container SBOM CDX Test
```

#### __Step 2: List Tag__

Lets list them out to illustrate how to check this for any SBOM in your database to get more info about it.

```bash
bomctl tag list urn:uuid:f360ad8b-dc41-4256-afed-337a04dff5db
```

#### __Step 3: Remove Tag__

It's probably redundant to have this SBOM tagged with `bomctl` since thats all we've been handling, so lets remove it.

```bash
bomctl tag remove urn:uuid:f360ad8b-dc41-4256-afed-337a04dff5db bomctl
```

Then run a quick list command to make sure it reflects the expected tags.

```bash
bomctl tag list urn:uuid:f360ad8b-dc41-4256-afed-337a04dff5db
```

#### __Step 4: Clear Tags__

This particular SBOM still has too many superfluous tags, so just clear them all.

```bash
bomctl tag clear urn:uuid:f360ad8b-dc41-4256-afed-337a04dff5db
```

Then run a quick list command to make sure it reflects the expected tags.

```bash
bomctl tag list urn:uuid:f360ad8b-dc41-4256-afed-337a04dff5db
```
