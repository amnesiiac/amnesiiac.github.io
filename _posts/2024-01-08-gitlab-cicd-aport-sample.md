---
layout: post
title: "gitlab cross project co-action ci/cd usecase analysis (ci/cd)"
author: "melon"
date: 2024-01-08 22:06
categories: "2024"
tags:
  - gitlab
---

this article mainly introduce a gitlab ci/cd toy example, showcase the basic methodology for ci/cd co-action
across multi-project under the same gitlab group.

<hr>

### # gitlab ci/cd code with comments
1 rebornlinux/aport/.gitlab-ci.yaml:

```text
# global var used in ci pipeline
variables:
  IMG_TAG: "$CI_ARTIFACTORY_DIR"
  GIT_STRATEGY: clone
  GIT_DEPTH: "1"

# avoid using any prompts for passwd before git operation
default:
  before_script:
    - |-
      cat << EOF > ~/.gitconfig
      [url "https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlabe1.ext.net.xxxxx.com"]
          insteadOf = https://gitlabe1.ext.net.xxxxx.com
      [safe]
          directory = ${CI_PROJECT_DIR}
      EOF

# enumerate stages used
stages:
  - verify
  - bump
  - build
  - buildmain-x86_64
  - bootstrap
  - buildmain
  - gen-toolchain
  - buildrepos
  - mk-release
  - customize-artifacts
  - updateindex

# image for baisc infra apk produce
.basedevimg:
  image:
    name: rebornlinux-docker-local.artifactory-blr1.int.net.xxxxx.com/rebornlinux/basedev:$IMG_TAG

# image for cross compilation toolchain apk produce
.crosstoolimg:
  image:
    name: rebornlinux-docker-local.artifactory-blr1.int.net.xxxxx.com/rebornlinux/crosstool:$IMG_TAG

# image for utility toolchain apk produce
.toolchainimg:
  image:
    name: rebornlinux-docker-local.artifactory-blr1.int.net.xxxxx.com/rebornlinux/toolchain:$IMG_TAG

# config for gitlab default artifacts for log stash (debug)
.artifacts:
  artifacts:
    paths:
      - logs/
    expire_in: 1 month
    when: always

lint:
  stage: verify
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  extends: .toolchainimg
  interruptible: true
  script:
    - echo "[Todo:] move lint script from alpinelinux/apkbuild-lint-tools image to rebornlinux/toolchain image"

# bump pkg by merge request via gitlab api
bump_pkg:
  stage: bump
  extends:
    - .toolchainimg
  script:
    - ci/bump_pkg.sh
  rules:
    - if: $CI_PIPELINE_SOURCE == "pipeline"

# build stage
# template job of build stage
.build:
  stage: build
  # rely on toolchainimg & artifacts settings
  extends:
    - .toolchainimg
    - .artifacts
  timeout: 4 hours
  rules:
    - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH && $CI_PIPELINE_SOURCE == 'push'

# build-${arch} jobs: compile the updated sw APKBUILD in supported 4 archs (extend the build template)
build-x86_64:
  extends: .build
  script:
    - ci/build.sh x86_64

build-aarch64:
  extends: .build
  script:
    - ci/build.sh aarch64

build-mips64:
  extends: .build
  script:
    - ci/build.sh mips64

build-ppc:
  extends: .build
  script:
    - ci/build.sh ppc

build-ppc64:
  extends: .build
  script:
    - ci/build.sh ppc64

# buildmain template, rely on basedevimg (manual trigger & commit tag is modified)
.buildmain:
  rules:
    - when: manual
    - if: "$CI_COMMIT_TAG"
  extends:
    - .basedevimg
    - .artifacts
  timeout: 480 hours
  # use specific gitlab runners for this jobs
  tags:
    - docker-aports

# buildmain jobs, extend buildmain template
buildmain-x86_64:
  stage: buildmain-x86_64
  extends: .buildmain
  needs: []
  script:
    - REPOS="main" CI_APORTS_ABUILD_PACKAGES="all/all" ci/buildrepos.sh x86_64

.bootstrap:
  extends:
    - .basedevimg
    - .artifacts
  stage: bootstrap
  rules:
    - when: manual
    - if: "$CI_COMMIT_TAG"
      needs: [buildmain-x86_64]
  timeout: 6 hours
  tags:
    - docker-aports

bootstrap-aarch64:
  extends: .bootstrap
  script:
    - ci/bootstrap.sh aarch64

bootstrap-mips64:
  extends: .bootstrap
  script:
    - ci/bootstrap.sh mips64

bootstrap-ppc:
  extends: .bootstrap
  script:
    - ci/bootstrap.sh ppc

bootstrap-ppc64:
  extends: .bootstrap
  script:
    - ci/bootstrap.sh ppc64

buildmain-aarch64:
  stage: buildmain
  extends: .buildmain
  needs: [bootstrap-aarch64]
  script:
    - REPOS="main" CI_APORTS_ABUILD_PACKAGES="all/all" ci/buildrepos.sh aarch64

buildmain-mips64:
  stage: buildmain
  extends: .buildmain
  needs: [bootstrap-mips64]
  script:
    - REPOS="main" CI_APORTS_ABUILD_PACKAGES="all/all" ci/buildrepos.sh mips64

buildmain-ppc:
  stage: buildmain
  extends: .buildmain
  needs: [bootstrap-ppc]
  script:
    - REPOS="main" CI_APORTS_ABUILD_PACKAGES="all/all" ci/buildrepos.sh ppc

buildmain-ppc64:
  stage: buildmain
  extends: .buildmain
  needs: [bootstrap-ppc64]
  script:
    - REPOS="main" CI_APORTS_ABUILD_PACKAGES="all/all" ci/buildrepos.sh ppc64

.gen-toolchain:
  stage: gen-toolchain
  rules:
    - when: manual
    - if: "$CI_COMMIT_TAG"

# compile cross toolchain after main pkg are built
trigger-devkit-crosstool:
  extends: .gen-toolchain
  needs: [buildmain-aarch64, buildmain-mips64, buildmain-ppc, buildmain-ppc64]
  variables:
    TRIGGER_STAGE: crosstool
  trigger:
    project: rebornlinux/devkit
    branch: main
    strategy: depend

extend-toolchain:
  extends:
    - .gen-toolchain
    - .crosstoolimg
  needs: [trigger-devkit-crosstool]
  script:
    - ci/mktoolchain.sh
  timeout: 6 hours
  tags:
    - docker-aports

trigger-devkit-toolchain:
  extends: .gen-toolchain
  needs: [extend-toolchain]
  variables:
    TRIGGER_STAGE: toolchain
  trigger:
    project: rebornlinux/devkit
    branch: main
    strategy: depend

.buildrepos:
  extends:
    - .toolchainimg
    - .artifacts
  stage: buildrepos
  rules:
    - when: manual
    - if: "$CI_COMMIT_TAG"
  timeout: 480 hours
  tags:
    - docker-aports

buildrepos-x86_64:
  extends: .buildrepos
  needs: []
  script:
    - ci/buildrepos.sh x86_64

buildrepos-aarch64:
  extends: .buildrepos
  needs: []
  script:
    - ci/buildrepos.sh aarch64

buildrepos-mips64:
  extends: .buildrepos
  needs: []
  script:
    - ci/buildrepos.sh mips64

buildrepos-ppc:
  extends: .buildrepos
  needs: []
  script:
    - ci/buildrepos.sh ppc

buildrepos-ppc64:
  extends: .buildrepos
  needs: []
  script:
    - ci/buildrepos.sh ppc64

.mk-release:
  stage: mk-release
  when: manual
  rules:
    - if: "$CI_COMMIT_TAG"

minirootfs:
  extends:
    - .mk-release
    - .basedevimg
  needs: []
  script:
    - ci/minirootfs.sh "x86_64 aarch64 mips64 ppc ppc64"
  tags:
    - docker-aports

customize-artifacts:
  extends:
    - .mk-release
    - .basedevimg
  needs: []
  script:
    - ci/customize_artifacts.sh
  tags:
    - docker-aports

# update APKINDEX.tar.gz: an index to hold all existed apk current url mirror (use basedevimg, manual trigger)
updateindex:
  when: manual
  extends:
    - .basedevimg
  needs: []
  stage: updateindex
  script:
    - ci/update_artifact_index.sh
  timeout: 20 hours
  tags:
    - docker-aports
```

