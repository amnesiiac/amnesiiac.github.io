---
layout: post
title: "yaml json parser (python, data-driven interface)"
author: "melon"
date: 2023-10-07 21:27
categories: "2023"
tags:
  - python
---

this article provide a toy parser code for yaml & json files, including the common apis like:
load yaml/json from file, dump data structure as yaml/json.
what's more, provides a unified api interface using the abstract class parser.

<hr>

### # parser code in action
parser.py (utility): an unified parser with yaml, json format supported.

```text
import os
import yaml
import json

class ParserYaml:
    def __init__(self, f):
        self.f = f
        self.d = {}

    def load(self, obj=None):
        if obj is None:
            if os.path.isfile(self.f):
                with open(self.f) as f:
                    self.d = yaml.load(f, Loader=yaml.FullLoader)
        else:
            self.d = yaml.load(obj, Loader=yaml.FullLoader)
        return self.d

    def dump(self, obj=None, out_f=None, default_flow_style=False):
        if not out_f:
            out_f = self.f
        if obj is None:
            obj = self.d
        if os.path.isdir(os.path.dirname(out_f)):
            try:
                with open(out_f, "w") as f:
                    f.write(yaml.dump(obj, default_flow_style=default_flow_style, sort_keys=False))
                    return obj
            except OSError as err:  # specially for OSError: quota exceed
                exit(f"{err} during yaml dump")
        else:
            print(f"parser dump out file {out_f} not existed")
            return None

class ParserJson:
    def __init__(self, f):
        self.f = f
        self.d = {}

    def load(self, obj=None):
        if os.path.isfile(self.f):
            with open(self.f) as f:
                self.d = json.loads(f.read())
        return self.d

    def dump(self, obj=None, out_f=None, default_flow_style=False):
        if not out_f:
            out_f = self.f
        if obj is None:
            obj = self.d
        try:
            with open(out_f, "w") as f:
                f.write(json.dumps(obj))
        except OSError as err:  # specially for OSError: quota exceed
            exit(f"{err} during json dump")

class JsonObj:
    def pretty(self, intput):
        ret = json.dumps(intput, indent=4, separators=(",", ": "))
        return ret

jsonObj = JsonObj()

class YamlObj:
    @staticmethod
    def get(path, base):
        if not path:
            return base

        subs = path.split(".")
        for i, s in enumerate(subs):
            if not s:
                continue
            if s.startswith("[") and i == 0 and s[1:-1].isdigit():
                index = int(s[1:-1])
                if not isinstance(base, list) or index >= len(base):
                    return None
                base = base[index]
            elif s.endswith("]") and not s.startswith("["):
                fields = s.split("[")
                name = fields[0]
                if fields[1][:-1].isdigit():
                    index = int(fields[1][:-1])
                    if not isinstance(base, dict):
                        return None
                    base = base.get(name)
                    if not isinstance(base, list) or index > len(base):
                        return None
                    base = base[index]
                else:
                    return None
            elif isinstance(base, dict):
                base = base.get(s, None)
            else:
                return None

            if base is None:
                break
        return base

    @staticmethod
    def set(path, value, base):
        # TODO: could be recursive, otherwise only support setting values whose parent is existing
        if not path:
            return base

        subs = path.split(".")
        if subs[-1].endswith("]"):
            fields = subs[-1].split("[")
            name = fields[0]
            if fields[1][:-1].isdigit():
                index = int(fields[1][:-1])
                parent = ".".join(subs[0:-1] + [name])
                key = int(index)
            else:
                return None
        elif len(subs) == 1:
            parent = None
            key = subs[-1]
        else:
            parent = ".".join(subs[0:-1])
            key = subs[-1]

        obj = YamlObj.get(parent, base)
        if(isinstance(obj, list) and isinstance(key, int) and key < len(obj)) or isinstance(obj, dict):
            obj[key] = value
            return obj
        else:
            print(f"yaml obj set an invalid path: {path}")
            return None

class Parser:
    def __init__(self, f):
        self.root = None
        ext = os.path.splitext(f)[-1]
        if ext in [".yaml", ".yml"]:
            self.p = ParserYaml(f)
        elif ext == ".json":
            self.p = ParserJson(f)
        else:
            raise ValueError(
                "\nThe configuration file format only supports '.yaml', '.yml' or '.json'"
                "\nplease check your file: {}".format(f)
            )

    def load(self, obj=None):
        self.root = self.p.load(obj)
        return self.root

    def get_root(self):
        return self.root

    def get(self, path, base=None):
        if self.root is None:
            self.load()
        if not base:
            base = self.root
        return YamlObj.get(path, base)

    def set(self, path, value, base=None):
        if self.root is None:
            self.load()
        if not base:
            base = self.root
        return YamlObj.set(path, value, base)

    def dump(self, obj=None, out_f=None, default_flow_style=False):
        return self.p.dump(obj, out_f, default_flow_style)

if __name__ == "__main__":
    p1 = Parser("./v.yaml")
    data = p1.load()
    obj = p1.get(" melons[1].origin")
    print(f"my hometown is {obj}, reading from yaml memories")
    print("parsing obj content from yaml file:")
    print()
    print("whole yaml file content is as:")
    print(data)

    print()

    p2 = Parser("./v.json")
    data = p2.load()
    obj = p2.get("melons[1].origin")
    print(f"my hometown is {obj}, reading from json memories")
    print()
    print("whole json file content is as:")
    print(data)
```

content of v.yaml file:

```text
melons:
  - origin: xinjiang
    block: 1
    slot: 1
  - origin: chifeng
    block: 1
    slot: 2
bananas:
  - origin: xinjiang
    block: 2
    slot: 5
  - origin: chifeng
    block: 3
    slot: 6
```

content of v.json file:

```text
{
  "melons": [
    {
      "origin": "xinjiang",
      "block": 1,
      "slot": 1
    },
    {
      "origin": "chifeng",
      "block": 1,
      "slot": 2
    }
  ],
  "bananas": [
    {
      "origin": "xinjiang",
      "block": 2,
      "slot": 5
    },
    {
      "origin": "chifeng",
      "block": 3,
      "slot": 6
    }
  ]
}
```
