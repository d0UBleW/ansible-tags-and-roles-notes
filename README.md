# Ansible Tags and Roles Notes

- [Roles](#roles)
- [No tag](#no-tag)
- [Tag `dep`](#tag-dep)
- [Tag `dep_task`](#tag-dep_task)
- [Tag `test`](#tag-test)
- [Tag `test_task`](#tag-test_task)
- [Tag `test_inc`](#tag-test_inc)
- [Tag `test_inc_task`](#tag-test_inc_task)
- [Tag `test_inc`, `test_inc_task`](#tag-test_inc-test_inc_task)
- [Tag `test_imp`](#tag-test_imp)
- [Tag `test_imp_task`](#tag-test_imp_task)
- [Tag `test_incfalse`](#tag-test_incfalse)
- [Tag `test_incfalse_task`](#tag-test_incfalse_task)
- [Tag `test_block`](#tag-test_block)
- [Tag `test_block_task1`](#tag-test_block_task1)
- [References](#references)

### Roles

When a role is imported/included, the first thing that it will do is resolve dependencies defined under `meta/main.yml`. After depedencies are taken care of, it will proceed to `tasks/main.yml` but just before that, it will load variables defined under both `vars/` and `defaults/` directories. If a particular task involve local file, such as `ansible.builtin.copy`, `ansible.builtin.unarchive`, `ansible.builtin.template`, etc., the task will look into `files/` or `templates/` directory when we only specify the file name without the full path nor relative path.

```YAML
- name: Copy
  ansible.builtin.copy:
    src: my-file # the task will look under `files/` directory
    ...

- name: Copy
  ansible.builtin.copy:
    src: ./my-file # the task will resolve the path as specified
    ...
```

### No tag

```sh
test
❯ ansible-playbook main.yml
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Play] **************************************************************************************************************************************************************

TASK [dep : Debug] *******************************************************************************************************************************************************
ok: [localhost] => {
            "msg": "Dep roles"
}

TASK [dep : Debug const_var] *********************************************************************************************************************************************
ok: [localhost] => {
            "const_var": "This variable is not overridable"
}

TASK [dep : Debug dyn_var1] **********************************************************************************************************************************************
ok: [localhost] => {
            "dyn_var1": "Override dyn_var1"
}

TASK [dep : Debug dyn_var2] **********************************************************************************************************************************************
ok: [localhost] => {
            "dyn_var2": "This variable is overridable"
}

TASK [dep : Debug] *******************************************************************************************************************************************************
ok: [localhost] => {
            "msg": "Dep roles"
}

TASK [dep : Debug const_var] *********************************************************************************************************************************************
ok: [localhost] => {
            "const_var": "This variable is not overridable"
}

TASK [dep : Debug dyn_var1] **********************************************************************************************************************************************
ok: [localhost] => {
            "dyn_var1": "Override dyn_var1"
}

TASK [dep : Debug dyn_var2] **********************************************************************************************************************************************
ok: [localhost] => {
            "dyn_var2": "This variable is overridable"
}

TASK [test : Test] *******************************************************************************************************************************************************
ok: [localhost] => {
            "msg": "Test role"
}

TASK [test : Include] ****************************************************************************************************************************************************
included: /home/doublew/projects/ansible/test/roles/test/tasks/inc.yml for localhost

TASK [test : Include from test] ******************************************************************************************************************************************
ok: [localhost] => {
            "msg": "included"
}

TASK [test : Imported] ***************************************************************************************************************************************************
ok: [localhost] => {
            "msg": "Imported task"
}

TASK [test : Include false] **********************************************************************************************************************************************
skipping: [localhost]

TASK [test : one] ********************************************************************************************************************************************************
skipping: [localhost]

TASK [test : two] ********************************************************************************************************************************************************
skipping: [localhost]

PLAY RECAP ***************************************************************************************************************************************************************
localhost                  : ok=12   changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
```

### Tag `dep`

The command `ansible-playbook main.yml --tags dep`, try to run all tasks with the tags `dep`. Since the `dep` role is imported and tagged with `dep`, the whole tasks inside `dep` role would inherit the tag. Keep in mind that the tasks are imported at the very beginning just before Ansible is trying to parse the YAML file. This is how `main.yml` would roughly look [like](#code-snippet-1) for Ansible when it parses the file and execute it. Notice that every task now have the `dep` tag and when the playbook is run, all these tasks are executed.

###### Code Snippet 1

```YAML
---
- name: Play
  hosts: localhost
  become: false
  gather_facts: false
  tasks:
    - name: dep
      vars:
        const_var: "This variable is not overridable"
        dyn_var1: "Override dyn_var1"
        dyn_var2: "This variable is not overridable"
      block:
        - name: Debug
          ansible.builtin.debug:
            msg: "Dep roles"
          tags:
            - dep_task
            - dep

        - name: Debug const_var
          ansible.builtin.debug:
            var: const_var
          tags: dep

        - name: Debug dyn_var1
          ansible.builtin.debug:
            var: dyn_var1
          tags: dep

        - name: Debug dyn_var2
          ansible.builtin.debug:
            var: dyn_var2
          tags: dep

# `test` role is not shown for clarity sake
```

###### Output for tag `dep`

```sh
test
❯ ansible-playbook main.yml --tags dep
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Play] **************************************************************************************************************************************************************

TASK [dep : Debug] *******************************************************************************************************************************************************
ok: [localhost] => {
            "msg": "Dep roles"
}

TASK [dep : Debug const_var] *********************************************************************************************************************************************
ok: [localhost] => {
            "const_var": "This variable is not overridable"
}

TASK [dep : Debug dyn_var1] **********************************************************************************************************************************************
ok: [localhost] => {
            "dyn_var1": "Override dyn_var1"
}

TASK [dep : Debug dyn_var2] **********************************************************************************************************************************************
ok: [localhost] => {
            "dyn_var2": "This variable is overridable"
}

PLAY RECAP ***************************************************************************************************************************************************************
localhost                  : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Tag `dep_task`

If we only specified `dep_task` tag, based on [code snippet 1](#code-snippet-1), only the first task, named `Debug`, would run while the three below would not run. See output [below](#output-for-tag-dep_task).

###### Output for tag `dep_task`

```sh
test
❯ ansible-playbook main.yml --tags dep_task
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Play] **************************************************************************************************************************************************************

TASK [dep : Debug] *******************************************************************************************************************************************************
ok: [localhost] => {
            "msg": "Dep roles"
}

TASK [dep : Debug] *******************************************************************************************************************************************************
ok: [localhost] => {
            "msg": "Dep roles"
}

PLAY RECAP ***************************************************************************************************************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Tag `test`

For this command, since the role `test` has its dependencies defined under `roles/test/meta/main.yml`, it would run the dependency first before executing tasks in `roles/test/tasks/main.yml`. [Code snippet 2](#code-snippet-2) provides rough estimation on how the `main.yml` file would look like. Notice that the `include_tasks` is still there but the `import_tasks` is substituted. This is the main difference between the two. The `include_tasks` task would only be substituted with the tasks in the specified file when it mets the condition. However, when this tasks is included, the tags specified under `include_tasks` would not be inherited (see under [references](#references) section). One thing to note is that tasks under `block` would inherit whatever tags specified in the `block` level.

###### Code Snippet 2

```YAML
---
- name: Play
  hosts: localhost
  become: false
  gather_facts: false
  tasks:
  # `dep` role is not shown for clarity sake
    - name: test
      block:
        - name: Dep role dependency
          tags:
            - test_dep
            - test
          block:
            # `dep` role is not shown for clarity sake
            # but every task would have its original tags plus `test_dep` and `test`
            ...

        - name: Test
          ansible.builtin.debug:
            msg: "Test roles"
          tags:
            - test_task
            - test

        - name: Include
          ansible.builtin.include_tasks:
            file: inc.yml
          tags:
            - test_inc
            - test

        - name: Import
          block:
            - name: Imported
              ansible.builtin.debug:
                msg: "Imported task"
              tags:
                - test_imp_task
                - test_imp
                - test

        - name: Include false
          ansible.builtin.include_tasks:
            file: inc_false.yml
          tags:
            - test_incfalse
            - test

        - name: Block
          when: 1 | int == 0
          tags:
            - test_block
            - test
          block:
            - name: one
              tags: test_block_task1
              ansible.builtin.debug:
                msg: "abcd"

            - name: two
              tags: test_block_task2
              ansible.builtin.debug:
                msg: "efgh"
```

###### Output for tag `test`

```sh
test
❯ ansible-playbook main.yml --tags test
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Play] **************************************************************************************************************************************************************

TASK [dep : Debug] *******************************************************************************************************************************************************
ok: [localhost] => {
            "msg": "Dep roles"
}

TASK [dep : Debug const_var] *********************************************************************************************************************************************
ok: [localhost] => {
            "const_var": "This variable is not overridable"
}

TASK [dep : Debug dyn_var1] **********************************************************************************************************************************************
ok: [localhost] => {
            "dyn_var1": "Override dyn_var1"
}

TASK [dep : Debug dyn_var2] **********************************************************************************************************************************************
ok: [localhost] => {
            "dyn_var2": "This variable is overridable"
}

TASK [test : Test] *******************************************************************************************************************************************************
ok: [localhost] => {
            "msg": "Test role"
}

TASK [test : Include] ****************************************************************************************************************************************************
included: /home/doublew/projects/ansible/test/roles/test/tasks/inc.yml for localhost

TASK [test : Include from test] ******************************************************************************************************************************************
ok: [localhost] => {
            "msg": "included"
}

TASK [test : Imported] ***************************************************************************************************************************************************
ok: [localhost] => {
            "msg": "Imported task"
}

TASK [test : Include false] **********************************************************************************************************************************************
skipping: [localhost]

TASK [test : one] ********************************************************************************************************************************************************
skipping: [localhost]

TASK [test : two] ********************************************************************************************************************************************************
skipping: [localhost]

PLAY RECAP ***************************************************************************************************************************************************************
localhost                  : ok=8    changed=0    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
```

###### Output for tag `test_dep`

```sh
test
❯ ansible-playbook main.yml --tags test_dep
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Play] **************************************************************************************************************************************************************

TASK [dep : Debug] *******************************************************************************************************************************************************
ok: [localhost] => {
            "msg": "Dep roles"
}

TASK [dep : Debug const_var] *********************************************************************************************************************************************
ok: [localhost] => {
            "const_var": "This variable is not overridable"
}

TASK [dep : Debug dyn_var1] **********************************************************************************************************************************************
ok: [localhost] => {
            "dyn_var1": "Override dyn_var1"
}

TASK [dep : Debug dyn_var2] **********************************************************************************************************************************************
ok: [localhost] => {
            "dyn_var2": "This variable is overridable"
}

PLAY RECAP ***************************************************************************************************************************************************************
localhost                  : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Tag `test_task`

###### Output for tag `test_task`

```sh
test
❯ ansible-playbook main.yml --tags test_task
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Play] **************************************************************************************************************************************************************

TASK [test : Test] *******************************************************************************************************************************************************
ok: [localhost] => {
            "msg": "Test role"
}

PLAY RECAP ***************************************************************************************************************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Tag `test_inc`

For this example, ansible only run the `include_tasks` task but not the tasks inside `roles/test/tasks/inc.yml` since the tasks inside this file do not contain `test_inc` tag (not inherited).

###### Output for tag `test_inc`

```sh
test
❯ ansible-playbook main.yml --tags test_inc
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Play] **************************************************************************************************************************************************************

TASK [test : Include] ****************************************************************************************************************************************************
included: /home/doublew/projects/ansible/test/roles/test/tasks/inc.yml for localhost

PLAY RECAP ***************************************************************************************************************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Tag `test_inc_task`

For this example, none of the task is run. This is because even though the `include_tasks` met the condition (boolean true), it does not contain the specified tag. Thus, the tasks under `roles/test/tasks/inc.yml` is not loaded even though they have the specified tag.

###### Output for tag `test_inc_task`

```sh
test
❯ ansible-playbook main.yml --tags test_inc_task
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Play] **************************************************************************************************************************************************************

PLAY RECAP ***************************************************************************************************************************************************************
```

### Tag `test_inc`, `test_inc_task`

In this example, we specified two tags such that the `include_tasks` task is run (since it has `test_inc` tag) and the included task is also run (since it has `test_inc_task` tag).

###### Output for tags `test_inc`, `test_inc_task`

```
test
❯ ansible-playbook main.yml --tags test_inc,test_inc_task
[WARNING]: provided hosts list is empty, only localhost is available. Note that
the implicit localhost does not match 'all'

PLAY [Play] ***********************************************************************

TASK [test : Include] *************************************************************
included: /home/doublew/projects/ansible/test/roles/test/tasks/inc.yml for localhost

TASK [test : Include from test] ***************************************************
ok: [localhost] => {
    "msg": "included"
}

PLAY RECAP ************************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Tag `test_imp`

###### Output for tag `test_imp`

```sh
test
❯ ansible-playbook main.yml --tags test_imp
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Play] **************************************************************************************************************************************************************

TASK [test : Imported] ***************************************************************************************************************************************************
ok: [localhost] => {
            "msg": "Imported task"
}

PLAY RECAP ***************************************************************************************************************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Tag `test_imp_task`

###### Output for tag `test_imp_task`

```sh
test
❯ ansible-playbook main.yml --tags test_imp_task
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Play] **************************************************************************************************************************************************************

TASK [test : Imported] ***************************************************************************************************************************************************
ok: [localhost] => {
            "msg": "Imported task"
}

PLAY RECAP ***************************************************************************************************************************************************************
localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Tag `test_incfalse`

###### Output for tag `test_incfalse`

```sh
test
❯ ansible-playbook main.yml --tags test_incfalse
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Play] **************************************************************************************************************************************************************

TASK [test : Include false] **********************************************************************************************************************************************
skipping: [localhost]

PLAY RECAP ***************************************************************************************************************************************************************
localhost                  : ok=0    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```

### Tag `test_incfalse_task`

###### Output for tag `test_incfalse_task`

```sh
test
❯ ansible-playbook main.yml --tags test_incfalse_task
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Play] **************************************************************************************************************************************************************

PLAY RECAP ***************************************************************************************************************************************************************
```

### Tag `test_block`

###### Output for tag `test_block`

```sh
test
❯ ansible-playbook main.yml --tags test_block
[WARNING]: provided hosts list is empty, only localhost is available. Note that
the implicit localhost does not match 'all'

PLAY [Play] ***********************************************************************

TASK [test : one] *****************************************************************
skipping: [localhost]

TASK [test : two] *****************************************************************
skipping: [localhost]

PLAY RECAP ************************************************************************
localhost                  : ok=0    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
```

### Tag `test_block_task1`

###### Output for tag `test_block_task1`

```sh
test
❯ ansible-playbook main.yml --tags test_block_task1
[WARNING]: provided hosts list is empty, only localhost is available. Note that
the implicit localhost does not match 'all'

PLAY [Play] ***********************************************************************

TASK [test : one] *****************************************************************
skipping: [localhost]

PLAY RECAP ************************************************************************
localhost                  : ok=0    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```

### References

- [Ansible 101 - Include vs Import](https://www.ansiblejunky.com/blog/ansible-101-include-vs-import/)
