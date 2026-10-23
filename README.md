# azcopy

Container images with [AzCopy](https://github.com/Azure/azure-storage-azcopy), Microsoft's command-line tool for copying data to and from Azure Storage. `azcopy` uploads, downloads and synchronizes files with Azure Blob Storage, Azure Files and Data Lake Storage, copies between storage accounts, and copies from Amazon S3 and Google Cloud Storage into Azure Blob Storage. It is compiled from the release tag with a current Go release on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`. The images are rebuilt when AzCopy publishes a release, when the pinned Go release is raised and when the base image changes. AzCopy's usage telemetry, which it sends to Microsoft by default, is turned off in these images (see [Telemetry](#telemetry)).

This is an unofficial build, not affiliated with or endorsed by Microsoft. Report problems with the image in this repository and problems with AzCopy itself [upstream](https://github.com/Azure/azure-storage-azcopy/issues).

## Quick start

Upload the directory `site` to a blob container, using a SAS token for the container:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/azcopy \
  copy site "https://<account>.blob.core.windows.net/<container>?<SAS>" --recursive
```

The blobs are named `site/...`; copy `"site/*"` instead to put the contents at the top of the container. Quote every URL that has a SAS token, since the token contains `&`. A few more commands:

```sh
# Download a container into ./<container>
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/azcopy \
  copy "https://<account>.blob.core.windows.net/<container>?<SAS>" . --recursive

# Make site/ in the container match the directory, deleting blobs that are not in it
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/azcopy \
  sync site "https://<account>.blob.core.windows.net/<container>/site?<SAS>" --delete-destination=true

# List the blobs in a container
docker run --rm ghcr.io/randomcontainers/azcopy \
  list "https://<account>.blob.core.windows.net/<container>?<SAS>"
```

The download goes into `./<container>/` in the same way that the upload went into `site/`; the source URL `"https://<account>.blob.core.windows.net/<container>/*?<SAS>"` puts the blobs straight into the current directory. `sync` compares the contents of `site` with what is under the destination URL and adds no directory name, so the URL ends in `/site` to match the upload. With the container URL, it would put the files at the top of the container and delete the `site/...` blobs.

Without arguments the image prints `azcopy --help`. `azcopy <command> --help` describes each command, and Microsoft's [AzCopy documentation](https://learn.microsoft.com/azure/storage/common/storage-use-azcopy-v10) covers the rest. In CI, pin a major version tag such as `10` or a digest.

## Signing in

Without a SAS token, `azcopy` signs in with Microsoft Entra ID when `AZCOPY_AUTO_LOGIN_TYPE` is set, for that one command. `-e NAME` without a value passes the variable from your shell:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  -e AZCOPY_AUTO_LOGIN_TYPE=SPN -e AZCOPY_TENANT_ID \
  -e AZCOPY_SPA_APPLICATION_ID -e AZCOPY_SPA_CLIENT_SECRET \
  ghcr.io/randomcontainers/azcopy copy site "https://<account>.blob.core.windows.net/<container>" --recursive
```

- `SPN`, a service principal: `AZCOPY_SPA_APPLICATION_ID` and `AZCOPY_TENANT_ID`, with `AZCOPY_SPA_CLIENT_SECRET`, or with `AZCOPY_SPA_CERT_PATH` and `AZCOPY_SPA_CERT_PASSWORD` for a certificate mounted into the container.
- `MSI`, the managed identity of the Azure VM or service the container runs on. `AZCOPY_MSI_CLIENT_ID` selects a user-assigned identity.
- `WORKLOAD`, workload identity federation, as in AKS: `AZURE_CLIENT_ID`, `AZURE_TENANT_ID` and `AZURE_FEDERATED_TOKEN_FILE`, with the token file mounted into the container.
- `DEVICE`, the device code flow: open the link that `azcopy` prints and enter its code.

The identity needs a data role on the storage account, such as Storage Blob Data Contributor. `azcopy login` cannot keep a session in a container: AzCopy saves it in the Linux kernel keyring, and Docker's default seccomp profile blocks the keyring system calls. `AZCLI` and `PSCRED` need the Azure CLI or Azure PowerShell, which the images do not have.

To copy from Amazon S3, pass `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`. For Google Cloud Storage, mount a service account key and point `GOOGLE_APPLICATION_CREDENTIALS` to it. The destination must be in Azure Blob Storage, and it still needs a SAS token or one of the sign-in methods above.

## Telemetry

Since version 10.34.0 AzCopy sends usage telemetry to Microsoft by default. Its `--disable-telemetry` option says the events include Azure Storage account names and pseudonymous installation and job identifiers. These images set `AZCOPY_TELEMETRY_CONNECTION_STRING` to `InstrumentationKey=00000000-0000-0000-0000-000000000000`, the all-zero key that AzCopy reads as "telemetry off", so `azcopy` sends no events and writes no installation ID. The image tests check that no installation ID appears.

To send telemetry as AzCopy does by default, pass the variable with an empty value:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" -e AZCOPY_TELEMETRY_CONNECTION_STRING= \
  ghcr.io/randomcontainers/azcopy copy site "https://<account>.blob.core.windows.net/<container>?<SAS>" --recursive
```

AzCopy then keeps a random installation ID in `/cache/.azcopy/installation_id`, which is a new one in each container unless you mount a volume at `/cache`. AzCopy does not document the all-zero key; its own end-to-end tests use it to turn telemetry off. If a release stops reading it that way, the image tests fail and that release is not published.

## Jobs and logs

`azcopy` writes a log and a job plan for every transfer to `$HOME/.azcopy`, and `HOME` is `/cache` in the images. Without a volume they are gone when the container exits. To look at the logs afterwards, or to resume an interrupted transfer, keep `/cache` in a volume:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" -v azcopy-state:/cache \
  ghcr.io/randomcontainers/azcopy copy site "https://<account>.blob.core.windows.net/<container>?<SAS>" --recursive
docker run --rm --user "$(id -u):$(id -g)" -v azcopy-state:/cache ghcr.io/randomcontainers/azcopy jobs list
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" -v azcopy-state:/cache \
  ghcr.io/randomcontainers/azcopy jobs resume <job-id> --destination-sas "<SAS>"
```

`azcopy` creates `/cache/.azcopy` with mode 0755 for the UID that first uses the volume, so use the same `--user` every time with a given volume. To keep the logs and job plans in your directory instead, pass `-e AZCOPY_LOG_LOCATION=/work/azcopy-logs -e AZCOPY_JOB_PLAN_LOCATION=/work/azcopy-plans`. Set both: with only `AZCOPY_LOG_LOCATION`, `azcopy` creates `/cache/.azcopy` with no permissions and then stops, because it cannot make its job plan folder there.

## What is in the image

- `azcopy` in `/usr/local/bin`, built without cgo, so it is statically linked and loads no libraries.
- `/etc/mime.types`, from the distro's `media-types` package on Ubuntu and `mailcap` on Alpine. `azcopy` sets the content type of uploaded files from their extension, with a small built-in table and this file.

Not included: the Azure CLI, Azure PowerShell and the AzCopy build with the GNOME keyring login (Microsoft's `azcopy_linux_se` binaries). The Go version, the AzCopy commit, the commit and SHA-256 of the Public Suffix List file and every Go module with its version and checksum are in `/usr/local/share/randomcontainers/azcopy/buildinfo`.

## Default or slim

AzCopy's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is an AzCopy release such as `10.34.1`. `<minor>` and `<major>` are its shorter forms, `10.34` and `10`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current AzCopy version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is `/cache` for every user, and `/cache` can be written by anyone. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

Uploads to NFS shares in Azure Files without `--preserve-permissions` look up the user `azcopy` runs as, to set the owner of the new files. A UID that has no entry in the image's `/etc/passwd` makes that lookup fail and `azcopy` crash, so pass `-e USER="$(id -un)"` for those uploads. UID 1000 has an entry on both distros.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new AzCopy release and are rebuilt when the base image changes. Switch to root to install more, then back. This adds `jq`, for reading the output of `azcopy list --output-type json` in a script:

```dockerfile
FROM ghcr.io/randomcontainers/azcopy:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends jq \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, start from `slim-alpine` and use `apk add --no-cache jq`. The entrypoint is `["tini", "--", "azcopy"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new AzCopy releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/azcopy/` holds the version, the source URLs, the build details, the license files, `mpl-sources/` with the Public Suffix List file, and `runtime-deps`, the list of distro packages `azcopy` needs at run time beyond those in the base image.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/azcopy:latest \
  --repo randomcontainers/azcopy --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/azcopy:latest --format '{{ json .SBOM }}'
```

AzCopy publishes prebuilt binaries but no source tarball, and its tags are not signed, so each version is pinned to the commit of its tag. The build clones the `v<version>` tag of [Azure/azure-storage-azcopy](https://github.com/Azure/azure-storage-azcopy) and stops unless the tag points to the commit recorded in `package.yml`. The Go toolchain is downloaded from Go's release server (dl.google.com, as linked from go.dev/dl) and checked against the SHA-256 in the Dockerfile, and the Go modules come from the Go module proxy and are checked against AzCopy's `go.sum`. The binary records the commit it was built from, and `buildinfo` lists it with the Go modules. The Public Suffix List file is downloaded from [publicsuffix/list](https://github.com/publicsuffix/list) at the commit pinned in `package.yml` and checked against the SHA-256 recorded there.

## Updates

The project checks the releases of [Azure/azure-storage-azcopy](https://github.com/Azure/azure-storage-azcopy/releases) every 15 minutes. A release is picked up once it is 3 days old, which leaves time to raise the Go pin (see below); pre-releases are skipped. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The Go toolchain is pinned in the Dockerfiles and raised by hand when Go publishes a security release, and each raise rebuilds the images of the current version. AzCopy often raises the Go version its `go.mod` requires in a release, sometimes to a Go release that is only days old, and such a release does not build until the pin is at least that version.

The Public Suffix List file is pinned by hand to a commit of publicsuffix/list. A release whose `golang.org/x/net` names another commit as the source of its tables stops the build until the pin is moved to that commit. A release that no longer links `golang.org/x/net/publicsuffix` stops the build until the license label and the files for the list are changed by hand.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  --build-arg PUBLIC_SUFFIX_LIST_VERSION=<version> \
  --build-arg PUBLIC_SUFFIX_LIST_SHA256=<sha256> \
  -t azcopy:local .
```

`PUBLIC_SUFFIX_LIST_VERSION` and `PUBLIC_SUFFIX_LIST_SHA256` are the `version` and `sha256` of the `public-suffix-list` entry under `extra-artifacts` in `package.yml`. Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of packages Go builds in parallel. The build downloads a few hundred megabytes of Go modules.

## Licenses

AzCopy is released under the [MIT license](https://github.com/Azure/azure-storage-azcopy/blob/main/LICENSE), and its `NOTICE.txt` lists the notices of the third-party code in Microsoft's builds. The binary also contains:

- the Go standard library (BSD-3-Clause);
- the Go modules AzCopy depends on, under MIT, Apache-2.0, BSD-3-Clause and BSD-2-Clause, among them the Azure SDK for Go, the Google Cloud Storage client and the MinIO client for S3;
- testify's copy of go-spew (ISC), linked because AzCopy uses testify's `assert` package outside its tests;
- Mozilla's [Public Suffix List](https://publicsuffix.org/) (MPL-2.0), which `golang.org/x/net/publicsuffix` embeds as tables generated from the list. The MinIO client uses that package; the rest of `golang.org/x/net` is BSD-3-Clause.

The image's license label is `MIT AND Apache-2.0 AND BSD-3-Clause AND BSD-2-Clause AND ISC AND MPL-2.0`. The license files, AzCopy's `NOTICE.txt` among them, are in `/usr/local/share/randomcontainers/azcopy/licenses/`, with one directory per Go module. `golang.org/x/net/publicsuffix/` there holds the list's notice and the text of the MPL 2.0, which this repository keeps in [licenses](licenses) because the module does not include them. Links to the source are in `/usr/local/share/randomcontainers/azcopy/source`. The MPL 2.0 requires whoever distributes the binary to make the source of the list available, so the image also carries the list itself, as `/usr/local/share/randomcontainers/azcopy/mpl-sources/public_suffix_list.dat`. It is `public_suffix_list.dat` from [publicsuffix/list](https://github.com/publicsuffix/list) at the commit that `publicsuffix/table.go` in the linked `golang.org/x/net` names as the source of its tables. That commit is the `version` of `public-suffix-list` in `package.yml`, and the file's URL in `source` contains it. `publicsuffix/gen.go` in that `golang.org/x/net`, whose version and checksum `buildinfo` lists, generates the tables from the file. The packages from Ubuntu or Alpine keep their own licenses. The SBOM lists them.

Microsoft and Azure are trademarks of the Microsoft group of companies. The files in this repository are available under the MIT license, see [LICENSE](LICENSE), except the license texts in `licenses/`, which keep their own terms.

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
