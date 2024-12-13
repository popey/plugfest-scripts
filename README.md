# Plugfest 

Resources for creating our SBOMs for submission to the [SBOM Harmonization Plugfest 2024](https://resources.sei.cmu.edu/news-events/events/sbom/participate.cfm). This attempts to fully automate the creation of the SBOMs using [Syft](https://github.com/anchore/syft) to output in various formats.

The plugfest organisers requested 'a couple' of SBOMs. We figured this would be a good opportunity to exercise all of the output format options in Syft, so we generated every supported format for every repo.

SBOMs are generated in Syft-versioned folders, so this could be run multiple times with different releases of Syft for comparison. For the final submission, I used [Syft](https://github.com/anchore/syft/) [v1.18.0](https://github.com/anchore/syft/releases/v1.18.0).

## Introduction

This repo contains scripts and other resources to automate the creation of SBOMs. SBOMs are generated using whatever Syft binary is referenced in the `PLUG_SYFT_CMD` variable. All defaults were used in the execution of Syft, other than the `--enrich` option, detailed below under "Enrichment"

* urls.txt contains the links pulled from the Plugfest [partitipate](https://resources.sei.cmu.edu/news-events/events/sbom/participate.cfm) page.

## Syft Methodology

Syft is a fully-offline, open-open source SBOM generator. It requires no login or API key to operate. 

Syft does not analyze:

- Source code for licenses
- Search for other SBOMs within the search scope
- Binary analysis

### Enrichment

Syft has an `--enrich` option (which is off by default) that can leverage some online resources to embellish SBOMs.

`      --enrich stringArray                        enable package data enrichment from local and online sources (options: all, golang, java, javascript)`

This results in potentially greater detail in the generated SBOM.

For example, (at the time of writing) compare the line count of the un-enriched vs enriched versions of the CycloneDX SBOM for the container build of `dependency-track`

```bash
$ wc -l sboms/binary/syft/1.18.0/dependency-track/dependencytrackapiserver/cyclonedx-json.json sboms/binary/syft/1.18.0/dependency-track/dependencytrackapiserver/cyclonedx-json_enriched.json 
  5494 sboms/binary/syft/1.18.0/dependency-track/dependencytrackapiserver/cyclonedx-json.json
  6117 sboms/binary/syft/1.18.0/dependency-track/dependencytrackapiserver/cyclonedx-json_enriched.json
```

Here's a snippet, showing the side-by-side `diff` of an un-enriched vs enriched CycloneDX SBOM.

**Note:** Spaces truncated for readabiliy.

```bash
$ diff -y sboms/binary/syft/1.18.0/dependency-track/dependencytrackapiserver/cyclonedx-json.json sboms/binary/syft/1.18.0/dependency-track/dependencytrackapiserver/cyclonedx-json_enriched.json | grep '>' | head -n 15 | awk '{$1=$1};1'
> {
> "license": {
> "name": "Apache 2",
> "url": "http://www.apache.org/licenses/LICENSE-2.
> }
> }
> ],
> "cpe": "cpe:2.3:a:org.sonatype.oss:JUnitParams:1.1.1:*:
> "name": "syft:cpe23",
> "value": "cpe:2.3:a:JUnitParams:JUnitParams:1.1.1:*
> },
> {
> "name": "syft:cpe23",
> "value": "cpe:2.3:a:sonatype:JUnitParams:1.1.1:*:*:
> },
```

Do note however, that Syft was run both with and without the `--enrich` option, but only some projects will have seen any gains, as currently a limited set of online resources are queried with this feature.

## Scripts

The scripts were split up to enable the atomic parts of the process to be re-run without having to redo previous steps. We were unsure if SBOMs for source *and* binary builds were expected, so added that as an option to the submission.

I added rough timings to complete each script on my remote, well-connected, mostly-idle, Debian server in Germany.

* `1_get_targets`         Grab all the source code we will sbom (~1 min)
* `2_gen_source_sboms`    Generate SBOMs from the cloned repos (~7 min)
* `3_build_all_targets`   Attempt to build binary artifacts or containers (optional) (~4 min)
* `4_gen_binary_sboms`    Generate SBOMs from binary artifacts (optional) (~16 min)
* `5_validate_sboms`      Use external tools to validate SBOMs (optional) (~10 min)

## Files Generated

The defaults will result in a folder called `./sboms` containing subdirectories, being generated, in which the SBOM files are created.

* Directory layout: `./sboms/source/syft/$syftver/$org/$repo/$format[_enriched].$extension`

| parameter | meaning | example |
| -- | -- | -- |
| `$syftver` | Syft version used | `1.18.0` |
| `$org` | repository owner | `httpie` |
| `$repo` | repository being scanned | `cli` |
| `$format` | SBOM file format | `spdx-json` |
| `_enricheed` | Whether syft `--enrich` option used | `_enriched` |
| `$extension` | File extension | `.json` |

For example:

* Example: `./sboms/source/syft/1.18.0/httpie/cli/spdx-json.json`
* Enriched SBOM: `./sboms/source/syft/1.18.0/httpie/cli/spdx-json_enriched.json`
* SBOM validation: `./sboms/source/syft/1.18.0/httpie/cli/spdx-json.json_pyspdxtools.txt`

## SBOM formats

Syft supports numerous output formats. The SBOM generation scripts attempt to output in every supported format.

* cyclonedx-json
* cyclonedx-xml
* github-json
* spdx-json
* spdx-tag-value
* syft-json
* syft-table
* syft-text

## Enrichment

Syft supports an `--enrich` option. Each SBOM was generated both with and without this option enabled to illustrate the difference.

## Environment

I created and tested these on an x86_64 Linux host running Debian 12 (bookworm). YMMV. Syft 1.18.0 was used as the latest version available at the time. However, this script can (in theory) be run with any release of Syft. This may be useful to compare the SBOMs generated by different releases.

**Note:** We identified a bug in Syft as part of the plugfest 2024 process, which resulted in a new release of Syft, fixing that issue. So if nothing else, this has been a worthwhile exercise for our users.

## Pre-requisites

### Source only SBOMs

When generating SBOMs based only the cloned source repos, the following tools are required by the scripts. I passed all Syft JSON output through `jq`, and xml through `xmllint --format` before writing to files, to make the files human readable, rather than compressed on one line. *No other external transformation was performed.*

* syft
* git
* jq
* xmllint

### Binary SBOMs

When also generating binary SBOMs (and thus building the code in the repos), also install these tools.

* docker
* docker-compose
* go
* gradle, openjdk-8-jdk (for minecolonies, docs recommend 8, I used openjdk-17-jdk on Debian)
* cmake (for opencv)
* cargo, rustup, rust => 1.64.0 (for hexyl)
* uv (for the venv/pip install of httpie)

## Variables:

The following environment variables can be set prior to running the scripts, or edited directly in each script:

e.g. `PLUG_DOCKER_CMD=/usr/bin/docker ./3_build_all_targets`

| Script(s) | Variable | Meaning |
| - | - | - |
| `1_get_targets` <br> `2_gen_source_sboms` <br> `3_build_all_targets` <br> `4_gen_binary_sboms` | `PLUG_REPOS_DIR` | Path of cloned repos |
| `1_get_targets` <br> `3_build_all_targets` | `PLUG_GIT_CMD` | Path to `git` command |
| `1_get_targets` | `PLUG_URL_LIST` | Path to `urls.txt` containing repos to clone |
| `2_gen_source_sboms` <br> `4_gen_binary_sboms` | `PLUG_SYFT_CMD` | Path to `syft` command  |
| `2_gen_source_sboms` <br> `4_gen_binary_sboms` | `PLUG_JQ_CMD` | Path to `jq` command  |
| `2_gen_source_sboms` <br> `4_gen_binary_sboms` | `PLUG_OUTPUT_BASE` | Path to generate SBOMs into |
| `2_gen_source_sboms` <br> `4_gen_binary_sboms` | `PLUG_SBOM_FORMATS` | List of supported formats to output |
| `3_build_all_targets` | `PLUG_DOCKER_CMD` | Path to `docker` command |
| `3_build_all_targets` | `PLUG_UV_CMD` | Path to `uv` command |
| `3_build_all_targets` | `PLUG_DOCKER_COMPOSE_CMD` | Path to `docker-compose` command |
| `5_validate_sboms` | `PLUG_PYSPDXTOOLS_CMD` | Path to `pyspdxtools` tool |
| `5_validate_sboms` | `PLUG_SBOM_UTILITY_CMD` | Path to `sbom-utiliy` tool |
| `5_validate_sboms` | `PLUG_SBOMS_BASE` | Path to generated SBOMs |
| `5_validate_sboms` | `PLUG_VALIDATION_SUMMARY` | Path to `README.md` to generate|

## Run

### Get Targets

`./1_get_targets`

Reads in `$PLUG_URL_LIST` (`urls.txt`) then clones the listed repos to `$PLUG_REPOS_DIR` (`./repos`), then checks out the required git commit.

### Generate Source SBOMs

`./2_gen_source_sboms`

Loops through each cloned repo in `$PLUG_REPOS_DIR` (under `org/repo`) to generate many formats of SBOM for each repo. Uses syft binary defined by `PLUG_SYFT_CMD` in the script or environment.

### Build All Targets

`./3_build_all_targets`

Loops through each cloned repo in `$PLUG_REPOS_DIR` (under `org/repo`) to create binary artifacts of each package. That might be single binaries or containers.

**Note:** We decided to build using whatever docker container configuration was available in the repo. If none was present, we used best-endeavour to build each project as a user might.

The following table summarises how each package is built, if at all:

| Package | Build tool |
| ------- | ---------- |
| DependencyTrack/dependency-track | docker |
| gin-gonic/gin | Not built as it's a library |
| httpie/cli | pip install |
| jqlang/jq | docker |
| ldteam/minecolonies | gradle |
| opencv | cmake |
| PHPMailer/PHPMailer | Not built as it's php |
| sharkdp/hexyl | cargo |
| snyk-labs/nodejs-goof | docker |

### Generate Binary SBOMs

`./4_gen_binary_sboms`

Builds SBOMs of each binary built in the previous stage. This leverages Syft's common usage pattern of scanning docker images, or directories which contain binary assets.

### Validate SBOMs

`./5_validate_sboms`

Runs two SBOM validation tools to check some of the SBOM formats. 

* [CycloneDX/sbom-utility](https://github.com/CycloneDX/sbom-utility) v0.17.1-pre to validate both SPDX and CycloneDX formatted SBOMs. 
* [spdx/pyspdxtools](https://github.com/spdx/tools-python) to validate SPDX formatted SBOMs.

**Note 1:** We were not able to validate the XML files, as there were no validators provided for those SBOM formats.

**Note 2:** We also tried the NTIA validation site, which complains about data that isn't present, or is unknown in the SBOM.

**Note 3:** There is some disparity between the results of the two tools. For example the `spdx-json_enriched.json` file gets a clean bill of health from `sbom_utility` but fails `pyspdxtools` validation.

Validation results (if any) are placed side-by-side with the SBOM file in a text file called `(sbom_file)_pyspdxtools.txt`, and `(sbom_file)_sbom_utility.txt`.

## TODO

- [X] Remove running validators against unsupported files
- [X] Describe syft methodology
- [X] Show a diff (e.g. d-t table output)
- [X] Pipe output through jq
- [ ] Consider removing xml sboms if we can't validate them
- [X] Check with Chris for a new release including fix for path
- [ ] Check with Alex to check the results are generally sane
- [X] Describe how we validated, tools missing for validating 
- [ ] Investigate additional offline version of https://tools.spdx.org/app/validate - maybe ping Keith to ask if he knows where it is (pyspdxtool is an equivalent / proxy for the java one)
- [X] Investigate NTIA (did that, we don't spit out spdx xml) - but we could upload json, but it moans about data that's not present (call this out in the validation section)
- [ ] Add XML linter - https://stackoverflow.com/questions/16090869/how-to-pretty-print-xml-from-the-command-line
- [ ] Ping to the team to review
