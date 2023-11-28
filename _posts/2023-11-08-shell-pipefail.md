---
layout: post
title: "pipefail (shell)"
author: "melon"
date: 2023-11-08 21:58
categories: "2023"
tags:
  - shell
---

### # background
gitlab-runner job fails when using yes and pipe for automation of interactive cmd

given __rebornlinux/devkit/.gitlab-ci.yml__ for gitlab ci/cd as:
```text
stages:
  - release

release:
  stage: release
  rules:                  # run on changes of .gitlab-ci.yml
    - changes:
      - .gitlab-ci.yml
  before_script:
    - mkdir -p $HOME/.docker
    - echo $DOCKER_AUTH_CONFIG > $HOME/.docker/config.json
  script:
    - yes | make base
    - yes | make toolchain
    - yes | make devkit
  tags:
    - shell-generic
```

given __rebornlinux/devkit/Makefile__ for build & push docker image to self-maintained registry:
```text
proxy := http://10.158.100.9:8080
repo_tag := $(shell git describe --tags --abbrev=0)
site := artifactory-blr1.int.net.nokia.com
artifactory := rebornlinux-docker-local.$(site)

define docker_push #from,to
	@echo Are you sure to push $(1) to $(2) ?                  # confirm for push to jfrog artifactory
	@read -p "if not press Ctrl+C -- any key to continue" tmp  # read input from stdin
	docker tag $(1) $(2)
	docker push $(2)
endef

default: devkit

base: docker_build.base
toolchain: docker_build.toolchain
devkit: docker_build.devkit

docker_build.%:
	$(eval DOCKERFILE := $(@:docker_build.%=Dockerfile.%))
	$(eval IMG := $(@:docker_build.%=rebornlinux/%))
	docker build -f $(DOCKERFILE) -t $(IMG):$(repo_tag) \
		--build-arg HTTP_PROXY=$(proxy) --build-arg HTTPS_PROXY=$(proxy) \
		--build-arg http_proxy=$(proxy) --build-arg https_proxy=$(proxy) \
		.
	docker tag $(IMG):$(repo_tag) $(IMG):latest
	$(call docker_push,$(IMG):$(repo_tag),$(artifactory)/$(IMG):$(repo_tag))
	$(call docker_push,$(IMG):latest,$(artifactory)/$(IMG):latest)
```

<hr>

### # actual behaviour
inside gitlab runner jobs result page, when exec 'yes | make base', returned:
```text
ERROR: Job failed: exit status 1
```
which means the yes pipe command return non-zero value.

<hr>

### # root cause analysis
for cmd like 'yes | ${interactive_cmd}' will fail when yes tries to write to stdout, 
but the interactive proc's is not reading from its stdin (connected to yes's stdout), 
this will cause an EPIPE signal.

by default, the gitlab runner enable pipefail check feature by 'set -o pipefail' for scripts running on it, 
which will cause the ci/cd job fail.

<hr>

### # solution
modify the __rebornlinux/devkit/.gitlab-ci.yml__, explcitly set pipefail=false:
```text
stages:
  - release

release:
  stage: release
  rules:                                                    # run on changes of .gitlab-ci.yml
    - changes:
      - .gitlab-ci.yml
  before_script:
    - set +o pipefail                                       # explicitly turn off pipefail
    - mkdir -p $HOME/.docker
    - echo $DOCKER_AUTH_CONFIG > $HOME/.docker/config.json  # docker registry login
  script:
    - yes | make base
    - yes | make toolchain
    - yes | make devkit
  tags:
    - shell-generic
```

<hr>

### # shell script for pipefail illustration
```text
#!/bin/bash

pipefail_analysis() {
    set +o pipefail
    yes | true
    echo "without setting pipefail,   yes | true          returns: $?"
    set -o pipefail
    yes | true
    echo "after setting pipefail,     yes | true          returns: $?"
}

another_method_to_use_yes_with_pipefail() {
    set +o pipefail
    { yes || :; } | true
    echo "without setting pipefail, { yes || :;} | true   returns: $?"
    set -o pipefail
    { yes || :; } | true
    echo "after setting pipefail,   { yes || :;} | true   returns: $?"
}

pipefail_analysis
another_method_to_use_yes_with_pipefail

# without setting pipefail,   yes | true          returns: 0
# after setting pipefail,     yes | true          returns: 141
# without setting pipefail, { yes || :;} | true   returns: 0
# after setting pipefail,   { yes || :;} | true   returns: 0

# return code 141 means pipefail, 0 for success.
```
