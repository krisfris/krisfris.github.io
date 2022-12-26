---
layout: post
title:  Securely running untrusted Python code
date:   2020-08-21
---

In this post, I present a layered approach to minimizing the inherent security risk
when running untrusted Python code.

### Step 1: Securing Python

Restrict the actions the code can perform. The method used differs depending on whether you use
CPython or PyPy.

#### CPython

Use [RestrictedPython](https://restrictedpython.readthedocs.io/en/latest/) to define
a restricted subset of Python.

Use [audit hooks](https://www.python.org/dev/peps/pep-0578/) (available since Python 3.8)
to completely prevent certain actions.

{% highlight python %}
import sys

def audit(event, args):
    if event == 'compile':
        sys.exit('nice try!')

sys.addaudithook(audit)

eval('5')
{% endhighlight %}

#### PyPy

Use [sandboxing](https://doc.pypy.org/en/latest/sandbox.html).
It allows you to run arbitrary Python code in a special environment that serializes all input/output so you can check it and decide which commands are allowed before actually running them.

This is the most secure way of restricting Python.

### Step 2: Securing the host OS

To protect your host OS, you have two options.

* **Virtualization** such as KVM or VirtualBox (more secure)
* **Containerization** such as [LXD](https://linuxcontainers.org/lxd/introduction/)
or [Docker](https://www.docker.com/) (much lighter)

In the case of containerization with Docker you may need to add AppArmor or SELinux policies for extra security. LXD already comes with AppArmor policies by default.

Make sure you run the code as a user with as little privileges as possible.

Rebuild the virtual machine/container for each user.

Whichever solution you use, don't forget to limit resource usage (RAM, CPU, storage, network).
Use [cgroups](http://man7.org/linux/man-pages/man7/cgroups.7.html) if your chosen virtualization/containerization solution does not support these kinds of limits.

### Step 3: Timeouts

Use timeouts on all layers to ensure that neither code nor container run longer than
they are supposed to.

### Step 4: Communication

If distinct pieces of untrusted code need to communicate with each other or with your host app,
use a language-agnostic API such as REST or gRPC.

### Conclusion

These measures help running untrusted code securely.