2 rebornlinux/nicon/.gitlab-ci.yaml:

```text
# default config for each stage: images, before script actions
default:
  image:
    name: rebornlinux-docker-local.artifactory-blr1.int.net.xxxxx.com/rebornlinux/toolchain:latest
    entrypoint: [""]
  before_script:
    - |-
      cat << EOF > ~/.gitconfig
      [url "https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlabe1.ext.net.xxxxx.com"]
          insteadOf = https://gitlabe1.ext.net.xxxxx.com
      [safe]
          directory = ${CI_PROJECT_DIR}
      EOF
    - sudo apk add go

stages:
  - check
  - build
  - test
  - apk
  - deploy
  - aports

check_format:
  stage: check
  script:
    - make format

lint_code:
  stage: check
  script:
    - echo "linting"

vet_code:
  stage: check
  script:
    - make vet

# compile the code & save the bin into artifact
make_build:
  stage: build
  script:
    - make build
  artifacts:
    paths:
      - bin

unit_test:
  stage: test
  script:
    - echo "testing"

# compile the code as apk, save the apk to artifacts
make_apk:
  stage: apk
  script:
    - make apk
  artifacts:
    paths:
      - abuild/nicon
  only:
    - main

# use gitlab pages to host apk repo
pages:
  stage: deploy
  script:
    - mv abuild/nicon public
  artifacts:
    paths:
      - public
  only:
    - main

# trigger aports repo action from nicon ci/cd pipeline
trigger_aports:
  stage: aports
  # variables to pass to next triggerred job
  variables:
    AB_PKG_NAME: nicon
    AB_PKG_VER: $CI_COMMIT_TAG
  # trigger aports project under rebornlinux group's ci/cd when nicon project tag modified
  trigger:
    project: rebornlinux/aports
    branch: reborn
    strategy: depend
  only:
    - tags
```

