# Dynamic vs. static inventory
The first decision to make is when using the playbook in this repository to deploy Fusion is whether you would like to manage the inventory for your deployments using the dynamic inventory scripts that are provided in the [common-utils](../common-utils) submodule for AWS and OpenStack environments or whether you are managing your inventory statically. Since the static inventory use case is the simplest, we'll start our discussion there. Then we'll move on to a discussion of how to control the deployment of Fusion in both AWS and OpenStack environments by tagging the VMs in those environments so that the dynamic inventory scripts from the [common-utils](../common-utils) submodule can be used to dynamically construct the lists of nodes that should be targeted by a given `ansible-playbook` run.

## Managing deployments with static inventory files
Let's assume that we are planning the deployment of a three-node Fusion cluster using the playbook in this repository and we are planning on using a [static inventory file](https://docs.ansible.com/ansible/intro_inventory.html) to control that deployment. In this example (and others like it in this document) we'll assume that we're going to construct a single static inventory file (an INI-like formatted file containing the list of hosts that are targeted by any given playbook run), and then pass that file into the `ansible-playbook` command using the `-i, --inventory-file` command-line flag.

In our discussion of the various deployment scenarios supported by this playbook, we show that deployments of a Fusion cluster actually require an associated (assumed to be external) Zookeeper ensemble. For purposes of this discussion, we will focus only on the inventory information associated with the Fusion nodes, not on the inventory information associated with the Zookeeper ensemble. So, returning to our example, the inventory file associated with our three-node Fusion cluster might look something like this:

```bash
$ cat test-cluster-inventory
# example inventory file for a clustered deployment

192.168.34.28 ansible_ssh_host=192.168.34.28 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/solr_cluster_private_key'
192.168.34.29 ansible_ssh_host=192.168.34.29 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/solr_cluster_private_key'
192.168.34.30 ansible_ssh_host=192.168.34.30 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/solr_cluster_private_key'
192.168.34.18 ansible_ssh_host=192.168.34.18 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.19 ansible_ssh_host=192.168.34.19 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.20 ansible_ssh_host=192.168.34.20 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'

[solr]
192.168.34.28
192.168.34.29
192.168.34.30

[zookeeper]
192.168.34.18
192.168.34.19
192.168.34.20

$
```
As you can see, our static inventory file consists of a list of the hosts that are going to be accessed during our deployment and a two host groups (the `solr` and `zookeeper` host groups in this case). For each host in this file, we provide a list of the parameters that Ansible will need to connect to that host (INI-file style) as a set of `name=value` pairs. In this example, we've defined the following values for each of the entries in our static inventory file:

* **`ansible_ssh_host`**: the hostname/address that Ansible should use to connect to that host; if not specified, the same hostname/address listed at the start of the line for that entry for that host will be used (in this example we are using IP addresses). This parameter can be important when there are multiple network interfaces on each host and only one of them (an admin network, for example) allows for SSH access
* **`ansible_ssh_port`**: the port that Ansible should use when connecting to the host via SSH; if not specified, Ansible will attempt to connect using the default SSH port (port 22)
* **`ansible_ssh_user`**: the username that Ansible should use when connecting to the host via SSH; if not specified, then the value of the global `ansible_user` variable will be used (this variable can be set as an extra variable in the `ansible-playbook` command if the same username is used for all hosts targeted by the playbook run)
* **`ansible_ssh_private_key_file`**: the private key file that should be used when connecting to the host via SSH; if not specified, then the value of the global `ansible_ssh_private_key_file` variable will be used instead (this variable can be set as an extra variable in the `ansible-playbook` command if the same private key is used for all hosts targeted by the playbook run)

The hosts in the `solr` host group are the target nodes for our deployment; the hosts in the `zookeeper` host group are assumed to be members of a pre-existing Zookeeper ensemble, and those nodes will be accessed by Ansible during the playook run in order to to gather that facts about those nodes that we need to configure the target nodes as a cluster that will communicate through that ensemble.

With this static inventory file built, it's a relatively simple matter to deploy our Fusion cluster using a single `ansible-playbook` command. Examples of these commands are shown [here](Deployment-Scenarios.md).

## Managing deployments using dynamic inventory scripts
For both AWS and OpenStack environments, the playbooks in this repository support the use of a [dynamic inventory](https://docs.ansible.com/ansible/intro_dynamic_inventory.html) to control deployments to those environments. This is accomplished by making use of the [build-app-host-groups](../common-roles/build-app-host-groups) common role, which builds the host groups that are needed for the playbook run by filtering the hosts in the AWS or OpenStack environment, based on the tags that are assigned them and the tags that were included in the `ansible-playbook` command. This provides an attractive alternative to building static inventory files to control the deployment process in these environments, since the meta-data needed to determine which nodes should be targeted by a given playbook run is readily available in the framework itself.

The process of using the [build-app-host-groups](../common-roles/build-app-host-groups) common role to control a playbook run starts out by tagging the VMs that are going to be the target of a particular deployment with the following tags:

* **Tenant**: the tenant that will be using the Fusion cluster being deployed. As an example, you might use a value of `labs` for this tag for Fusion clusters that the Labs department in your organization will be using)
* **Project**: this tag will be given the value of the project that will be using the Fusion cluster that is being deployed; for example we might use the value `projectx` for this tag for clusters that will be used for ProjectX
* **Domain**: this tag is used to separate out the various domains that might be defined by the project team to control their deployments. For example, there might be separate clusters built for `development`, `test`, `preprod`, and `production`. Each of these environments would be tagged appropriately based on their intended use.
* **Application**: this tag is used to identify the application that should be deployed to a given virtual machine; in this playbook only nodes that are tagged with the application `solr` are targeted by default (and although the application name is something you could customize, we recommend against doing so)

It should be noted here that this playbook assumes that an associated Zookeeper ensemble has already been deployed that can be associated with the Fusion cluster that the playbook run is deploying. In the dynamic inventory use case, the nodes that make up this Zookeeper ensemble must be tagged with the same `Tenant`, `Project`, and `Domain` tag values that are used to build the associated Fusion cluster. Obviously, these Zookeeper instances would have been tagged with an `Application` tag of `zookeeper`, but if the remaining tags do not match then the playbook will not be able to find the associated Zookeeper ensemble and the Fusion cluster nodes will fail to start.

Once these tags have been assigned to the VMs that will be targeted by a given Fusion deployment, it is a relatively simple process to define values for the tags that should be targeted by the `ansible-playbook` command (as extra variables or in a *local variables file*, for example). Specifically, the following variables must be defined when using this dynamic inventory to manage a playbook run:

* **tenant**
* **project**
* **domain**
* **application**

Note that the `application` value defaults to `solr` (the default value defined at the top of the [vars/solr.yml](../vars/solr.yml) file), so this value does not really need to be redefined as part of the `ansible-playbook` command or in a or in a *local variables file*, but it is shown here for completeness.

With these values assigned as part of the `ansible-playbook` command or in a or in a *local variables file*, the [build-app-host-groups](../common-roles/build-app-host-groups) role that is used by this playbook can correctly identify the nodes in the AWS or OpenStack environment that are targeted by a given playbook run and then use the resulting (dynamically constructed) lists of Fusion and Zookeeper nodes to control the deployment of Fusion to the nodes that will make up the cluster being built.
