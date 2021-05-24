---
layout: page
permalink: /cpman/
title: Container Project Manager
---

## Container Project Manager

## Purpose

Allow projects to be built, debugged, ran, and deployed in containers (through either docker compose or kubernetes+helm) configured through a single yaml file (or multiple, if wanted).

## Requirements
- Linux (only testing is done on Debian-based system, ymmv on other distros).
- Docker
- Kubernetes + Helm for K8S configuration, or
- Docker-compose for docker-compose configuration



## Yaml Breakdown 
---
### Expandable variables (subject to change)

| Name | Description |
|---|---|
| [ENV] | environment name |
| <EXPAND_ENV> | expands environment variables to k8s format (name/value object) |
| [VERSION] | version value | 
| [DEBUG] | add "-debug" if debug is true | 
| [CONTAINER_NAME] | container name |
| [CONTAINER_VERSION] | container version |
| [BIN_DIRECTORY] | directoryInContainer value |
| [BIN_PATH] | binPath value |
| [BIN_FILE] | binFile value | 
| [DATA_DIR] | location of output data directory (hostBasePath/name[ENV][DEBUG]/data) |
| [CURR_DIR] | current working directory |
| [WEBPACK_DIR] | current iteration of specified webpack directories |
| [OPTIMIZE] | optimize value | 
| ./ | current working directory |

<br/>


### Images Manifest
Image types:

| Name | Description |
| --- | --- |
| build | container configuration for compiling project uses docker run |
| debug | container configuration for an environment where debug = true |
| run | container configuration for an environment where debug = false |
| webpack | container configuration for environment where webpack is true, will run the docker command and postrun command for each webpack directory specified. |

<br/>

Options:

| Name | Description |
| --- | --- |
| name | Docker image name (including version tag) |
| buildScript | (ba)sh script location to build image if it does not exist |
| dockerCmd | full command line to execute |
| postOptimizeCmd | run if optimize = true |
| env | environment values to pass to selected environment |
| chartValues | helm chart values to pass to selected environment |

Example:

```yaml
build: 
  name: rust-alpine-build:4.17.21
  buildScript: ./rust-alpine-build/build.sh
  dockerCmd: |
    docker run --rm -t --user $UID:$UID -v "[CURR_DIR]":/home/rust/src -v "$HOME/.cargo":/home/rust/.cargo rust-alpine-build:4.17.21 /bin/sh -c "export CARGO_TARGET_DIR=\"/home/rust/src/target-musl\"; cargo build [OPTIMIZE] --target x86_64-unknown-linux-musl"
  postOptimizeCmd: |
    echo "Running cargo strip"; 
    export CARGO_TARGET_DIR="[CURR_DIR]/target-musl/x86_64-unknown-linux-musl"; 
    cargo strip
debug: 
  name: rust-alpine-debug:4.17.21
  buildScript: ./rust-alpine/build-lldb.sh
  env: 
    BIN_DIR: "[BIN_DIRECTORY]"
    BIN_NAME: "[BIN_FILE]"
    DEBUG_AND_RUN: "true"
    DEBUG: "true"
  chartValues:
    securityContext:
      capabilities:
        add:
          - SYS_PTRACE
run: 
  name: rust-alpine:4.17.21
  buildScript:  ./rust-alpine/build.sh
  env: 
    BIN_DIR: "[BIN_DIRECTORY]"
    BIN_NAME: "[BIN_FILE]"
webpack:
  name: js-obfuscator
  buildScript: ./js-obfuscator/build.sh 
  dockerCmd: |
    docker run --rm --user $UID:$UID -v [WEBPACK_DIR]:/webpack-dir js-obfuscator \
    javascript-obfuscator --compact true --self-defending false --source-map true --source-map-mode separate /webpack-dir
  postRunCmd: |
    /bin/bash <<'BREAK_ALL'
    #!/bin/bash
      source_dir="[WEBPACK_DIR]/../js_sourcemaps"
      mkdir -p "$source_dir"
      find "[WEBPACK_DIR]" -name '*-obfuscated.js' | while read name; do
        newname="${name%-obfuscated.js}.js"
        mv "$name" "$newname"
        mapfile=$(basename -- "$name").map
        newmap=${mapfile%-obfuscated.js.map}.js.map
        awk -i inplace -v o="$mapfile" -v n="$newmap" '{gsub(o,n)}1' "$newname"
        mv "$name.map" "$source_dir/$newmap"
      done
    BREAK_ALL
```
## K8s manifest

