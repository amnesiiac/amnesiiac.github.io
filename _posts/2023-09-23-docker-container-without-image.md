---
layout: post
title: "run a container without a image (docker & runc)"
author: "melon"
date: 2023-09-23 17:28
categories: "2023"
tags:
  - container
  - ongoing
---

### # introduction to low-level container api: runc
the docker relies on a bunch of softwares, a typical reliance map is as follows:

```txt

                           <─── runc ───>
                                     <─── cri-o ───>
                                                <─── containerd ───>
                                                                <─── docker ───>
          low-level <----------------------------------------------------------------> high-level

```

take a closer look at the concise workflow of runc:

```txt
                        (1)
            prepare files for container-1                                                  container c309add4f1 created
┌─────────────────────────────────────────────────────┐                                  ┌──────────────────────────────┐
│  rootfs              config.json                    │                                  │            namespaces        │
│ ┌──────────────┐    ┌─────────────────────────────┐ │                                  │     ...┌───────────────────┐ │
│ │ /            │    │{                            │ │                                  │    net┌───────────────────┐│ │
│ │ ├── bin      │    │  "ociVersion": "1.0.2",     │ │                                  │   pid┌───────────────────┐││ │
│ │     ├── cat  │    │  "process": {               │ │                                  │  mnt┌───────────────────┐│││ │
│ │     ├── sh   │    │    "args": ["sleep"],       │ │                                  │     │                   ││││ │
│ │ ...          │    │  },                         │ │                                  │     │  ┌─────────────┐  ││││ │
│ │ ├── usr      │    │  "root": {"path": "rootfs"},│ │               (2)                │     │  │  runc init  │  ││││ │
│ │ └── var      │    │  "linux": {                 │ │           runc create            │     │  │ "steb" proc │  ││││ │
│ └──────────────┘    │    "namespaces": {          │ │ --bundle container-1 c309add4f1  │     │  └─────────────┘  ││││ │
│                     │      {"type": "pid"},       │ │ ───────────────────────────────> │caps │  ┌─────────────┐  ││││ │
│                     │      {"type": "network"},   │ │                                  │     │  │ /           │  ││││ │
│                     │      {"type": "ipc"},       │ │                                  │     │  │ ├── bin     │  ││││ │
│                     │      {"type": "uts"},       │ │                                  │     │  │     ├── cat │  ││││ │
│                     │      {"type": "mount"},     │ │                                  │     │  │     ├── sh  │  ││││ │
│                     │      {"type": "user"},      │ │                                  │     │  │ ...         │  ││││ │
│                     │  ...                        │ │                                  │     │  │ ├── usr     │  │││┘ │
│                     └─────────────────────────────┘ │                                  │     │  │ └── var     │  ││┘  │
└─────────────────────────────────────────────────────┘                                  │     │  └─────────────┘  │┘   │
                                                                                         │     └───────────────────┘    │
                                                                                         │            cgroups           │ 
                                                                                         └───────────────┬──────────────┘
                                                                                                         │ 
                                                                               (3) runc start c309add4f1 │
                                                                                                         │
                                                                                           container c309add4f1 running 
                                                                                         ┌───────────────+──────────────┐
                                                                                         │           namespaces         │
                                                                                         │     ...┌───────────────────┐ │
                                                                                         │    net┌───────────────────┐│ │
                                                                                         │   pid┌───────────────────┐││ │
                                                                                         │  mnt┌───────────────────┐│││ │
                                                                                         │     │                   ││││ │
                                                                                         │     │  ┌─────────────┐  ││││ │
                                                                                         │     │  │    sleep    │  ││││ │
                          (5)                     c309add4f1               (4)           │     │  │ "spec" proc │  ││││ │
                 runc delete c309add4f1     ┌───────────────────┐  runc kill c309add4f1  │     │  └─────────────┘  ││││ │
             <───────────────────────────── │ container stopped │ <───────────────────── │caps │  ┌─────────────┐  ││││ │
               (release residual resources) └───────────────────┘  (kill main process)   │     │  │ /           │  ││││ │
                                            no process & namespaces                      │     │  │ ├── bin     │  ││││ │
                                                 at this time                            │     │  │     ├── cat │  ││││ │
                                                                                         │     │  │     ├── sh  │  ││││ │
                                                                                         │     │  │ ...         │  ││││ │
                                                                                         │     │  │ ├── usr     │  │││┘ │
                                                                                         │     │  │ └── var     │  ││┘  │
                                                                                         │     │  └─────────────┘  │┘   │
                                                                                         │     └───────────────────┘    │
                                                                                         │            cgroups           │
                                                                                         └──────────────────────────────┘
```

based on the above diagram, the runc runtime has nothing to do with "image", but only relies on a filesystem directory
and at least executable file inside and a config.json.  

docker or any other container engine like containerd, or podman take an image and converts the image to an oci bundle
before invoking the lower-level container runtime like runc.
that is to say, docker use image to generate the bundle for lower runc.

<hr>

### # basic steps to create container with runc

<hr>

### # why do we need container images





ref: https://iximiuz.com/en/posts/you-dont-need-an-image-to-run-a-container/
