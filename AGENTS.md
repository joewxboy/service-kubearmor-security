# AGENTS.md

This file provides guidance to agents when working with code in this repository.

## Project Overview

This repository contains an **Open Horizon edge service configuration** for deploying [KubeArmor](https://kubearmor.io/), an open-source runtime security enforcement system. KubeArmor provides container-aware security by restricting the behavior of containers and nodes at the system level using Linux Security Modules (LSMs) like AppArmor and BPF-LSM.

### Technology Stack
- **Open Horizon**: Edge computing platform for autonomous service deployment and lifecycle management
- **Docker**: Container runtime for KubeArmor deployment
- **KubeArmor**: Third-party security container (kubearmor/kubearmor:stable from DockerHub)
- **Policy-based deployment**: Uses Open Horizon's policy engine for automated service placement

### Architecture
This is an **infrastructure-as-code project**, not a traditional application codebase. It consists of:
- Service definition files (JSON) in the `horizon/` directory that describe the containerized service
- Policy files in the `horizon/` directory that control where and how the service deploys
- Makefile automation for publishing and managing the service lifecycle
- No custom source code - deploys a pre-built third-party container

**File Organization**: All Open Horizon configuration files are located in the `horizon/` subdirectory following best practices for Open Horizon projects.

The service requires **privileged container access** to interact with kernel security modules and must run on nodes with `openhorizon.allowPrivileged == true` and `purpose == security` properties.

## Building and Running

### Prerequisites
1. **Open Horizon Management Hub**: Access to an Open Horizon hub (community or commercial like IBM Edge Application Manager)
2. **Edge Node**: x86 Linux or arm64 (Raspberry Pi) with Open Horizon agent (anax) installed and registered
3. **Required utilities**: `gcc`, `make`, `git`, `jq`, `curl`, `net-tools`
4. **Agent status**: Must be in "unconfigured" state before deployment

### Key Commands

**Initial Setup:**
```bash
git clone https://github.com/open-horizon-services/service-kubearmor-security.git
cd service-kubearmor-security
make clean  # Verify make is working
hzn version  # Confirm Open Horizon agent is installed
```

**Local Testing (without Open Horizon):**
```bash
make init          # Create Docker volume
make run           # Run KubeArmor container locally
make test          # Verify service is responding
make attach        # Connect to running container shell
make stop          # Stop local container
```

**Publishing to Open Horizon Hub:**
```bash
make publish       # Publishes service definition, policies, and registers node
                   # This is the primary deployment command
```

**Individual Publishing Steps:**
```bash
make publish-service              # Publish service definition only
make publish-service-policy       # Publish service policy only
make publish-deployment-policy    # Publish deployment policy only
make agent-run                    # Register node with hub
```

**Monitoring and Debugging:**
```bash
make log           # View event logs and service logs
make deploy-check  # Verify policy compatibility before deployment
make check         # Display environment variables and service definition
```

**Cleanup:**
```bash
make agent-stop    # Unregister node and stop all agreements
make clean         # Remove container image and volume
make distclean     # Full cleanup including hub service removal
```

### Environment Variables

Key variables that can be customized (set before running make commands):

- `HZN_ORG_ID` - Open Horizon organization (default: `examples`)
- `SERVICE_NAME` - Service identifier (default: `service-kubearmor`)
- `SERVICE_VERSION` - Version number (default: `0.0.1`)
- `DOCKER_IMAGE_BASE` - Base image name (default: `kubearmor/kubearmor`)
- `DOCKER_IMAGE_VERSION` - Image tag (default: `stable`)
- `MY_TIME_ZONE` - Container timezone (default: `America/New_York`)
- `ARCH` - Target architecture (default: `amd64`, also supports `arm64`)

Example:
```bash
export HZN_ORG_ID=myorg
export SERVICE_VERSION=1.0.0
make publish
```

## Development Conventions

### Policy-Based Deployment Model

This service uses Open Horizon's **policy-based deployment** rather than patterns. Three policy files in the `horizon/` directory work together:

1. **horizon/node.policy.json** - Defines node properties:
   - `openhorizon.allowPrivileged: true` - Node allows privileged containers
   - `purpose: security` - Node is designated for security workloads

2. **horizon/service.policy.json** - Defines service constraints:
   - Requires nodes with both `openhorizon.allowPrivileged == true` AND `purpose == security`

3. **horizon/deployment.policy.json** - Defines deployment rules:
   - Matches service to nodes based on constraints
   - Specifies service version and user inputs
   - Uses wildcard architecture (`*`) for multi-arch support

**Policy Matching**: The Open Horizon agent automatically forms agreements when node properties satisfy both service and deployment policy constraints.

### Container Configuration

The KubeArmor container requires extensive host access:
- **Privileged mode**: Full access to host kernel features
- **Host namespaces**: `--pid=host`, `--ipc=host` for system-level visibility
- **Volume mounts**: Access to BPF, AppArmor, Docker socket, kernel security/debug interfaces
- **Port mapping**: 32767:32767 for KubeArmor gRPC API

The `make run` target in the Makefile shows the complete Docker run configuration.

### Service Definition Structure

The `horizon/service.definition.json` uses environment variable substitution:
- Variables like `$HZN_ORG_ID`, `$SERVICE_NAME` are replaced at publish time
- The `envsubst` command (used in `make check`) shows the resolved values
- User inputs (like `MY_TIME_ZONE`) can be overridden at deployment time

### No Custom Build Process

Unlike typical code projects:
- **No `make build`**: Uses pre-built third-party container from DockerHub
- **No `make push`**: No custom image to push to a registry
- **No source code compilation**: Pure configuration/deployment project

### Makefile Targets Organization

The Makefile follows a logical workflow:
1. **Local testing**: `init`, `run`, `dev`, `attach`, `test`, `stop`
2. **Publishing**: `publish-service`, `publish-service-policy`, `publish-deployment-policy`
3. **Node management**: `agent-run`, `agent-stop`
4. **Debugging**: `check`, `deploy-check`, `log`
5. **Cleanup**: `clean`, `distclean`

The `publish` target is a convenience wrapper that executes the full publishing workflow.

### Multi-Architecture Support

The service supports both x86_64 and arm64 architectures:
- Set `ARCH` environment variable before publishing
- Deployment policy uses `"arch": "*"` to match any architecture
- Same configuration works on both Intel/AMD and ARM-based edge nodes

## Important Notes for Agents

1. **This is NOT a source code project** - Do not look for application code, tests, or typical software development artifacts. This is a deployment configuration repository.

2. **Third-party dependency** - The actual KubeArmor software is maintained externally. This repo only configures how it deploys via Open Horizon.

3. **Privileged access required** - Any modifications must preserve the privileged container requirements and extensive volume mounts, or KubeArmor will not function.

4. **Policy constraints are critical** - The service will only deploy to nodes that satisfy BOTH `openhorizon.allowPrivileged == true` AND `purpose == security`. Changing these requires coordinated updates across all three policy files.

5. **Environment variable substitution** - When modifying JSON files, maintain the `$VARIABLE` syntax for values that should be configurable at deployment time.

6. **No direct KubeArmor configuration** - To configure KubeArmor's behavior (security policies, etc.), refer to the [KubeArmor documentation](https://open-horizon.github.io/docs/kubearmor-integration/docs/README/). This repository only handles the Open Horizon deployment wrapper.