3 nicon/makefile as automation utilities:

```text
BIN_NAME := nicon
VERSION ?= $(shell git describe --tags --abbrev=0 | sed 's/^v//' 2>/dev/null)
GIT_COMMIT ?= $(shell git rev-parse HEAD)
GIT_DIRTY ?= $(shell test -n "`git status --porcelain`" && echo "+CHANGES" || true)
BUILD_DATE := $(shell date '+%Y-%m-%d-%H:%M:%S')

LDFLAGS := -X github.com/tces1/NiCon/pkg/version.Version=${VERSION}
LDFLAGS := ${LDFLAGS} -X github.com/tces1/NiCon/pkg/version.GitCommit=${GIT_COMMIT}${GIT_DIRTY}
LDFLAGS := ${LDFLAGS} -X github.com/tces1/NiCon/pkg/version.BuildDate=${BUILD_DATE}

default: build

format: ## Check coding style
	@DIFF=$$(gofmt -d .); echo -n "$$DIFF"; test -z "$$DIFF"

lint: ## Lint the files
	@golint -set_exit_status ./...

vet: ## Examine and report suspicious constructs
	@go vet ./...

apk: ## Generate .apk package
	@cp abuild/APKBUILD abuild/APKBUILD.tmp                       # stash APKBUILD
	@sed -i "s/^pkgver=.*/pkgver=${VERSION}/" abuild/APKBUILD     # update pkgver
	@sed -i "s/^pkgrel=.*/pkgrel=999/" abuild/APKBUILD            # update pkgrel
	cd abuild && REPODEST=`pwd` abuild -r                         # compile to apk
	@mv abuild/APKBUILD.tmp abuild/APKBUILD                       # restore APKBUILD

build: ## Compile the project
	@echo "building ${BIN_NAME} ${VERSION}"
	@echo "GOPATH=${GOPATH}"
	rm -rf bin/*; CGO_ENABLED=0 go build -ldflags "${LDFLAGS}" -o bin/${BIN_NAME}

clean: ## Clean the directory tree
	@test ! -e bin/${BIN_NAME} || rm bin/${BIN_NAME}
	@rm -rf abuild/nicon

test: ## Run go test
	go test ./...

help: ## Display this help screen
	@grep -h -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; \
    {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

# upload bin to artifactory for release
push:
	@echo "upload version: https://artifactory-xxxxx.com/artifactory/download/apps/${BIN_NAME}"
	@echo "upload version: https://artifactory-xxxxx.com/artifactory/download/apps/${BIN_NAME}-${GIT_COMMIT}"
	cd ./bin; cp ${BIN_NAME} ${BIN_NAME}-${GIT_COMMIT}; jf rt u "*" hostfw-local/download/apps/

# upload bin to artifactory for debug purpose: assign the nicon url env inside hostfw to test on it
push-debug:
	@echo "upload debug version: https://artifactory-xxxxx.com/artifactory/download/template/${BIN_NAME}-debug-${GIT_COMMIT}"
	cd ./bin; mv ${BIN_NAME} ${BIN_NAME}-debug-${GIT_COMMIT}; jf rt u "*" hostfw-local/download/template/
```
