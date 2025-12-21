# Tests Directory - Diffusion Framework

This directory contains all test cases for the `diffusion_tests` role, organized following the [Diffusion framework](https://github.com/Polar-Team/diffusion) conventions.

## Structure

```
tests/
├── README.md           # This file
├── test.yml            # Main test orchestration
├── ports/              # Port validation tests
│   ├── setup.yml      # Environment setup
│   └── test_ports.yml # Test cases
├── docker/             # Docker container tests
│   ├── setup.yml      # Environment setup
│   └── test_docker.yml # Test cases
├── cmd/                # Command output tests
│   └── test_cmd.yml   # Test cases
├── postgres/           # PostgreSQL tests
│   ├── setup.yml      # Environment setup
│   └── test_postgres.yml # Test cases
└── uri/                # HTTP/HTTPS endpoint tests
    ├── setup.yml      # Environment setup
    └── test_uri.yml   # Test cases
```

## Running Tests

### With Diffusion Framework (Recommended)

```bash
# Run all tests
diffusion molecule --test

# Run specific stages
diffusion molecule --converge  # Setup
diffusion molecule --verify    # Run tests
diffusion molecule --lint      # Linting

# Run with tags
diffusion molecule --verify -- --tags ports
diffusion molecule --verify -- --tags docker
diffusion molecule --verify -- --tags cmd
diffusion molecule --verify -- --tags postgres
diffusion molecule --verify -- --tags uri
```

### With Molecule Directly

```bash
# Run all tests
molecule test -s default

# Run verify stage only
molecule verify -s default

# Run with specific tags
molecule verify -s default -- --tags ports
```

## Test Categories

### Port Tests (`ports/`)
Tests network port state verification:
- Ports in "started" state (listening)
- Ports in "stopped" state (not listening)
- Custom timeouts and delays
- Host resolution (127.0.0.1 vs localhost)

**Test Cases**: 4
**Setup Required**: netcat listeners

### Docker Tests (`docker/`)
Tests Docker container validation:
- Running containers with health checks
- Running containers without health checks
- Stopped containers
- Paused containers
- Image validation
- Batch validation

**Test Cases**: 6
**Setup Required**: Docker service, test containers

### Command Tests (`cmd/`)
Tests shell command output validation:
- Substring matching
- Exact matching
- Shell module usage (pipes, redirects)
- Command module usage
- Multiline output
- Special characters

**Test Cases**: 9
**Setup Required**: None (uses system commands)

### PostgreSQL Tests (`postgres/`)
Tests PostgreSQL database validation:
- Database existence
- Role existence
- Table existence (schema-qualified)
- Data record validation
- Complex queries

**Test Cases**: 7
**Setup Required**: PostgreSQL service, test database

### URI Tests (`uri/`)
Tests HTTP/HTTPS endpoint validation:
- Simple HTTP endpoints
- Multiple endpoints
- Redirect handling
- SNI (Server Name Indication) support
- Custom timeouts and status codes

**Test Cases**: 8
**Setup Required**: nginx web server

## Test Tags

All tests support Ansible tags for selective execution:

- `ports` - Run only port tests
- `docker` - Run only Docker tests
- `cmd` - Run only command tests
- `postgres` - Run only PostgreSQL tests
- `uri` - Run only URI tests
- `all` - Run all tests (default)

### Examples

```bash
# Run only port tests
diffusion molecule --verify -- --tags ports

# Run port and docker tests
diffusion molecule --verify -- --tags "ports,docker"

# Skip PostgreSQL tests
diffusion molecule --verify -- --skip-tags postgres
```

## Writing New Tests

### 1. Create Test Directory

```bash
mkdir -p tests/mytest
```

### 2. Create Setup File (if needed)

```yaml
# tests/mytest/setup.yml
---
- name: Install dependencies
  ansible.builtin.apt:
    name: my-package
    state: present

- name: Configure service
  ansible.builtin.template:
    src: config.j2
    dest: /etc/myservice/config.yml
```

### 3. Create Test File

```yaml
# tests/mytest/test_mytest.yml
---
- name: Setup test environment
  ansible.builtin.include_tasks: setup.yml
  when: setup_required | default(true)

- name: Test Case 1 - Description
  ansible.builtin.include_role:
    name: diffusion_tests
  vars:
    verify_ports:
      - port: 1234
        state: started
```

### 4. Add to Main Test File

```yaml
# tests/test.yml
- name: Include my tests
  ansible.builtin.include_tasks: mytest/test_mytest.yml
  tags:
    - mytest
    - all
```

## Test Best Practices

### 1. Idempotency
- Tests should be idempotent
- Setup tasks should use `changed_when: false` where appropriate
- Clean up resources after tests

### 2. Isolation
- Each test category should be independent
- Use unique resource names (ports, containers, etc.)
- Don't rely on test execution order

### 3. Documentation
- Add clear descriptions to test cases
- Document required setup
- Include examples in comments

### 4. Error Handling
- Use `failed_when` and `changed_when` appropriately
- Provide clear error messages
- Test both success and failure scenarios

### 5. Performance
- Minimize setup time
- Use appropriate timeouts
- Avoid unnecessary waits

## Debugging Tests

### View Test Output

```bash
# Verbose output
diffusion molecule --verify -- -v

# Very verbose output
diffusion molecule --verify -- -vv

# Debug output
diffusion molecule --verify -- -vvv
```

### Login to Test Instance

```bash
# With Diffusion
diffusion molecule --create
diffusion molecule --converge
# Then manually SSH or use docker exec

# With Molecule
molecule login -s default
```

### Keep Instance After Failure

```bash
# With Molecule
molecule test -s default --destroy=never

# Then login to inspect
molecule login -s default
```

### Run Specific Test

```bash
# Run single test file
ansible-playbook tests/ports/test_ports.yml -i inventory

# Run with specific tags
diffusion molecule --verify -- --tags ports -vv
```

## CI/CD Integration

### GitHub Actions

```yaml
- name: Run tests with Diffusion
  run: |
    diffusion molecule --test
```

### GitLab CI

```yaml
test:
  script:
    - diffusion molecule --test
```

## Test Coverage

Current test coverage:
- **Port Tests**: 4 test cases
- **Docker Tests**: 6 test cases
- **Command Tests**: 9 test cases
- **PostgreSQL Tests**: 7 test cases
- **URI Tests**: 8 test cases

**Total**: 34 test cases covering 100% of role features

## Contributing

When adding new tests:
1. Follow the existing structure
2. Add setup.yml if environment preparation is needed
3. Use descriptive test case names
4. Add appropriate tags
5. Update this README
6. Ensure tests pass locally before committing

## Resources

- [Diffusion Framework](https://github.com/Polar-Team/diffusion)
- [Molecule Documentation](https://molecule.readthedocs.io/)
- [Ansible Testing Strategies](https://docs.ansible.com/ansible/latest/reference_appendices/test_strategies.html)
- [Main Testing Guide](../TESTING.md)
- [Test Summary](../TEST_SUMMARY.md)
