# Diffusion Tests Role

A comprehensive Ansible testing role for validating container environments, network services, shell commands, and database configurations using the [Diffusion framework](https://github.com/Polar-Team/diffusion).

> 🚀 **Quick Start**: See [QUICKSTART.md](QUICKSTART.md) to get started in 5 minutes  
> 📚 **Complete Guide**: See [SUMMARY.md](SUMMARY.md) for a complete overview  
> 📖 **Documentation**: See [DOCUMENTATION_INDEX.md](DOCUMENTATION_INDEX.md) for all documentation  
> 🔧 **Framework**: This role is designed to work with the [Diffusion testing framework](https://github.com/Polar-Team/diffusion)

## Description

This role provides a DRY (Don't Repeat Yourself) approach to testing common infrastructure components in CI/CD pipelines. It validates:

- Docker container states, images, and health checks
- Network port availability and states
- Shell command outputs
- URI/HTTP endpoint accessibility
- PostgreSQL database configurations and data

## Requirements

### Ansible
```yaml
ansible-core >= 2.15
```

### Python Dependencies
```yaml
docker >= 6.0.0
psycopg2 >= 2.9.0  # For PostgreSQL tests
```

### Ansible Collections
```yaml
community.general >= 7.4.0
community.docker >= 3.4.0
community.postgresql >= 2.4.0
```

Install collections:
```bash
ansible-galaxy collection install community.general
ansible-galaxy collection install community.docker
ansible-galaxy collection install community.postgresql
```

## Role Variables

All test types are optional. Define only the variables for the tests you need to run.

### Docker Container Tests

Validates Docker container states, images, and health status.

```yaml
verify_docker_containers:
  - name: "nginx-web"                    # required - container name
    state: "running"                     # required - expected state
    image: "nginx:alpine"                # optional - verify image
    health: "healthy"                    # optional - verify health check status

verify_docker_env: {}                    # optional - environment variables for docker commands
```

**Supported states:** `running`, `stopped`, `paused`, `restarting`, `created`

**Note:** Health check validation requires the container to have a health check configured.

### Port Tests

Verifies network ports are in expected states (listening or not listening).

```yaml
verify_ports:
  - port: 8080                           # required - port number (1-65535)
    host: "127.0.0.1"                    # optional - defaults to 127.0.0.1
    timeout: 10                          # optional - seconds to wait, defaults to 5
    state: "started"                     # optional - "started" or "stopped", defaults to "started"
    description: "Web API"               # optional - for clearer output
    delay: 0                             # optional - seconds before first check
    sleep: 1                             # optional - seconds between retries

# Global configuration (optional)
port_summary_limit: 10                   # Number of failed ports shown in summary (default: 10)
port_check_show_success: false           # Show success messages at default verbosity (default: false)
```

**Supported states:**
- `started`: Port must be listening (accepting connections)
- `stopped`: Port must NOT be listening
- `drained`: Port must stop accepting new connections (rare, for graceful shutdown)

**Performance Note:** Ports are checked sequentially. For many ports with long timeouts, consider using async/poll.

### Shell Command Tests

Executes shell commands and validates output patterns.

```yaml
verify_output_in_cmd:
  - cmd: "echo 'Hello World'"            # required - command to execute
    output: "Hello"                      # required - expected output pattern
    exact_match: false                   # optional - true for exact match, false for substring (default: false)
    use_shell: false                     # optional - true to use shell module (default: false)
```

**Note:** Use `use_shell: true` only when you need shell features (pipes, redirects, etc.). The `command` module is safer and preferred.

### URI/HTTP Tests

Tests URI accessibility with optional SNI (Server Name Indication) support.

```yaml
verify_uris:
  - "https://api.example.com/health"
  - "http://localhost:8080/status"

# Optional configuration
uri_test_ip: "127.0.0.1"                 # IP for SNI testing (default: 127.0.0.1)
uri_validate_certs: false                # Enable SSL cert validation (default: false for testing)
uri_timeout: 10                          # Request timeout in seconds (default: 10)
uri_expected_status: [200, 301, 302]     # Expected HTTP status codes (default: [200, 301, 302])
uri_follow_redirects: "safe"             # Redirect behavior: "safe", "all", "none" (default: "safe")
```

**SNI Testing:** The role temporarily adds hosts file entries to enable SNI testing against localhost. Entries are automatically cleaned up.

### PostgreSQL Tests

Validates PostgreSQL database existence, roles, tables, and data.

```yaml
# Connection parameters (required)
postgres_db: "production_db"
postgres_host: "localhost"
postgres_user: "postgres"
postgres_password: "{{ vault_password }}"
postgres_port: 5432                      # optional - defaults to 5432

# Test definitions (all optional - use empty lists to skip)
postgres_expected_roles:
  - "app_user"
  - "readonly_user"
  - "backup_user"

postgres_expected_tables:
  - "public.users"                       # Can be schema-qualified
  - "public.orders"
  - "audit.log_entries"

postgres_expected_records:
  - query: "SELECT * FROM users WHERE role = 'admin'"
    description: "At least one admin user exists"  # optional
  - query: "SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '1 day'"
    description: "Recent orders exist"
```

**Security Notes:**
- Credentials are protected with `no_log: true`
- Only use trusted input in queries to prevent SQL injection
- Only SELECT queries are supported for record validation

**Batch Validation:** Role/table checks report ALL missing items in one run, not just the first failure.

## Dependencies

None. This is a standalone testing role.

### Development Dependencies

For running the test suite:

```yaml
# Python packages (requirements.txt)
ansible-core >= 2.15
molecule >= 6.0.0
molecule-docker >= 2.1.0
docker >= 6.0.0
psycopg2-binary >= 2.9.0

# Ansible collections (requirements.yml)
community.general >= 7.4.0
community.docker >= 3.4.0
community.postgresql >= 2.4.0
```

## Example Molecule Verify Playbook

### Basic Usage

```yaml
---
- name: Verify
  hosts: all
  gather_facts: false
  tasks:
    - name: Test Docker containers
      ansible.builtin.include_role:
        name: diffusion_tests
      vars:
        verify_docker_containers:
          - name: "test-nginx"
            state: "running"
            image: "nginx:alpine"
```

### Comprehensive Testing

```yaml
---
- name: Verify
  hosts: all
  gather_facts: false
  vars:
    system_name: "Debian-12"

  tasks:
    # Test ports are accessible
    - name: Verify service ports
      ansible.builtin.include_role:
        name: diffusion_tests
      vars:
        verify_ports:
          - port: 80
            description: "HTTP port"
          - port: 443
            description: "HTTPS port"
          - port: 5432
            host: "db.example.com"
            timeout: 10
            description: "PostgreSQL"

    # Test Docker containers
    - name: Verify containers
      ansible.builtin.include_role:
        name: diffusion_tests
      vars:
        verify_docker_containers:
          - name: "{{ system_name }}-web"
            state: "running"
            image: "nginx:alpine"
            health: "healthy"
          - name: "{{ system_name }}-db"
            state: "running"
            image: "postgres:14"

    # Test shell commands
    - name: Verify system commands
      ansible.builtin.include_role:
        name: diffusion_tests
      vars:
        verify_output_in_cmd:
          - cmd: "systemctl is-active nginx"
            output: "active"
          - cmd: "df -h / | tail -1 | awk '{print $5}'"
            output: "%"
            use_shell: true

    # Test HTTP endpoints
    - name: Verify API endpoints
      ansible.builtin.include_role:
        name: diffusion_tests
      vars:
        verify_uris:
          - "http://localhost/health"
          - "https://api.example.com/status"
        uri_validate_certs: true

    # Test PostgreSQL database
    - name: Verify database
      ansible.builtin.include_role:
        name: diffusion_tests
      vars:
        postgres_db: "production"
        postgres_host: "localhost"
        postgres_user: "postgres"
        postgres_password: "{{ vault_db_password }}"
        postgres_expected_roles:
          - "app_user"
          - "readonly_user"
        postgres_expected_tables:
          - "public.users"
          - "public.orders"
        postgres_expected_records:
          - query: "SELECT * FROM users WHERE is_admin = true"
            description: "Admin users exist"
```

### Security-Focused Testing

```yaml
---
- name: Verify security posture
  hosts: all
  gather_facts: false
  tasks:
    - name: Verify insecure ports are closed
      ansible.builtin.include_role:
        name: diffusion_tests
      vars:
        verify_ports:
          - port: 23
            state: "stopped"
            description: "Telnet disabled"
          - port: 21
            state: "stopped"
            description: "FTP disabled"
          - port: 445
            state: "stopped"
            description: "SMB disabled"
```

## Output Examples

### Successful Port Check
```
TASK [Display port check summary] ****
ok: [localhost] =>
  msg: |-
    Port Verification Summary
    ========================================
    Total checks: 3
    Passed: 3
    Failed: 0
```

### Failed Docker Container Check
```
TASK [Validate container state and configuration] ****
fatal: [localhost]: FAILED! =>
  msg: |-
    Container 'nginx-web' validation failed:
    Expected state: running, Actual: stopped
    Expected image: nginx:alpine, Actual: nginx:latest
```

### PostgreSQL Batch Validation
```
TASK [Assert all required roles exist] ****
fatal: [localhost]: FAILED! =>
  msg: |-
    [FAIL] PostgreSQL roles validation failed
    Missing roles: app_user, backup_user
    Expected roles: app_user, readonly_user, backup_user
    Available roles: postgres, readonly_user, test_user
```

## Best Practices

### 1. Use Ansible Vault for Secrets
```yaml
postgres_password: "{{ vault_postgres_password }}"
```

### 2. Test in Order of Dependencies
```yaml
tasks:
  - name: Test network connectivity first
    include_role: { name: diffusion_tests }
    vars: { verify_ports: [...] }

  - name: Then test containers
    include_role: { name: diffusion_tests }
    vars: { verify_docker_containers: [...] }

  - name: Finally test application data
    include_role: { name: diffusion_tests }
    vars: { postgres_expected_records: [...] }
```

### 3. Use Descriptive Names and Descriptions
```yaml
verify_ports:
  - port: 8080
    description: "Application API (main service)"
  - port: 9090
    description: "Prometheus metrics endpoint"
```

### 4. Leverage Verbosity Flags
```bash
# Quiet output (only summaries and failures)
molecule verify

# Detailed output (includes success messages)
molecule verify -- -v

# Debug output (includes all task details)
molecule verify -- -vv
```

### 5. Use Empty Lists to Skip Checks
```yaml
postgres_expected_roles: []      # Skip role validation
postgres_expected_tables:        # Check tables only
  - "users"
  - "orders"
```

## Troubleshooting

### Docker Permission Errors
If you see "permission denied" errors:
```yaml
verify_docker_env:
  DOCKER_HOST: "unix:///var/run/docker.sock"
```

### PostgreSQL Connection Failures
Verify credentials and network access:
```bash
psql -h localhost -U postgres -d mydb -c "SELECT 1"
```

### Port Check Timeouts
Increase timeout for slow services:
```yaml
verify_ports:
  - port: 8080
    timeout: 30
    delay: 10  # Wait 10s before first check
```

### URI SSL Certificate Errors
For testing environments, disable certificate validation:
```yaml
uri_validate_certs: false
```

For production, ensure proper certificates:
```yaml
uri_validate_certs: true
```

## Testing the Role

This role uses the [Diffusion framework](https://github.com/Polar-Team/diffusion) for testing.

### Prerequisites

1. **Install Diffusion CLI**:
   ```bash
   # Using Go install
   go install github.com/Polar-Team/diffusion@latest
   
   # Or from source
   git clone https://github.com/Polar-Team/diffusion.git
   cd diffusion
   make build
   ```

2. **Install dependencies**:
   ```bash
   make install
   # Or manually:
   pip install -r requirements.txt
   ansible-galaxy collection install -r requirements.yml
   ```

### Running Tests with Diffusion

```bash
# Run all tests
diffusion molecule --test

# Run specific test stages
diffusion molecule --converge    # Setup environment
diffusion molecule --verify      # Run tests
diffusion molecule --lint        # Run linting
diffusion molecule --idempotence # Test idempotence

# Run specific test categories using tags
diffusion molecule --verify -- --tags ports
diffusion molecule --verify -- --tags docker
diffusion molecule --verify -- --tags cmd
diffusion molecule --verify -- --tags postgres
diffusion molecule --verify -- --tags uri

# Run multiple categories
diffusion molecule --verify -- --tags "ports,docker"
```

### Using Makefile

```bash
make help          # Show all available commands
make install       # Install all dependencies
make test          # Run all tests
make lint          # Run linting
make verify        # Run verification tests
make clean         # Clean up test environments
```

### Test Structure

Tests are organized in the `tests/` directory following Diffusion framework conventions:

```
tests/
├── test.yml              # Main test orchestration
├── ports/
│   ├── setup.yml        # Port test setup
│   └── test_ports.yml   # Port test cases (4 tests)
├── docker/
│   ├── setup.yml        # Docker test setup
│   └── test_docker.yml  # Docker test cases (6 tests)
├── cmd/
│   └── test_cmd.yml     # Command test cases (9 tests)
├── postgres/
│   ├── setup.yml        # PostgreSQL setup
│   └── test_postgres.yml # PostgreSQL test cases (7 tests)
└── uri/
    ├── setup.yml        # URI test setup
    └── test_uri.yml     # URI test cases (8 tests)
```

### Test Categories

All tests support tag-based execution:

- **ports** - Network port state verification (4 test cases)
- **docker** - Container validation (6 test cases)
- **cmd** - Shell command output validation (9 test cases)
- **postgres** - PostgreSQL database validation (7 test cases)
- **uri** - HTTP/HTTPS endpoint testing (8 test cases)

**Total**: 34 test cases with 100% feature coverage

For detailed test documentation, see [tests/README.md](tests/README.md)

## License

MIT

## Author Information

**Polar Team - Daniel Dalavurak**

Created for comprehensive infrastructure testing in CI/CD pipelines using Molecule framework.

## Contributing

Contributions are welcome! Please ensure:
- All tests pass
- Code follows existing patterns
- Documentation is updated
- Security best practices are maintained

### Quick Links

- [Quick Reference Guide](QUICK_REFERENCE.md) - Fast command reference
- [Testing Guide](TESTING.md) - Complete testing documentation
- [Test Summary](TEST_SUMMARY.md) - Detailed test case breakdown

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for detailed version history.

### Latest Release: v1.2.0 (2024-12-23)
- Updated CI/CD workflow to tag-based releases
- Manual control over release timing and versioning
- Automated testing on tag push
- Automatic GitHub release creation after successful tests

### Automated Releases

This project uses automated releases triggered by git tags:

1. **Create and push a tag**:
   ```bash
   git tag -a v1.2.0 -m "Release v1.2.0"
   git push origin v1.2.0
   ```

2. **Automated workflow**:
   - Runs all tests with Diffusion framework
   - Creates GitHub release if tests pass
   - Generates changelog from commits

3. **Installation**:
   ```yaml
   # requirements.yml
   roles:
     - name: polar.diffusion_tests
       version: v1.2.0
   ```

### Version 1.0.0
- Initial release
- Docker container validation
- Port state checking
- Shell command output validation
- URI/HTTP endpoint testing
- PostgreSQL database validation
- Batch validation with complete feedback
- Security hardening with no_log
- Comprehensive error handling

