---
title: A Template Python CLI Application
tags: python python3 template cli application linux
excerpt: "Here's a template python CLI application that, when executed directly, loads a YAML configuration (being specificed as an argument at execution via CLI) and defines this as `self.config`. that I use to as a quick starting point to build more specialised / niche tools with " 
---

## Overview

Here's a template python CLI application that, when executed directly, loads a YAML configuration (being specificed as an argument at execution via CLI) and defines this as `self.config`

Really, this is just a quick starting point for me to build more specialised / niche tools with, but I thought I'd share 

## Source Code

```python
#!/usr/local/bin/python3

import argparse, yaml, os


class template:
    def __init__(self) -> None:
        self.script_path: str = str(os.path.dirname(os.path.realpath(__file__))) + "/"
        self.config: dict = self.__load_configuration__(
            self.script_path + arguments.config
        )

    def __load_configuration__(self, config_path) -> dict:
        return yaml.load(open(config_path), Loader=yaml.FullLoader)


if __name__ == "__main__":
    metadata: dict = {
        "application_name": "template",
        "application_description": "template_desc",
    }

    argument_setup: argparse.ArgumentParser = argparse.ArgumentParser(
        prog=metadata["application_name"],
        description=metadata["application_description"],
    )
    argument_setup.add_argument(
        "-c", "--config", metavar="config.yaml path", help="Path to config.yaml"
    )
    arguments = argument_setup.parse_args()

    template()

```
