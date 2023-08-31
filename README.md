# Execution Environment

## Links

[Creating and Consuming Execution Environments](https://access.redhat.com/documentation/en-us/red_hat_ansible_automation_platform/2.4/html-single/creating_and_consuming_execution_environments/index)


## Execution Environment

Automation execution environments are container images on which all automation in Red Hat Ansible Automation Platform is run.

Execution environment is expected to contain the following:
- Ansible 2.9 or Ansible Core 2.x
- Python 3.8-3.10
- Ansible Runner
- Ansible content collections
- Collection, Python, or system dependencies


## ansible-builder CLI

Ansible Builder is a tool that aids in the creation of Ansible Execution Environments (EE).

EE are defined in a YAML file called Execution Environment Definition.

Execution Environment Definition file supports multiple versions:
- Version 1: Supported by **all** `ansible-builder` versions.
- Version 2: Supported by `ansible-builder` versions **1.2** and **later**.
- Version 3: Supported by `ansible-builder` versions **3.0** and **later**.


In a nutshell, Ansible Builder will:

1. Read and validate the definition file
2. Create the Containerfile file
3. Pass the Containerfile to podman/docker to build the container image


## `infra.ee_utilities.ee_builder` Ansible Role

[infra.ee_utilities.ee_builder](https://github.com/redhat-cop/ee_utilities/tree/devel/roles/ee_builder) is an **Ansible role** developed and maintained by the Red Hat CoP (Community of Practice) that can be used to **build** and optionally **push** EE to a private automation hub (PAH) or a container registry.
 

## Demo / Labs

1. EE building using `ansible-builder` 
2. EE building using `infra.ee_utilities.ee_builder`
3. CICD pipelines
     - Option 1: Jenkins + `ansible-builder`
     - Option 2: Jenkins + `infra.ee_utilities.ee_builder`
     - Option 3: AAP Workflow + Playbook using `infra.ee_utilities.ee_builder`


### EE building using `ansible-builder`

**Use Case**: Build an EE called **ansible-builder-ee** that can be used in other tools (ansible-navigator, Jenkins, AAP, ect.) to build EE.

The EE should:
- Be based on ansible-automation-platform-24/ee-minimal-rhel8:latest
- Contain ansible-builder
- Contain podman
- Contain containers.podman collection
- Contain infra.ee_utilities collection

#### Steps

1. Create bindep.txt (System Packages)

```sh
cat << EOF > bindep.txt
podman
EOF
```

1. Create requirements.txt (Python Modules in pip freeze format)

```sh
cat << EOF > requirements.txt
ansible-builder
EOF
```

3. Create requirements.yml (Ansible Collections)

```sh
cat << EOF > requirements.yml
# A requirements.yml file for Galaxy
collections:
  - containers.podman
  - infra.ee_utilities
EOF
```

4. Create execution-environment.yml

```yaml
cat << EOF > execution-environment.yml
version: 1

# Build arguments
build_arg_defaults:
  # arguments to ansible-galaxy CLI e.g. -c to skip SSL verification
  ANSIBLE_GALAXY_CLI_COLLECTION_OPTS: "-v"
  EE_BASE_IMAGE: registry.redhat.io/ansible-automation-platform-24/ee-minimal-rhel8:latest

# Path to ansible.cfg relative to the definition file.
# THIS IS NEEDED TO RETRIEVE COLLECTION FROM A PRIVATE AUTOMATION HUB (PAH)
ansible_config: "./ansible.cfg"

# Requirements files
dependencies:
  galaxy: requirements.yml
  python: requirements.txt
  system: bindep.txt

# Command for custom build steps
additional_build_steps:
  prepend: |
    RUN whoami
    RUN cat /etc/os-release
  append:
    - RUN echo This is a post-install command!
EOF
```

5. Build the EE

```sh
### prep
podman login registry.redhat.io
podman pull registry.redhat.io/ansible-automation-platform-24/ee-minimal-rhel8:latest

### build the EE
ansible-builder build -t ansible-builder-ee
```

6. Push the EE to PAH

```sh
### Tag the EE
podman tag localhost/ansible-builder-ee:latest pah.lan/ansible-builder-ee

### Pushing to PAH
podman login pah.lan --tls-verify=false
podman push pah.lan/ansible-builder-ee:latest --tls-verify=false
```


### EE building using `infra.ee_utilities.ee_builder`

**Use case**: Build an EE using `infra.ee_utilities.ee_builder` role

#### Steps

1. Create the vars file

See [group_vars/all.yml](./infra.ee_utilities/group_vars/all.yml)

2. Create the playbook

See [build-ee_playbook.yml](./infra.ee_utilities/build-ee_playbook.yml)

3. Run the playbook to build and push the EE to PAH

```sh
cd ee/infra.ee_utilities
ansible-navigator run build-ee_playbook.yml -m stdout
```


### CICD pipelines: Jenkins + `ansible-builder`

- Use docker agent and `ansible-builder-ee` container image built previously
- Login to Red Hat registry
- Pull `registry.redhat.io/ansible-automation-platform-24/ee-minimal-rhel8:latest` image
- Builder custom EE using `ansible-builder` 
- Push custom EE to Private automation hub

More details in [Jenkinsfile](../Jenkinsfile)


### CICD pipelines: Jenkins + `infra.ee_utilities.ee_builder`

Idea here is to use `infra.ee_utilities.ee_builder` ansible role instead of `ansible-builder` to build and push the EE to PAH.


### CICD pipelines: AAP Workflow / Job Template + Playbook using `infra.ee_utilities.ee_builder`

#### Idea

- Import `ansible-builder-ee` in AAP
- Build a playbook using `infra.ee_utilities.ee_builder` ansible role
- Create an AAP Project and Workflow / JT using the playbook
- Workflow / JT and be triggered via Webhooks or API

Issues:
- Current EE `ansible-builder-ee` needs to be run with `--privileged` flag. NEED TO FIGURE OUT HOW TO ACHIEVE THIS IN AAP


---

# APPENDIX

## Example of Execution Environment Definition version 3

```yaml
---
version: 3

build_arg_defaults:
  ANSIBLE_GALAXY_CLI_COLLECTION_OPTS: -vv --pre

options:
  package_manager_path: /usr/bin/microdnf

dependencies:
  galaxy:
      collections:
      -   name: containers.podman
          source: https://galaxy.ansible.com
          version: 1.10.3
      -   name: infra.ee_utilities
          source: https://galaxy.ansible.com
          version: 3.1.2
      - ssmalick.test_collection
  python:
  - boto>=2.49.0
  - botocore>=1.12.249
  - pytz
  - ansible-builder
  system:
  - podman

additional_build_files:
  - src: ansible.cfg
    dest: configs
additional_build_steps:
  prepend_galaxy:
    - ADD _build/configs/ansible.cfg /etc/ansible/ansible.cfg
  prepend_final:
    - RUN whoami
    - RUN cat /etc/os-release
  append_final:
    - RUN echo This is a post-install command!
images:
  base_image:
    name: registry.redhat.io/ansible-automation-platform-24/ee-supported-rhel8:latest
```