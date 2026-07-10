> ## Documentation Index
> Fetch the complete documentation index at: https://docs.osmedeus.org/llms.txt
> Use this file to discover all available pages before exploring further.

# Runners

> Execution environments for commands and steps

Runners execute commands in different environments.

## Runner Types

| Runner   | Description              | Use Case                               |
| -------- | ------------------------ | -------------------------------------- |
| `host`   | Local machine execution  | Default, fast, no isolation            |
| `docker` | Container execution      | Isolated, reproducible, tool packaging |
| `ssh`    | Remote machine execution | Distributed scanning, remote resources |

## Host Runner

Executes commands on the local machine using the shell.

### Configuration

```yaml theme={null}
kind: module
name: local-scan
runner: host        # Default, can be omitted

steps:
  - name: scan
    type: bash
    command: nmap -sV {{target}}
```

### Characteristics

* Uses `sh -c` for command execution
* Inherits environment from Osmedeus process
* No isolation between steps
* Fastest execution

## Docker Runner

Executes commands inside Docker containers.

### Module-Level Configuration

```yaml theme={null}
kind: module
name: docker-scan
runner: docker
runner_config:
  image: projectdiscovery/nuclei:latest
  volumes:
    - "{{Output}}:/output"
  environment:
    - "API_KEY={{api_key}}"
  persistent: false    # Default: ephemeral containers

steps:
  - name: scan
    type: bash
    command: nuclei -u {{target}} -o /output/nuclei.txt
```

### Runner Config Options

| Option        | Type      | Description                          |
| ------------- | --------- | ------------------------------------ |
| `image`       | string    | Docker image (required)              |
| `volumes`     | \[]string | Volume mounts (`host:container`)     |
| `environment` | \[]string | Environment variables                |
| `persistent`  | bool      | Keep container running between steps |
| `network`     | string    | Docker network name                  |
| `extra_args`  | \[]string | Additional docker run arguments      |

### Execution Modes

**Ephemeral (default)**: Each step runs `docker run --rm`

```yaml theme={null}
runner_config:
  image: alpine:latest
  persistent: false    # New container per step
```

**Persistent**: Container stays running, steps use `docker exec`

```yaml theme={null}
runner_config:
  image: alpine:latest
  persistent: true     # Reuse container
```

### Per-Step Docker (remote-bash)

Use Docker for specific steps without module-level runner:

```yaml theme={null}
kind: module
name: hybrid
runner: host

steps:
  - name: local-step
    type: bash
    command: echo "Running locally"

  - name: docker-step
    type: remote-bash
    step_runner: docker
    step_runner_config:
      image: alpine:latest
      volumes:
        - "{{Output}}:/output"
    command: cat /etc/os-release
```

## SSH Runner

Executes commands on remote machines via SSH.

### Module-Level Configuration

```yaml theme={null}
kind: module
name: remote-scan
runner: ssh
runner_config:
  host: scanner.example.com
  port: 22
  user: scanner
  key_file: ~/.ssh/scanner_key
  # OR
  password: secret     # Not recommended

steps:
  - name: scan
    type: bash
    command: nmap -sV {{target}}
```

### Runner Config Options

| Option        | Type   | Description                |
| ------------- | ------ | -------------------------- |
| `host`        | string | SSH hostname (required)    |
| `port`        | int    | SSH port (default: 22)     |
| `user`        | string | SSH username (required)    |
| `key_file`    | string | Path to private key        |
| `password`    | string | SSH password (less secure) |
| `known_hosts` | string | Path to known\_hosts file  |

### Per-Step SSH (remote-bash)

Use SSH for specific steps:

```yaml theme={null}
kind: module
name: hybrid
runner: host

steps:
  - name: local-prep
    type: bash
    command: echo {{target}} > /tmp/target.txt

  - name: remote-scan
    type: remote-bash
    step_runner: ssh
    step_runner_config:
      host: "{{ssh_host}}"
      port: 22
      user: "{{ssh_user}}"
      key_file: ~/.ssh/id_rsa
    command: nmap -sV {{target}}
```

### File Transfer

Copy files from remote to local:

```yaml theme={null}
- name: remote-scan
  type: remote-bash
  step_runner: ssh
  step_runner_config:
    host: scanner.example.com
    user: scanner
    key_file: ~/.ssh/key
  command: nmap -sV {{target}} -oN /tmp/result.txt
  step_remote_file: /tmp/result.txt
  host_output_file: "{{Output}}/nmap-result.txt"
```

## Runner Interface

All runners implement this interface:

```go theme={null}
type Runner interface {
    Execute(ctx context.Context, command string) (*CommandResult, error)
    Setup(ctx context.Context) error
    Cleanup(ctx context.Context) error
    Type() core.RunnerType
    IsRemote() bool
}

type CommandResult struct {
    Output   string
    ExitCode int
    Error    error
}
```

## Lifecycle

```
1. Setup()    - Initialize runner (connect SSH, start container)
2. Execute()  - Run commands (called per step)
3. Cleanup()  - Tear down (disconnect, remove container)
```

## Choosing a Runner

| Scenario                 | Recommended Runner        |
| ------------------------ | ------------------------- |
| Simple local scans       | `host`                    |
| Tool isolation           | `docker`                  |
| Reproducible builds      | `docker`                  |
| Remote server with tools | `ssh`                     |
| Distributed scanning     | `ssh` or distributed mode |
| Mixed environments       | `remote-bash` per step    |

## Best Practices

### Docker

1. **Use specific image tags**
   ```yaml theme={null}
   image: projectdiscovery/nuclei:v2.9.0  # Good
   image: projectdiscovery/nuclei:latest  # Less predictable
   ```

2. **Mount only needed volumes**
   ```yaml theme={null}
   volumes:
     - "{{Output}}:/output:rw"
     - "{{Data}}/templates:/templates:ro"
   ```

3. **Use persistent mode for many steps**
   ```yaml theme={null}
   runner_config:
     persistent: true  # Faster for multi-step workflows
   ```

### SSH

1. **Use key authentication**
   ```yaml theme={null}
   key_file: ~/.ssh/scanner_key
   # Avoid: password: secret
   ```

2. **Parameterize host details**
   ```yaml theme={null}
   runner_config:
     host: "{{ssh_host}}"
     user: "{{ssh_user}}"
   ```

3. **Check remote tool availability**
   ```yaml theme={null}
   - name: check-tools
     type: bash
     command: which nmap nuclei httpx
   ```

### remote-bash

1. **Use for hybrid workflows**
   * Local file preparation
   * Remote heavy scanning
   * Local result processing

2. **Transfer results back**
   ```yaml theme={null}
   step_remote_file: /remote/output.txt
   host_output_file: "{{Output}}/output.txt"
   ```

## Next Steps

* [Step Types](../workflows/step-types) - Using remote-bash
* [Deployment](../getting-started/deployment) - Distributed mode
* [Extending Runners](../extending/runners) - Custom runners
