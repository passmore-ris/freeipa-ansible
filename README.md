# Ansible FreeIPA for Russian International School
This is a repository containing an ansible collection used to set up and configure FreeIPA for Russian International School

## Requirements
- The ansible user, as specified in ansible.cfg, is *root*. When distributing ssh keys, it must be assigned to the root user of all machines.
- The domain is **ipa.rischool.ru**
- The IPA server must be called ipa00.ipa.rischool.ru
- The NFS server must be named **nfs.ipa.rischool.ru**
- You must have the *freeipa.ansible_freeipa* and *community.general* collections installed from ansible galaxy.

### The *hosts.ini* File

### CSV data


## Setup
Here are instructions on how to set up the FreeIPA infrastructure using this repository.
### The IPA Server
To do this all in one command:
```
ansible-playbook install-server.yml && ansible-playbook groups.yml --tags default_groups && ansible-playbook automember_rules.yml
```
This consists of three parts:
1. First, the server must be installed. Simply run:
```
ansible-playbook install-server.yml
```
2. Set up the two default groups by running
```
ansible-playbook groups.yml --tags default_groups
```
3. Then set up the automember rules for these groups:
```
ansible-playbook automember_rules.yml
```

### The NFS Server
For now both the NFS server and workstations should all be grouped as IPA clients in the inventory.However, we will treat the NFS server differently.
1. First it must be installed as an IPA client with some directories and files added:
```
ansible-playbook setup_nfs_server.yml
```
2. Then we need to configure the automounts, so that users have roaming home directories.
```
ansible-playbook mount_homes.yml
```

#### IMPORTANT
**Remove the nfs server from the list of ipa clients in the hosts.ini file!** If it is reinstalled as an IPA client, the keytab will be reset, and it will be a pain to fix. Comment out the *nfsserver* subgroup in *ipaclients*.

## Users
To add users, we use a csv file found in the *data* folder. The *userclass* attribute should point to which default group they should go into, either staff or pupil. Users can be deleted by setting the '**state**' attribute to '*absent*'.
To add users, simply run
```
ansible-playbook users.yml
```
This playbook will add the users sequentially and then create a home directory for them. If there are a large number of users, this may take a while. After users have been added, their home directories will also be created on the NFS server.
For whatever reason, if it is desirable to update only the users without touching home directories, then tags can be used as follows:
```
ansible-playbook users.yml --tags user_setup
```
Likewise, if only home directories should be targeted, run
```
ansible-playbook users.yml --tags homes
```
### The IPA Clients
The clients are sorted into two types - staff workstations and pupils workstations. Each host should be placed into each subgroup in the *hosts.ini* file accordingly.
1. First, ensure that the NFS server is not part of the *ipaclients* group:
```
ansible ipaclients -m ping
```
Make sure that the nfs server is not pinged. If it is, remove it from the *ipaclients* group.
2. Then run the setup_client playbook:
```
ansible-playbook setup_client.yml
```
This will ensure the domain name is already part of the hostname and install a local ansible fact. The fact relates to whether the host is a pupil or staff workstation.
3. Then, ansible-pull must be executed. This will set up the workstations and allow them to be configured from a central repository:
```
ansible ipaclients -m command -a "ansible-pull -U https://github.com/passmore-ris/ansible_pull_config.git -d /opt/ansible" -b
```
This may take a long time, so be patient.
Configuration of the worktations should be done from [this repository](https://github.com/passmore-ris/ansible_pull_config.git). Ad-hoc commands may still be run from the ansible controller.