| Name | Description |
| --- | --- | 
| hostBasePath | The base path for deploying an environment (will create subdirectory under this when deploying) |
| directoryInContainer | directory in resultant container where bin resides |
| binFile | name of binary | 
| chart | location of helm chart for applying resultant values |
| volumes | array of host volumes to copy to output directory, if copyVolumes = true in environment |
| sharedEnv | shared environment variables for all environments | 
| sharedChartValues | shared helm chart values for all environments |


### Environment options

| Name | Description |
| --- | --- |
| debug | if true, runs environment using debug image | 
| postRunCmd | bash command to run after deploying (e.g. a web browser launching the project page) |
| optimize | if true, sets [OPTIMIZE] variable to --release |
| chartValues | helm chart values for selected environment |
| copyVolumes | copies volumes to output data directory |
| copyBin | if true, copies bin to output data directory |
| webpack | array of directories, will run the webpack image for each using the [WEBPACK_DIR] variable.  Must be a copied volume (copyVolumes=true) | 

### Shared variable stacks

Environment and Chart Variables are combined in the order:

Image Variables > Shared Variables > Environment specific variables.


### Example:

```yaml
version: "1.0.7"
name: sample-app
imagesManifest: ./.git-submodules/docker-rust-base-images/images.yaml
k8s:
  hostBasePath: ./target-k8s
  directoryInContainer: /site
  binFile: sample_app
  chart: ./.git-submodules/cpman/charts/default
  volumes: 
    - wwwroot
    - pages
  sharedEnv:
    TZ: America/Los_Angeles
    LISTEN_ADDRESS: "0.0.0.0:8088"
    RUST_ENVIRONMENT: "[ENV]"
    SITE_VERSION: "[VERSION]"
  sharedChartValues: 
    fullnameOverride: "sample-app-[ENV][DEBUG]"
    containerSpec:
      terminationGracePeriodSeconds: 0
    image: 
      repository: "[CONTAINER_NAME]"
      pullPolicy: Never
      tag: "[CONTAINER_VERSION]"
    env: <EXPAND_ENV>
  development: 
    debug: true
    postRunCmd: "/bin/chromium http://localhost/sample_app_page"
    optimize: false
    copyBin: false
    binPath: ./target-musl/x86_64-unknown-linux-musl/debug/sample_app
    chartValues:
      volumes:
      - type: hostPath
        path: ./wwwroot
        mountPath: /site/wwwroot
        readOnly: true
      - type: hostPath
        path: ./pages
        mountPath: /site/pages
        readOnly: true
      - type: hostPath
        path: "[BIN_PATH]"
        mountPath: /site/sample_app
        mountType: File
      namespace: vlan7
      service:
        ports:
          - name: http
            port: "8088"
            hostPort: "80"
          - name: gdb-port
            port: "5001"
            hostPort: "5001"
          - name: gdb-listen-port
            port: "19111"
            hostPort: "19111"
  production:
    binPath: ./target-musl/x86_64-unknown-linux-musl/release/sample_app
    postRunCmd: "/bin/chromium http://localhost:9008/sample_app_page"
    copyVolumes: true
    copyBin: true
    debug: false
    optimize: true
    chartValues:
      volumes:
      - type: hostPath
        path: "[DATA_DIR]/wwwroot"
        mountPath: /site/wwwroot
        readOnly: true
      - type: hostPath
        path: "[DATA_DIR]/pages"
        mountPath: /site/pages
        readOnly: true
      - type: hostPath
        path: "[DATA_DIR]/sample_app"
        mountPath: /site/sample_app
        mountType: File
      namespace: vlan7
      service:
        ports:
          - name: http
            port: "8088"
            hostPort: "9008"
    webpack:
      - pages
```

## Command line arguments

| Name | Short | Description |
| --- | --- | --- | 
| --file |-f | yaml file | 
| --environment | -e | select environment from yaml file | 
| --build | -b | Build (Compile) selected environment |
| --run | -r | Package and run environment |


## TODO

- better error / problem handling 
- clean up / more descriptive variables
- implement docker-compose combinations
- implement remote key/values to push deployments over ssh/sftp
- more testing


