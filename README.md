# libxml2

Container images with `xmllint` and `xmlcatalog` from [libxml2](https://gitlab.gnome.org/GNOME/libxml2) and `xsltproc` from [libxslt](https://gitlab.gnome.org/GNOME/libxslt), the XML libraries of the GNOME project. `xmllint` checks that XML files are well formed, validates them against DTDs, XML Schemas and RELAX NG schemas, indents and canonicalizes them and evaluates XPath expressions; `xsltproc` applies XSLT 1.0 stylesheets. Both libraries are compiled from GNOME's release tarballs on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when libxml2 publishes a release and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the GNOME project or the libxml2 and libxslt maintainers. Report problems with the image in this repository, and bugs in the tools at [libxml2](https://gitlab.gnome.org/GNOME/libxml2/-/issues) or [libxslt](https://gitlab.gnome.org/GNOME/libxslt/-/issues).

## Quick start

Validate a document against an XML Schema:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/libxml2 --noout --schema schema.xsd document.xml
```

The entrypoint runs `xmllint` under `tini` in `/work`, so file names are relative to the directory you mount. It prints `document.xml validates` to stderr and exits with 0. The exit status is 3 when a document is not valid, 4 when it is not well formed or cannot be read, and 5 when the schema has errors. A few more commands:

```sh
# Check that a file is well formed; nothing is printed when it is
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/libxml2 \
  --noout document.xml

# Validate against a RELAX NG schema, or against a DTD that the file does not reference
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/libxml2 \
  --noout --relaxng schema.rng document.xml
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/libxml2 \
  --noout --dtdvalid schema.dtd document.xml

# Indent a file and write the result to another file
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/libxml2 \
  --format --output formatted.xml document.xml

# Print the result of an XPath expression
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/libxml2 \
  --xpath 'string(/feed/title)' feed.xml

# Write the canonical form (C14N 1.0) of a file, for example before comparing two files
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/libxml2 \
  --c14n document.xml > canonical.xml

# Read from stdin, without a mount
docker run --rm -i ghcr.io/randomcontainers/libxml2 --format - < document.xml
```

`--html` parses HTML instead of XML, for example to run `--xpath` on a web page. The [xmllint manual](https://gnome.pages.gitlab.gnome.org/libxml2/xmllint.html) describes every option.

## XSLT

Run `xsltproc` with `--entrypoint`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint xsltproc \
  ghcr.io/randomcontainers/libxml2 -o output.html style.xsl document.xml
```

`--stringparam name value` sets a stylesheet parameter. libxslt implements XSLT 1.0 and XPath 1.0, plus the [EXSLT](https://exslt.github.io/) extensions for common, math, sets, strings, dates and times, dynamic and functions. The EXSLT crypto functions are not built. See the [xsltproc manual](https://gnome.pages.gitlab.gnome.org/libxslt/xsltproc.html) for the other options.

## Catalogs and remote DTDs

libxml2 removed its HTTP client in version 2.15, so `xmllint` and `xsltproc` do not download DTDs, schemas or stylesheets named by an `http://` URL. Without a local copy, `xmllint --valid` reports `failed to load` and `no DTD found` for a document like this one:

```xml
<!DOCTYPE note SYSTEM "http://example.org/dtd/note.dtd">
```

An XML catalog maps such URLs to local files. `xmllint` and `xsltproc` look up the DTDs, schemas and stylesheets they load in the catalogs listed in `XML_CATALOG_FILES`, or in `/etc/xml/catalog` when the variable is not set. The image has no catalog of its own. Create one with `xmlcatalog`, using paths as the container sees them:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint xmlcatalog \
  ghcr.io/randomcontainers/libxml2 --create --noout catalog.xml
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint xmlcatalog \
  ghcr.io/randomcontainers/libxml2 --noout \
  --add rewriteSystem http://example.org/dtd/ file:///work/dtd/ catalog.xml
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" -e XML_CATALOG_FILES=/work/catalog.xml \
  ghcr.io/randomcontainers/libxml2 --noout --valid document.xml
```

`rewriteSystem` maps every URL under a prefix; `--add system <url> <file>` maps a single one, and `rewriteURI` and `uri` entries do the same for schemas and stylesheets. Instead of setting `XML_CATALOG_FILES`, you can mount a catalog at the default location with `-v "$PWD/catalog.xml:/etc/xml/catalog:ro"`. The [xmlcatalog manual](https://gnome.pages.gitlab.gnome.org/libxml2/xmlcatalog.html) lists the entry types.

## What is in the image

- `xmllint` and `xmlcatalog` from libxml2, `xsltproc` from libxslt, and their shared libraries (`libxml2`, `libxslt` and `libexslt`) in `/usr/local`.
- Validation with DTDs, XML Schema 1.0 and RELAX NG, XPath 1.0, XPointer, XInclude, Canonical XML, XML catalogs and the HTML parser. `xmllint --version` lists the compiled features.
- libxml2 converts UTF-16, ISO-8859-1, windows-1252 and ASCII itself. Other character encodings go through the C library's iconv: glibc on Ubuntu, with its extra converters from `libc-gconv-modules-extra`, and musl on Alpine. musl knows fewer encodings; for example, it reads GB18030 and Big5 but cannot write them with `--encode`. When musl writes a legacy encoding, it also replaces a character that the encoding lacks with `*`, where glibc lets libxml2 write a character reference such as `&#1071;`.

Apart from their own libraries, the programs load only the C library and its iconv converters. On Ubuntu, they find their libraries in `/usr/local/lib` through their RUNPATH.

Not included: Schematron (off by default since libxml2 2.15), the Python bindings, reading compressed `.gz` files (needs zlib), ICU, line editing and history in `xmllint --shell`, the EXSLT crypto functions, libxslt plugins, the C headers, `xml2-config` and `xslt-config`, the pkg-config and CMake files, and the man pages. The configure flags are in `/usr/local/share/randomcontainers/libxml2/buildinfo`.

## Default or slim

libxml2's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a libxml2 release such as `2.15.4`. `<minor>` and `<major>` are its shorter forms, `2.15` and `2`, and follow the newest release in that series. Each row lists the default tag and its `slim` twin, which point to the same image.

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

The version in the tags is the libxml2 version. The libxslt version is in `/usr/local/share/randomcontainers/libxml2/buildinfo` and in `xsltproc --version`.

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current libxml2 version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

`xmllint --output` and `xsltproc -o` write into the mounted directory, so it must be writable. Validation and `--noout` only read, and work with a read-only mount (`-v "$PWD:/work:ro"`).

## Untrusted files

libxml2's own README advises against using it on untrusted data. If you have to, give the container as little as possible:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  --network none --read-only --cap-drop ALL --security-opt no-new-privileges \
  --memory 1g --pids-limit 64 \
  ghcr.io/randomcontainers/libxml2 --noout --schema schema.xsd untrusted.xml
```

A document can name other files through external entities, a DTD or XInclude. `xmllint` reads them only with options such as `--noent`, `--loaddtd`, `--valid` or `--xinclude`, and then it can read any file the container can, including everything you mount. Do not pass `--huge`, which lifts the parser's internal size limits.

`xsltproc` loads DTDs and substitutes external entities by default, in the input documents and in the stylesheet, and no option turns the substitution off (`--novalid` only skips the external DTD). An untrusted document or stylesheet can therefore read any file the container can. A stylesheet can also read files with `document()` and write files with `exsl:document`; `xsltproc --nowrite` forbids the writes. Run `xsltproc` with the same `docker run` options as above, adding `--entrypoint xsltproc`, and mount only the files it needs, read-only.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new libxml2 release and are rebuilt when the base image changes. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/libxml2:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends make \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, start from `slim-alpine` and use `apk add --no-cache make`. The entrypoint is `["tini", "--", "xmllint"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new libxml2 releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Ubuntu's `libxml2-16` and `libxslt1.1` packages have the same sonames as the libraries here. If a package you add depends on them, that package's programs may load either copy; `xmllint`, `xmlcatalog` and `xsltproc` keep loading the ones in `/usr/local/lib`.

On Alpine, `libxslt.so.1` and `libexslt.so.0` have the same sonames as the libraries in Alpine's `libxslt` package, and musl searches `/usr/local/lib` before `/usr/lib`. An Alpine package that uses libxslt, such as `xmlstarlet`, `php84-xsl` or `py3-lxml`, would load Alpine's `libxml2.so.2` and, through the libxslt here, also `libxml2.so.16` from `/usr/local/lib`. Two versions of libxml2 in one process can crash it. Do not add packages that depend on Alpine's libxslt to the `slim-alpine` image; use `slim-ubuntu` for them. Packages that use only Alpine's libxml2 are fine, because its soname differs from the one here.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/libxml2/` holds the version, the source URLs, the build options, the license files and `runtime-deps`, the list of distro packages the tools need at run time. On Ubuntu it lists `libc-gconv-modules-extra`, the glibc converters for most encodings, which the programs load at run time; on Alpine it names only musl, which the base image already has.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/libxml2:latest \
  --repo randomcontainers/libxml2 --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/libxml2:latest --format '{{ json .SBOM }}'
```

GNOME publishes a `.sha256sum` file next to each tarball but no signatures. Before compiling, the build checks the libxml2 and libxslt tarballs against the SHA-256 values recorded in `package.yml`. The SBOM covers distro packages only. libxml2 and libxslt are compiled from source, and their versions and tarball hashes are in `/usr/local/share/randomcontainers/libxml2/buildinfo`.

## Updates

The project checks the [releases of GNOME/libxml2](https://gitlab.gnome.org/GNOME/libxml2/-/releases) on GNOME's GitLab every 15 minutes. A release is picked up once it is 24 hours old. Its tarball is checked against the `libxml2-<version>.sha256sum` file on download.gnome.org, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built, and fixes released only for an older series are not built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

libxslt is pinned to a version in `package.yml` and does not follow its upstream releases automatically. A new libxslt release is checked against its `.sha256sum` file and pinned by hand.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  --build-arg LIBXSLT_VERSION=<version> \
  --build-arg LIBXSLT_SHA256=<sha256> \
  -t libxml2:local .
```

`LIBXSLT_VERSION` and `LIBXSLT_SHA256` are the `version` and `sha256` of the `libxslt` entry under `extra-artifacts` in `package.yml`. Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

libxml2 is licensed under the MIT license. Two of its source files, `dict.c` and `list.c`, carry an ISC-style notice instead (SPDX `ISC-Veillard`), and the Timsort code in `timsort.h`, used by XPath, has an MIT notice of its own from Christopher Swenson and others. libxslt and libexslt are licensed under an MIT license with an added clause that the authors' names may not be used in advertising without permission (SPDX `X11`). All of it is compiled into the libraries in the image.

`/usr/local/share/randomcontainers/libxml2/licenses/` holds the `Copyright` files of both projects as `COPYING.libxml2` and `COPYING.libxslt`, and the notices from `dict.c`, `list.c` and `timsort.h` as `COPYING.dict`, `COPYING.list` and `COPYING.timsort`. The source tarball URLs are in `/usr/local/share/randomcontainers/libxml2/source`. The image's license label is `MIT AND ISC-Veillard AND X11`. The Ubuntu and Alpine packages in the image keep their own licenses; the SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
