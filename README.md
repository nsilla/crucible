# Crucible: OpenShift 4 Management Cluster Seed Playbooks

> ❗ _Red Hat does not provide commercial support for the content of this repo. Any assistance is purely on a best-effort basis, as resource permits._

```bash
##############################################################################
DISCLAIMER: THE CONTENT OF THIS REPO IS EXPERIMENTAL AND PROVIDED "AS-IS"

THE CONTENT IS PROVIDED AS REFERENCE WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
##############################################################################
```

This repository contains playbooks for automating the creation of an OpenShift Container Platform cluster on premise using the Developer Preview version of the OpenShift Assisted Installer. The playbooks require only minimal infrastructure configuration and do not require any pre-existing cluster. Virtual and Bare Metal deployments have been tested in restricted network environments where nodes do not have direct access to the Internet.

These playbooks are intended to be run from a `bastion` host, running a subscribed installation of RHEL 8.4, inside the target environment. Pre-requisites can be installed manually or automatically, as appropriate.

See [how the playbooks are intended to be run](docs/connecting_to_hosts.md) and understand [what steps the playbooks take](docs/pipeline_into_the_details.md).

## OpenShift Versions Tested

- 4.6
- 4.7
- 4.8

## Assisted Installer versions Tested

- v1.0.18.3
- v1.0.19.3
- v1.0.20.3
- v1.0.21.3
- v1.0.22.3
- v1.0.23.2
- v1.0.24.2

### Dependencies

Requires the following to be installed on the deployment host:

- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-ansible-on-specific-operating-systems)
- [netaddr](https://github.com/netaddr/netaddr)
- [skopeo](https://github.com/containers/skopeo)

```bash
dnf install ansible python3-netaddr skopeo
```

There's also some required Ansible modules that can be installed with the following command:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Before Running The Playbook

- Configure NTP time sync on the BMCs and confirm the system clock among the master nodes is synchronized within a second. The installation fails when system time does not match among nodes because etcd database will not be able to converge.
- Modify the provided inventory file `inventory.yml.sample`. Fill in the appropriate values that suit your environment and deployment requirements. See the sample file and [docs/inventory.md](docs/inventory.md) for more details.
- Modify the provided inventory vault file `inventory.vault.yml.sample`. Fill in the corresponding secret values according to the configuration of the inventory file. See the sample file and [docs/inventory.md#required-secrets](docs/inventory.md#required-secrets) for more details.
- Place the following prerequisites in this directory:
  - OpenShift pull secret stored as `pull-secret.txt` (can be downloaded from [here](https://console.redhat.com/openshift/install/metal/installer-provisioned))
  - SSH Public Key stored as `ssh_public_key.pub`
  - If `deploy_prerequisites.yml` is NOT being used; SSL self-signed certificate stored as `mirror_certificate.txt`

### Inventory Vault File Management

The inventory vault files should be encrypted and protected at all times, as they may contain secret values and sensitive information. 

To encrypt a vault file named `inventory.vault.yml`, issue the following command.

```bash
ansible-vault encrypt inventory.vault.yml 
```

An encrypted vault file can be referenced when executing the playbooks with the `ansible-playbook` command.  
To that end, provide the option `-e "@{PATH_TO_THE_VAULT_FILE}"`.

To allow Ansible to read values from an encrypted vault file, a password for decrypting the vault must be provided. Provide the `--ask-vault-pass` flag to force Ansible to ask for a password to the vault before the selected playbook is executed.

A complete command to execute a playbook that takes advantage of both options can look like this:
```bash
ansible-playbook -i inventory ${SELECTED_PLAYBOOK} -e "@inventory.vault.yml" --ask-vault-pass
```

If a need arises to decrypt an encrypted vault file, issue the following command.

```bash
ansible-vault decrypt inventory.vault.yml
```

For more information on working with vault files, see the [Ansible Vault documentation](https://docs.ansible.com/ansible/latest/user_guide/vault.html#encrypting-content-with-ansible-vault).

### Pre-Deployment Validation

Some utility playbooks are provided to perform some validation before attempting a deployment:

```bash
ansible-playbook -i inventory prereq_facts_check.yml -e "@inventory.vault.yml" --ask-vault-pass
ansible-playbook -i inventory playbooks/validate_inventory.yml -e "@inventory.vault.yml" --ask-vault-pass
```

## Running The Playbooks

There are a few main playbooks provided in this repository:

- `deploy_prerequisites.yml`: sets up the services required by Assisted Installer, and an Assisted Installer configured to use them.
- `deploy_cluster.yml`: uses Assisted Installed to deploy a cluster
- `post_install.yml`: fetches the `kubeconfig` for the deployed cluster and places it on the bastion host.
- `site.yml` simply runs all three in order.

Each of the playbooks requires only an inventory, and can be run like this:

```bash
ansible-playbook -i inventory site.yml -e "@inventory.vault.yml" --ask-vault-pass
```

## Prerequisite Services

Crucible can automatically set up the services required to deploy and run a cluster. Some are required for the Assisted Installer tool to run, and some are needed for the resulting cluster.

- NTP - The NTP service helps to ensure clocks are synchronised across the resulting cluster which is a requirement for the cluster to function.
- Container Registry Local Mirror - Provides a local container registry within the target environment. The Crucible playbooks automatically populates the registry with required images for cluster installation. The registry will continue to be used by the resulting cluster.
- HTTP Store - Used to serve the Assisted Installer discovery ISO and allow it to be used as Virtual Media for nodes to boot from.
- DNS - Optionally set up DNS records for the required cluster endpoints, and nodes. If not automatically set up then the existing configuration will be validated.
- Assisted Installer - A pod running the Assisted Installer service, database store and UI. It will be configured for the target environment and is used by the cluster deployment playbook to coordinate the cluster deployment.

While setup of each of these can be disabled if you wish to manually configure them, but it's highly recommended to use the automatic setup of all prerequisites.

## Outputs

Note that the exact changes made depend on which playbooks or roles are run, and the specific configuration.

### Cluster

The obvious output from these playbooks is a clean OCP cluster with minimal extra configuration. Each node that has been added to the resulting cluster will have:

- CoreOS installed and configured
- The configured SSH public key as an authorised key for `root` to allow debugging

### Prerequisite Services

Various setup is done on the prerequisite services. These are informational and are not needed unless you encounter issues with deployment.
The following are defaults for a full setup:

- Registry Host

  - `opt/registry` contains the files for the registry, including the certificates.
  - `tmp/wip` is used during the playbook execution as a temporary file store.

- DNS Host

  - Using dnsmasq: `/etc/dnsmasq.d/dnsmasq.<clustername>.conf`
  - using Network Manager: `/etc/NetworkManager/dnsmasq.d/dnsmasq.<clustername>.conf` and `/etc/NetworkManager/conf.d/dnsmasq.conf`

- Assisted Installer

  - A running pod containing the Assisted Installer service.
  - `/opt/assisted-installer` contains all the files used by the Assisted Installer container

- HTTP Store
  - A running pod containing the `httpd` service
  - The discovery image from Assisted Installer will be placed in and served from `/opt/http_store/data`

### Bastion

As well as deploying prerequisites and a cluster, the playbooks create or update various local artifacts in the repository root and the `fetched/` directory (configured with `fetched_dest` var in the inventory).

- An updated `pull-secret.txt` containing an additional secret to authenticate with the deployed registry.
- The self-signed certificate created for the registry host as `domain.crt`.
- The SSH public and private keys generated for access to the nodes, if any, at `/home/redhat/ssh_keys` (temporarily stored in `/tmp/ssh_key_pair`)
- Any created CoreOS ignition files.

When doing multiple runs ensure you retain any authentication artefacts you need between deploys.

## Testing

Existing tests can be run using

```bash
ansible-playbook tests/run_tests.yml
```

## Related Documentation

### General

- [How the playbooks are intended to be run](docs/connecting_to_hosts.md)
- [How to configure the inventory file](docs/inventory.md)
- [Steps the playbooks take when executed](docs/pipeline_into_the_details.md)

### Troubleshooting

Some useful help for troubleshooting if you find any issues can be found in [docs/troubleshooting](docs/troubleshooting)

- [Discovery ISO not booting](docs/troubleshooting/discovery_iso_not_booting.md)

## References

This software was adapted from [sonofspike/cluster_mgnt_roles](https://github.com/sonofspike/cluster_mgnt_roles)