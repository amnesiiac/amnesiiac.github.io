---
layout: post
title: "how to setup proxy inside gitlab-runner (gitlab-runner)"
author: "melon"
date: 2023-12-08 23:03
categories: "2023"
tags:
  - gitlab
---

### # steps to setup the proxy env var for certain gitlab-runner
check the runner status & get the config file path
```text
$ systemctl status gitlab-runner
● gitlab-runner.service - GitLab Runner
   Loaded: loaded (/etc/systemd/system/gitlab-runner.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2023-11-27 07:02:16 UTC; 1 weeks 3 days ago
  Main PID: 18645 (gitlab-runner)
   Tasks: 38 (limit: 4915)
  CGroup: /system.slice/gitlab-runner.service
     └─18645 /usr/bin/gitlab-runner run --working-directory /home/gitlab-runner --config /etc/gitlab-runner/config.toml
```

modify gitlab-runner config for adding proxy settings (per runner)
```text
$ cat /etc/gitlab-runner/config.toml | tail -n
[[runners]]
  name = "cloud-server-1"
  url = "https://gitlabe1.ext.net.xxxxx.com/"
  token = "q2SyArbaxwrh_joKBVp7"
  executor = "docker"

  # setup git clone proxy
  pre_get_sources_script = "git config --global http.proxy $HTTP_PROXY; git config --global https.proxy $HTTPS_PROXY"
  environment = [
    "https_proxy=http://10.144.1.10:8080",
    "http_proxy=http://10.144.1.10:8080",
    "HTTPS_PROXY=http://10.144.1.10:8080",
    "HTTP_PROXY=http://10.144.1.10:8080"
  ]

  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "alpine:latest"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false

    # vol mount for docker runner
    volumes = ["/repo/apk/packages:/home/reborn/packages:rw", "/cache"]
    shm_size = 0
```
more details about toml config lang see: https://github.com/toml-lang/toml.

validate the configured proxy settings inside rebornlinux/aport/.gitlab-ci.yml:
```text
.init:
  when: manual
  stage: bootstrap

init-x86_64:
  extends: .init                                 # reusable
  script:
    - echo "$CI_BUILDS_DIR"                      # /builds
    - echo "$CI_PROJECT_DIR"                     # default ci job clone dir: /builds/rebornlinux/aports
    - export | grep proxy                        # test for proxy settings
    - echo 'bootstrap.sh ${options_for_x86_64}'  # fake routine
  tags:
    - aport-specific                             # specify runner
```

manual trigger the init-x86_64 job, check the result in the gitlab runner logs:
```text
Running with gitlab-runner 13.4.0 (4e1f20da) on cloud-server-1 q2SyArba
Preparing the "docker" executor
Using Docker executor with image rebornlinux-docker-local.artifactory-xxxxx.com/rebornlinux/toolchain:v1.2.4 ...
Pulling docker image rebornlinux-docker-local.artifactory-xxxxx.com/rebornlinux/toolchain:v1.2.4 ...
Preparing environment
Running on runner-q2syarba-project-64564-concurrent-0 via cloud-server-1...
Getting source from Git repository
Fetching changes with git depth set to 500...

Initialized empty Git repository in /builds/rebornlinux/aports/.git/
Created fresh repository.

Checking out 55fa1d63 as 5-add-bootstrap-stage-add-trigger-when-maintainer-push-f-manually...

$ git config --global url."https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlabe1.x.com".insteadOf "https://gitlabe1.x.com"

$ echo "$CI_BUILDS_DIR"
/builds

$ echo "$CI_PROJECT_DIR"
/builds/rebornlinux/aports

$ export | grep proxy
declare -x CI_DEPENDENCY_PROXY_DIRECT_GROUP_IMAGE_PREFIX="gitlabe1.ext.net.xxxxx.com:443/rebornlinux/..."
declare -x CI_DEPENDENCY_PROXY_GROUP_IMAGE_PREFIX="gitlabe1.ext.net.xxxxx.com:443/rebornlinux/..."
declare -x http_proxy="http://10.144.1.10:8080"
declare -x https_proxy="http://10.144.1.10:8080"
```
that is, the proxy is settled successfully.

<hr>

### # reference
https://docs.gitlab.com/runner/configuration/proxy.html#adding-the-proxy-to-the-docker-containers
