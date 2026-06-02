# service-kubearmor-security
![](https://img.shields.io/github/license/open-horizon-services/service-kubearmor-security) ![](https://img.shields.io/badge/architecture-x86,_arm64-green) ![Contributors](https://img.shields.io/github/contributors/open-horizon-services/service-kubearmor-security.svg)

This is an Open Horizon configuration to deploy a vanilla instance of the open-source [KubeArmor](https://kubearmor.io/) software.

## Prerequisites

**Management Hub:** [Install the Open Horizon Management Hub](https://open-horizon.github.io/quick-start) or have access to an existing hub in order to publish this service and register your edge node.  You may also choose to use a downstream commercial distribution based on Open Horizon, such as IBM's Edge Application Manager.  If you'd like to use the Open Horizon community hub, you may [apply for a temporary account](https://wiki.lfedge.org/display/LE/Open+Horizon+Management+Hub+Developer+Instance) and have credentials sent to you.

**Edge Node:** You will need an x86 computer running Linux, or a Raspberry Pi computer (arm64) running Raspberry Pi OS or Ubuntu to install and use KubeArmor deployed by Open Horizon.  You will need to install the Open Horizon agent software, anax, on the edge node and register it with a hub.

**Optional utilities to install:**  With `brew` on macOS (you may need to install _that_ as well), `apt-get` on Ubuntu or Raspberry Pi OS, `yum` on Fedora, install `gcc`, `make`, `git`, `jq`, `curl`, `net-tools`.  Not all of those may exist on all platforms, and some may already be installed.  But reflexively installing those has proven helpful in having the right tools available when you need them.

## Installation

Clone the `service-kubearmor-security` GitHub repo from a terminal prompt on the edge node and enter the folder where the artifacts were copied.

  NOTE: This assumes that `git` has been installed on the edge node.

  ``` shell
  git clone https://github.com/open-horizon-services/service-kubearmor-security.git
  cd service-kubearmor-security
  ```

Run `make clean` to confirm that the "make" utility is installed and working.

Confirm that you have the Open Horizon agent installed by using the CLI to check the version:

  ``` shell
  hzn version
  ```

  It should return values for both the CLI and the Agent (actual version numbers may vary from those shown):

  ``` text
  Horizon CLI version: 2.30.0-744
  Horizon Agent version: 2.30.0-744
  ```

  If it returns "Command not found", then the Open Horizon agent is not installed.

  If it returns a version for the CLI but not the agent, then the agent is installed but not running.  You may run it with `systemctl horizon start` on Linux or `horizon-container start` on macOS.

Check that the agent is in an unconfigured state, and that it can communicate with a hub.  If you have the `jq` utility installed, run `hzn node list | jq '.configstate.state'` and check that the value returned is "unconfigured".  If not, running `make agent-stop` or `hzn unregister -f` will put the agent in an unconfigured state.  Run `hzn node list | jq '.configuration'` and check that the JSON returned shows values for the "exchange_version" property, as well as the "exchange_api" and "mms_api" properties showing URLs.  If those do not, then the agent is not configured to communicate with a hub.  If you do not have `jq` installed, run `hzn node list` and eyeball the sections mentioned above.

NOTE: If "exchange_version" is showing an empty value, you will not be able to publish and run the service.  The only fix found to this condition thus far is to re-install the agent using these instructions:

``` shell
hzn unregister -f # to ensure that the node is unregistered
systemctl horizon stop # for Linux, or "horizon-container stop" on macOS
export HZN_ORG_ID=myorg   # or whatever you customized it to
export HZN_EXCHANGE_USER_AUTH=admin:<admin-pw>   # use the pw deploy-mgmt-hub.sh displayed
export HZN_FSS_CSSURL=http://<mgmt-hub-ip>:9443/
curl -sSL https://github.com/open-horizon/anax/releases/latest/download/agent-install.sh | bash -s -- -i anax: -k css: -c css: -p IBM/pattern-ibm.helloworld -w '*' -T 120
```

## Usage

To check how to operate KubeArmor, please follow [this ref](https://open-horizon.github.io/docs/kubearmor-integration/docs/README/).

To create [the service definition](https://github.com/open-horizon/examples/blob/master/edge/services/helloworld/CreateService.md#build-publish-your-hw) and publish it to the hub, enter `make publish`. This will publish the service definition, service policy, and deployment policy to the hub.

To register your edge node and form an agreement to download and run KubeArmor, enter `make agent-run`. This will register the node with the hub and start a watch command showing agreement formation progress. When installation is complete and an agreement has been formed, exit the watch command with Control-C.

Alternatively, you can run both steps together:
```shell
make publish && make agent-run
```

### Docker Image Versions

This service uses the `stable` tag by default, which automatically tracks the latest stable release of KubeArmor. The `stable` tag currently points to version v1.7.3.

To use a different version, set the `DOCKER_IMAGE_VERSION` environment variable before publishing:

```shell
export DOCKER_IMAGE_VERSION=v1.7.3  # Pin to specific version
make publish
```

Available version options:
- `stable` - Latest stable release (recommended, auto-updates)
- `latest` - Most recent build (may include pre-release versions)
- `v1.7.3` - Specific version (for reproducibility)
- See [KubeArmor DockerHub](https://hub.docker.com/r/kubearmor/kubearmor/tags) for all available tags

## Advanced details

### Debugging

The Makefile includes several targets to assist you in inspecting what is happening to see if they match your expectations.  They include:

`make log` to see both the event logs and the service logs.

`make check` to see the values in your environment variables and how they compare to the default values.  It will also show the service definition file with those values filled in.

`make deploy-check` to see if the properties and constraints that you've configured match each other to potentially form an agreement.

`make test` to check if the KubeArmor service is responding (note: this target may need updating for the correct port).

`make attach` to connect to the running container and open a shell inside it.

### All Makefile targets

* `default` - init run
* `init` - create the docker volume for KubeArmor configuration
* `run` - manually run the kubearmor-init and kubearmor containers locally as a test
* `check` - view current settings
* `stop` - halt a locally-run container
* `dev` - manually run kubearmor locally and connect to a terminal in the container
* `attach` - connect to a terminal in the kubearmor container
* `test` - request the web UI from the terminal to confirm that it is running and available
* `clean` - remove the container image and docker volume
* `distclean` - clean (see above) AND unregister the node and remove the service files from the hub
* `build` - Not applicable (uses third-party pre-built image from DockerHub)
* `push` - Not applicable (uses third-party pre-built image from DockerHub)
* `publish-service` - Publish the service definition file to the hub in your organization
* `remove-service` - Remove the service definition file from the hub in your organization
* `publish-service-policy` - Publish the [service policy](https://github.com/open-horizon/examples/blob/master/edge/services/helloworld/PolicyRegister.md#service-policy) file to the hub in your org
* `remove-service-policy` - Remove the service policy file from the hub in your org
* `publish-deployment-policy` - Publish a [deployment policy](https://github.com/open-horizon/examples/blob/master/edge/services/helloworld/PolicyRegister.md#deployment-policy) for the service to the hub in your org
* `remove-deployment-policy` - Remove a deployment policy for the service from the hub in your org
* `agent-run` - register your agent's [node policy](https://github.com/open-horizon/examples/blob/master/edge/services/helloworld/PolicyRegister.md#node-policy) with the hub and watch for agreement formation
* `publish` - Publish the service definition, service policy, and deployment policy to the hub (does NOT register the agent - use `agent-run` separately)
* `agent-stop` - unregister your agent with the hub, halting all agreements and stopping containers
* `deploy-check` - confirm that a registered agent is compatible with the service and deployment
* `log` - check the agent event logs
