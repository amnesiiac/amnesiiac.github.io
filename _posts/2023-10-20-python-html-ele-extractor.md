---
layout: post
title: "html elements extractor (python)"
author: "melon"
date: 2023-10-20 21:07
categories: "2023"
tags:
  - python
  - html
---

given the dir tree as below, serve the dir using nginx to establish a file server
on ${server_ip}:8888.
the nginx file server setup can refer blog post: 
```text
.
├── hello
├── world
└── host_builds
    ├── zbuild1.json
    └── vobs
        ├── dsl
        │   └── sw
        │       └── y
        │           └── build
        │               ├── xxxxplano
        │               │   └── generated
        │               │       └── xxxx-b
        │               │           └── default
        │               │               ├── device-extension-ls-fx-xxxx-x-24.6-202.zip
        │               │               └── device_extension_mapping_xxxx-x_xxxx-x.txt
        │               └── xxxxx_app
        │                   └── generated
        │                       └── xxxx-b
        │                           ├── default
        │                           │   ├── migration-xmls-xxxx-x-default.tar
        │                           │   └── yang-attributes-xx-xx-xxxx-x.json
        │                           ├── partial_xxxxxx2406.202.tar
        │                           └── product_name_xxxx-x.txt
        └── xxxx
            └── build
                ├── docs
                │   └── yang
                │       └── oam
                │           └── OAM_files_for_atc.tgz
                └── host
                    └── xxxxx
                        ├── bld_changed_items.json
                        ├── xxxx-x_board.json
                        └── images
                            └── xxxx-x
                                ├── device-extension-xxxx-24.6.202.zip
                                ├── host-docker-rootfs.tar.gz
                                ├── host_image.tar
                                └── host-target-persistent.tar.gz
24 directories, 16 files
```

the following code aim to search for certain url identified by regex string for
a given root url.

<hr>

### # bfs url search sample
search bfs recursively into the url given, to find the file url matching the regex:
start with zbuild, end with json, with maximum 3 level of depth to search into.
```text
import re
import requests
try:
    from html.parser import HTMLParser
    class ElementExtractor(HTMLParser):
        def __init__(self):
            super().__init__()
            self.links = []
        def handle_starttag(self, tag, attrs):
            if tag == 'a':
                for attr in attrs:
                    if attr[0] == 'href':
                        self.links.append(attr[1])
        @staticmethod
        def extract_links_from_html(html_content):
            parser = ElementExtractor()
            parser.feed(html_content)
            return parser.links
except ModuleNotFoundError:
    print("html module not found!")
    return ""

def bfs_url_search(site: str, regexstr: str, level: int=3) -> str:
    r = requests.get(site)
    s = ElementExtractor.extract_links_from_html(r.text)
    sites = []
    curlevel = 0
    sites.append((s, site, curlevel))

    # bfs to search for target file pattern recursively
    while len(sites) != 0:
        req_text = sites[0][0]
        site = sites[0][1]
        curlevel = sites[0][2]
        del sites[0]
        for href in req_text:
            url = site + href
            print(url)
            if re.search(regexstr, href):
                print(f"found file matches given regex: {url}")
                return url
            if href.endswith("/") and not href.startswith('.'):
                r = requests.get(url)
                s = ElementExtractor.extract_links_from_html(r.text)
                sites.append((s, url, curlevel+1))
        if curlevel == level:
            print("nothing found under url match the regex given")
            return ""
    return ""

if __name__ == '__main__':  # bfs search function unit test
    depth = 3
    url = 'http://localhost:8888/host_builds'
    url = url if url.endswith('/') else url + '/'
    ret = bfs_url_search(url, r"^zbuild.*\.json$", depth)
    print(ret)
```

output of bfs search path:
```text
http://10.182.101.165:8888/host_builds/../
http://10.182.101.165:8888/host_builds/vobs/
http://10.182.101.165:8888/host_builds/zbuild1.json
```

<hr>

### # dfs url search sample
search dfs recursively into the url given, to find the file url matching the regex:
start with zbuild, end with json.
```text
try:
    import re
    import requests
    from bs4 import BeautifulSoup
except ModuleNotFoundError:
    exit("module not found!")

urls=[]

def dfs_url_search(site: str) -> str:
    r = requests.get(site)
    s = BeautifulSoup(r.text, "html.parser")

    for i in s.find_all("a"):
        href = i.attrs['href']
        comb = site + href
        print(comb)
        if href.startswith("zbuild") and href.endswith(".json"):  # found
            return comb
        if href.endswith("/") and not href.startswith('.'):       # continue search
            if comb not in urls:
                urls.append(comb)
                dfs_url_parse(comb)                               # recursive
    return ""

if __name__ == '__main__':  # dfs search function unit test
    url = 'http://localhost:8888/host_builds'
    url = url if url.endswith('/') else url + '/'
    ret = dfs_url_search(url)
    print(ret)
```

output of dfs search path:
```text
http://10.182.101.165:8888/host_builds/../
http://10.182.101.165:8888/host_builds/vobs/
http://10.182.101.165:8888/host_builds/vobs/../
...
http://10.182.101.165:8888/host_builds/vobs/xxxx/
http://10.182.101.165:8888/host_builds/vobs/xxxx/../
http://10.182.101.165:8888/host_builds/vobs/xxxx/build/
...
http://10.182.101.165:8888/host_builds/vobs/xxxx/build/host/xxxxx/images/xxxx-x/../
http://10.182.101.165:8888/host_builds/vobs/xxxx/build/host/xxxxx/images/xxxx-x/host-docker-rootfs.tar.gz
http://10.182.101.165:8888/host_builds/vobs/xxxx/build/host/xxxxx/images/xxxx-x/host-target-persistent.tar.gz
http://10.182.101.165:8888/host_builds/vobs/xxxx/build/host/xxxxx/images/xxxx-x/host_image.tar
http://10.182.101.165:8888/host_builds/vobs/xxxx/build/host/xxxxx/bld_changed_items.json
http://10.182.101.165:8888/host_builds/vobs/xxxx/build/host/xxxxx/xxxx-x_board.json
http://10.182.101.165:8888/host_builds/zbuild1.json
```

<hr>

### # conclusion
judging from output of bfs & dfs search path, the bfs has better performance than
the dfs does.
