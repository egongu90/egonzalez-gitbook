---
description: Simplifying your DevOps life
---

# Ansible INI file module

If you don't read docs, one day you'll realize that your an idiot as i am\|was.

A few days back, I've realized that i was using wrong all Ansible modules power since i started with it. What happened?

Most of the time i use Ansible is related to OpenStack configuration jobs. Almost, all OpenStack projects use INI formatted files for their configuration files. When i started using Ansible, I searched on Google how to configure any kind of file with Ansible modules. Almost all blogs/forums that i saw, talked about lineinfile module. So i used these guidelines on my next few months, now i realize that i was using in the wrong way Ansible modules.

Ansible have a module called ini\_file, you change values inside INI formatted files in a easy way , you don't need to use complicated regular expressions to change a value in a file.

Here you have ini\_file module usage docs: [http://docs.ansible.com/ansible/ini\_file\_module.html](http://docs.ansible.com/ansible/ini_file_module.html)

We are going to change Neutron user password in his dump config file, so we create a simple task on which we can see how ini\_file module can be used.

```text
- hosts: localhost
  tasks:
  - name: Change neutron user password
    ini_file:
      dest: ~/neutron.conf
      section: keystone_authtoken
      option: password
      value: 12345
```

Once the task has been applied, we can see how the values are applied in a proper ini style.

```text
cat neutron.conf
[keystone_authtoken]
password = 12345
```

How many times you need to make a change in an INI formatted configuration file with Ansible and used lineinfile module? If the answer is many times, it's OK, you are a dump like me.

Regards, Eduardo Gonzalez

