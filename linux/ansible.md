https://www.linode.com/docs/applications/configuration-management/getting-started-with-ansible

## Install and Set Up Miniconda

With Miniconda, it’s possible to create a virtualized environment for Ansible which can help to streamline the installation process for most Distros and environments that require multiple versions of Python. Your control node will require Python version 2.7 or higher to run Ansible.

1. Download and install Miniconda:

   ```
   curl -OL https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh
   bash Miniconda3-latest-Linux-x86_64.sh
   ```

2. You will be prompted several times during the installation process. Review the terms and conditions and select “yes” for each prompt.

3. Restart your shell session for the changes to your PATH to take effect.

   ```
   exec bash -l
   ```

4. Create a new virtual environment for Ansible:

   ```
   conda create -n ansible-dev python=3
   ```

5. Activate the new environment:

   ```
   conda activate ansible-dev
   ```

6. Check your Python version:

   ```
   python --version
   ```

## Connect to your Managed Nodes

Use the `all` directive to ping all servers in your inventory. By default, Ansible will use your local user account’s name to connect to your nodes via SSH. You can override the default behavior by passing the `-u` option, plus the desired username. Since there are no standard user accounts on the nodes, in the example, you run the command as the root user.

```
ansible all -u root -m ping
```

https://www.linode.com/docs/applications/configuration-management/running-ansible-playbooks/

## Disable Host Key Checking

Open the `/etc/ansible/ansible.cfg` configuration file in a text editor of your choice, uncomment the following line, and save your changes.

```ini
#host_key_checking = False
```

## Create the Limited User Account Playbook

You are now ready to create the Limited User Account Playbook. This Playbook will create a new user on your Linode, add your Ansible control node’s SSH public key to the Linode, and add the new user to the Linode’s `sudoers` file.

- `yourusername` with the user name you would like to create on the Linode
- `$6$rounds=656000$W.dSl` with the password hash you create in the [Create a Password Hash](https://www.linode.com/docs/applications/configuration-management/running-ansible-playbooks/#create-a-password-has) section of the guide.

```yaml
---
- hosts: webserver
  remote_user: root
  vars:
    NORMAL_USER_NAME: 'yourusername'
  tasks:
    - name: "Create a secondary, non-root user"
      user: name={{ NORMAL_USER_NAME }}
            password='$6$rounds=656000$W.dSl'
            shell=/bin/bash
    - name: Add remote authorized key to allow future passwordless logins
      authorized_key: user={{ NORMAL_USER_NAME }} key="{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
    - name: Add normal user to sudoers
      lineinfile: dest=/etc/sudoers
                  regexp="{{ NORMAL_USER_NAME }} ALL"
                  line="{{ NORMAL_USER_NAME }} ALL=(ALL) ALL"
                  state=present
```

## Configure the Base System

This next Playbook will take care of some common server setup tasks, such as setting the timezone, updating the hosts file, and updating packages.

- `yourusername` with the username you created in the [Create the Limited User Account Playbook](https://www.linode.com/docs/applications/configuration-management/running-ansible-playbooks/#create-the-limited-user-account-playbook) section of the guide
- `web01` with the hostname you would like to set for your Linode.
- If you have a domain name you would like to set up, replace `www.example.com` with it.

```yaml
---
- hosts: webserver
  remote_user: yourusername
  become: yes
  become_method: sudo
  vars:
    LOCAL_HOSTNAME: 'web01'
    LOCAL_FQDN_NAME: 'www.example.com'
  tasks:
    - name: Set the timezone for the server to be UTC
      command: ln -sf /usr/share/zoneinfo/UTC /etc/localtime
    - name: Set up a unique hostname
      hostname: name={{ LOCAL_HOSTNAME }}
    - name: Add the server's domain to the hosts file
      lineinfile: dest=/etc/hosts
                  regexp='.*{{ item }}$'
                  line="{{ hostvars[item].ansible_default_ipv4.address }} {{ LOCAL_FQDN_NAME }} {{ LOCAL_HOSTNAME }}"
                  state=present
      when: hostvars[item].ansible_default_ipv4.address is defined
      with_items: "{{ groups['linode'] }}"
    - name: Update packages
      apt: update_cache=yes upgrade=dist
```

## fetch with_fileglob

All of the `with_*` looping mechanisms are local lookups unfortunately so there's no really clean way to do this in Ansible.

What you can do is generate your fileglob by shelling out to the host and then registering the output and looping over the `stdout_lines` part of the output.

```yaml
- name    : get files in /path/
  shell   : ls /path/*
  register: path_files

- name: fetch these back to the local Ansible host for backup purposes
  fetch:
    src : /path/"{{item}}"
    dest: /path/to/backups/
  with_items: "{{ path_files.stdout_lines }}"
```

https://stackoverflow.com/questions/33543551/is-there-with-fileglob-that-works-remotely-in-ansible

## Copy

### 使用with_items复制多个文件/目录

```yaml
- hosts: blocks
  tasks:
  - name: Ansible copy multiple files with_items
    copy:
      src: ~/{{item}}
      dest: /tmp
      mode: 0774
    with_items:
      ['hello1','hello2','hello3','sub_folder/hello4']
```

### 复制具有不同权限/目的地设置的多个文件

```yaml
- hosts: all
  tasks:
  - name: Copy multiple files in Ansible with different permissions
    copy:
      src: "{{ item.src }}"
      dest: "{{ item.dest }}"
      mode: "{{ item.mode }}"
    with_items:
      - { src: '/home/mdtutorials2/test1',dest: '/tmp/devops_system1', mode: '0777'}
      - { src: '/home/mdtutorials2/test2',dest: '/tmp/devops_system2', mode: '0707'}
      - { src: '/home/mdtutorials2/test3',dest: '/tmp2/devops_system3', mode: '0575'}
```

### 复制与pattern（通配符）匹配的文件夹中的所有文件

```yaml
- hosts: blocks
  tasks:
  - name: Ansible copy multiple files with wildcard matching.
    copy:
      src: "{{ item }}"
      dest: /etc
    with_fileglob:
      - /tmp/hello*
```

### 复制查找到的文件

```yaml
- hosts: lnx
  tasks:
    - find: paths="/appl/scripts/inq" recurse=yes patterns="inq.Linux*"
      register: file_to_copy
    - copy: src={{ item.path }} dest=/usr/local/sbin/
      owner: root
      mode: 0775
      with_items: "{{ files_to_copy.files }}"
```

https://www.ewhisper.cn/how-to-copy-files-and-directories-in-ansible.html



提示用户输入信息 vars_prompt http://www.zsythink.net/archives/2680
内置变量 http://www.zsythink.net/archives/2715
jinja2 filters处理数据 http://www.zsythink.net/archives/2862
include http://www.zsythink.net/archives/2962 http://www.zsythink.net/archives/2977
ansible-vault http://www.zsythink.net/archives/3250

