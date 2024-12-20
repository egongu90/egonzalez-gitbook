# To conditional or to skip, that's the Ansible question

Have you ever think about if an Ansible task should be skipped with a conditional or without (hidden skip)?. | Well, this post will analyse both methods.

Let's use a example to create the same result and analyse the both methods:

In OpenStack Kolla we found that sometimes operators need to customise policy.json files. That\\'s fine, but the problem is that policy.json files are installed with the services by default and we don\\'t need/want to maintain policy.json files in our repository because for sure will cause bugs from outdated policy files in the future.

What\\'s the proposed solution to this? Allow operators use their own policy files only when their files exists in a custom configuration folder. If custom files are not present, default policy.json files are already present as part of the software installation. ( Actually this change is under review )

### To Conditional method

Code snippet:

```
    - name: Check if file exists
      stat:
        path: "/tmp/custom_file.json"
      register: check_custom_file_exist

    - name: Copy custom policy when exist
      template:
        src: "/tmp/custom_file.json"
        dest: "/tmp/destination_file.json"
      when: "{{ check_custom_file_exist.stat.exists }}"
```

The first task checks if the file is present and register the stat result.&#x20;

The second task, copy the file only when the registered result of the previous task is True. (exists == True)

Outputs the following when the file is not present:

```
PLAY [localhost] ***************************************************************

TASK [Check if file exists] ****************************************************
ok: [localhost]

TASK [Copy custom policy when exist] *******************************************
skipping: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0  
```

We can see the copy file task is skipped with a skipping message.

### To Skip method

Code snippet:

```
    - name: Copy custom policy when exist
      template:
        src: "{{ item }}"
        dest: "/tmp/destination_file.json"
      with_first_found:
      - files:
        - custom_file.json
        skip: True
```

This playbook contains a single task, this task will use the first found file in a list of files. If no file is present will skip the task.

Output from this execution:

```
PLAY [localhost] ***************************************************************

TASK [Copy custom policy when exist] *******************************************

PLAY RECAP *********************************************************************
localhost                  : ok=0    changed=0    unreachable=0    failed=0 
```

We can see that no task is executed when custom files are not present, no output from the task (hidden skip).

### Analysis and own opinion

Both methods do the same, both copy custom files when are present and both skip copy task when are not present. What are the differences between both methods?

To\_skip method is simpler to read and unified in a single task, to\_conditional is created within two tasks. To\_conditional method takes longer to be executed as it has to check the existence of a file and then evaluate a conditional.

You may think that to\_skip method is better than to\_conditional method, that\\'s right in terms of code syntax and execution times. But... As both, operator and infrastructure developer, I always use to\_conditional method because when I\\'m deploying something new, I want to know what is executed and what not. In to\_skip method you don\\'t know because there is no output provided from the task (not really true) but in to\_conditional method it clearly says Skipping.

Execution times are not a problem in most use cases, as is not commonly used this kind of tasks in CM systems, only a few tasks will need this type of logic.

Regards, Eduardo Gonzalez
