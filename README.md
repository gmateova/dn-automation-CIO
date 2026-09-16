# DN CIO VM Automation

Idempotent Ansible automation for provisioning and reconciling Windows and Linux virtual machines in VMware vSphere, including PHPIPAM address reservation.

## Why Ansible instead of Terraform here

This repository deliberately starts with Ansible.

Terraform does **not** automatically modify VMware VMs merely because they already exist. Terraform only manages resources recorded in its state. Existing VMs would therefore have to be imported into Terraform state before Terraform could safely manage them. A Terraform plan containing only newly declared VMs would not touch unrelated existing VMs, but adopting an existing VMware estate into Terraform means deciding which existing objects are authoritative in Terraform, importing them, and reconciling configuration drift.

For the requested workflow -- create a VM from a template, reconcile CPU/RAM/disks, and reserve an address in PHPIPAM while coexisting with existing manually-created VMware VMs -- Ansible has the lower adoption risk. Existing VMs stay outside an IaC state database, while every task in this repository first discovers/reconciles the desired state.

Terraform can be introduced later for greenfield infrastructure or after an explicit import/adoption project.

## Scope

- clone a new VM from a VMware template
- separate Windows and Linux entry points
- template selected from variables
- desired CPU and RAM from variables
- additional disks with variable sizes
- request/reserve an IP address in PHPIPAM
- optionally use a preselected IP address
- idempotent reruns
- no credentials committed to Git

## Repository layout

```text
.
├── ansible.cfg
├── collections/
│   └── requirements.yml
├── inventory/
│   └── hosts.yml
├── playbooks/
│   ├── linux.yml
│   └── windows.yml
├── roles/
│   ├── phpipam_ip/
│   │   └── tasks/main.yml
│   └── vmware_vm/
│       └── tasks/main.yml
├── vars/
│   ├── common.yml
│   ├── linux.yml
│   └── windows.yml
└── examples/
    └── vm-request.yml
```

## Required secrets

Export credentials in the execution environment (AWX/AAP credentials, Vault, or shell environment). Do not put them in `vars/`.

```bash
export VMWARE_HOST='vcenter.example.internal'
export VMWARE_USER='svc-vmware-automation@example.internal'
export VMWARE_PASSWORD='...'

export PHPIPAM_URL='https://phpipam.example.internal'
export PHPIPAM_APP_ID='ansible'
export PHPIPAM_USER='svc-phpipam-automation'
export PHPIPAM_PASSWORD='...'
```

`PHPIPAM_URL`, service account and app ID can also be supplied through AWX/AAP credential/environment injection. Passwords remain secrets and must not be committed.

## Define the environment

Edit `vars/common.yml` with the VMware placement and PHPIPAM subnet IDs. Edit `vars/linux.yml` and `vars/windows.yml` with the appropriate templates.

A VM request is passed separately, for example:

```yaml
vm_name: dn-test-app01
vm_os: linux
vm_cpu: 4
vm_memory_mb: 8192
vm_disks:
  - size_gb: 100
    datastore: datastore01
    type: thin
phpipam_subnet_id: 123
vm_ip_address: null
vm_ip_hostname: dn-test-app01.example.internal
vm_ip_description: DN CIO automation - dn-test-app01
```

If `vm_ip_address` is `null`, PHPIPAM is asked for the first free address in `phpipam_subnet_id`. If an IP is supplied, the role verifies/reserves that exact address.

## Run

Linux:

```bash
ansible-playbook playbooks/linux.yml -e @examples/vm-request.yml
```

Windows:

```bash
ansible-playbook playbooks/windows.yml -e @examples/vm-request.yml
```

## Idempotency model

The PHPIPAM role searches for an existing reservation by hostname before allocating anything. It therefore reuses the previous address on subsequent runs. If an explicit address is requested, it checks the address before creating it.

The VMware role uses the vSphere VM module declaratively: an existing VM with the requested name is reconciled instead of cloned again. CPU, RAM and declared disks are expressed as desired state. The role does not delete arbitrary VMs.

Disk shrinking is intentionally rejected. Growing a virtual disk is safe from the VMware side, but growing the partition/filesystem inside the guest is a separate OS configuration operation and is not performed by this provisioning role.

## Important boundary

This automation reserves an address in PHPIPAM and creates/reconciles the VM in VMware. Applying that address inside the guest OS depends on how the DN Windows/Linux templates are customized (VMware guest customization, DHCP reservation, cloud-init, sysprep, etc.). The repository keeps the IP reservation independent until the exact DN template customization method is confirmed.
