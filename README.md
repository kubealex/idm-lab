# IDM Demo env installer

This is a simple demo environment provisioner for IdM to install.
It creates:
- libvirt-network with DHCP/DNS
- libvirt-pool for your VMS
- idm-server.<your-domain> VM with IdM installed and configured with DNS/adtrust
- idm-client.<your-domain> VM with IdM configured to work as a client to idm-server

### Install Ansible

You need to follow the instructions in [Ansible Website](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-the-ansible-community-package) to proceed and install Ansible on your machine.

## Needed variables

In order to work, the playbooks need some basic variables in the **lab_vars.yml** file:

| Variable | Value | Description |
|--|--|--|
| **network_cidr** | Defaults to 192.168.210.0/24 | The subnet that is assigned to libvirt network |
| **offline_token** | No default | [Offline Token](https://access.redhat.com/management/api) for images/packages download from Red Hat Portal |
| **rhsm_user** | No default | The Red Hat Account username |
| **rhsm_password** | No default | The Red Hat Account username |
| **rhsm_pool_id** | No default | The pool ID of the subscription covering the product [in subscription manager](https://access.redhat.com/management/subscriptions/) |

## Lab provisioning

The provisioner consists of two playbooks, that configure the underlying components (VM, network) and prepare the guests to install Satellite.

The first playbook is **provision-lab.yml** which takes care of creating KVM resources.

The package comes with an inventory:

    localhost ansible_connection=local
    [ipaserver]
    idm-server.idmdemo.labs ansible_user=sysadmin ansible_password=redhat

    [ipaclients]
    idm-client.idmdemo.labs ansible_user=sysadmin ansible_password=redhat

The playbook can download RHEL 9 image, or work with pre-downloaded images. The only requirement is that the images need to be placed in the playbook directory with the name and **rhel9.iso**

To download the images via the playbook, you will need your [Offline Token](https://access.redhat.com/management/api).

**IMPORTANT** If you don't want to download images (it's around 20GB), just leave the variable blank.

Since some modules rely on additional collections you will need to install them via:

    ansible-galaxy install -r requirements.yml

Once you set the *network_cidr* variable to the desired value, you can run the playbook:

    ansible-playbook -i inventory provision-lab.yml

It takes around 20-25 minutes to be up and running. If you experience last step of the playbook being hanging after the machines are completely installed, **relaunch** the playbook as sometimes the ping module gets stuck.

## IdM setup

Once your VMs are up and running I prepared an execution-environment to use with Ansible, to use freeipa.ansible_freeipa collection to provision both server and client.

From the *idm* folder, build your execution-environment:

    ansible-builder build -c ansible/execution-environment/context -f ansible/execution-environment/execution-environment.yml -t ansible-execution-env

Edit *ansible/inventory* file if you need fine tuning on attributes (i.e. if you changed the domain):

    [ipaserver]
    idm-server.rhdemo.labs

    [ipaclients]
    idm-client.rhdemo.labs

    [ipaservers]
    idm-server.rhdemo.labs

    [ipaserver:vars]
    ipaserver_domain=rhdemo.labs
    ipaserver_realm=RHDEMO.LABS
    ipaserver_setup_dns=yes
    ipaserver_setup_adtrust=yes
    ipaserver_auto_forwarders=yes
    ipaadmin_password=admin123
    ipadm_password=admin123

    [ipaclients:vars]
    ipaclient_domain=rhdemo.labs
    ipaclient_realm=RHDEMO.LABS
    ipaserver_domain=rhdemo.labs
    ipaserver_realm=RHDEMO.LABS
    ipaadmin_principal=admin
    ipaadmin_password=admin123
    ipassd_enable_dns_updates=true

Then launch the playbook to install **idm-server**:

    ansible-navigator run -m stdout --eei=ansible-execution-env --pp never --pae false -i ansible/inventory ansible/idm-server-setup.yml

If you want to configure the client to connect to idm-server launch the playbook to setup **idm-client**:

    ansible-navigator run -m stdout --eei=ansible-execution-env --pp never --pae false -i ansible/inventory ansible/idm-client-setup.yml


## Test your configuration

If the setup was good, you will be able to access your IdM server on https://idm-server.<your-domain>