Role Name
=========

This role created for testing molecule base cases in diffusion framework. Use verify molecule to use that DRY.

Requirements
------------

```
ansible-core >= 2.15
docker api >= 1.25
```
Role Variables
--------------

```
port_test          bool - Whether to test port mapping (default=false)
docker_health_test bool - Whether to test docker healthcheck (default=false)
docker_rootless    bool - Whether to test docker in rootless mode (default=true)
shell_test         bool - Whether to test shell command execution (default=false)
You can only use one true per role initializing.

```
Dependencies
------------
```
**community.general** collection >= 6.4.0
**community.docker** collection >= 5.0.1
```
Example Verify
----------------

Including an example of how to use your role in molecule verify (for instance, with variables passed in as parameters) is always nice for users too:
```yaml
---
- name: Verify
  hosts: all
  gather_facts: false
  vars:
   system_name: "Debian-12-12"
   docker_user_uid: 1000
  roles:
   - role: tests/diffusion_tests
     vars:
       port_test: true
       port: "9092"
   - role: tests/diffusion_tests
     vars:
       container_name: "{{ system_name }}-kafka"
       docker_health_test: true
```


License
-------

MIT

Author Information
------------------

Polar Team - Daniel Dalavurak
