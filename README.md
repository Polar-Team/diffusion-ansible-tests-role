Role Name
=========

This role created for testing molecule base cases in diffusion framework. Use verify molecule to use that DRY.

Requirements
------------


**ansible-core** >= 2.15
**docker api** >= 1.25

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

**community.general** collection >= 6.4.0
**community.docker** collection >= 5.0.1


Example Verify
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: servers
      roles:
         - { role: username.rolename, x: 42 }

License
-------

MIT

Author Information
------------------

An optional section for the role authors to include contact information, or a website (HTML is not allowed).
