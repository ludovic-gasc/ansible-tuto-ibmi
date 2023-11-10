# Ansible tutorial for IBM i: 

## setup requirements on your IBM i LPARs

Please follow [the official documentation to install Ansible requirements](https://ibm.github.io/ansible-for-i/installation.html#enabling-ibm-i-nodes).

You might also use [the official setup.yml Ansible playbook](https://github.com/IBM/ansible-for-i/tree/devel/playbooks/enable-ansible-for-i).

## Adding your SSH keys on the virtual machines

To follow this tutorial, you'll need to have your keys in VMs root's
`authorized_keys`. While this is not absolutely necessary (Ansible can use
sudo, password authentication, etc...), it will make things way easier.

Ansible is perfect for this and we will use it for the job. However I won't
explain what's happening for now. Just trust me.

```bash
ansible-playbook -i step-00/hosts step-00/setup.yml
```

If you get "Connections timed out" errors, please check the firewall
settings of your machine.

If you get errors like:

```none
fatal: [192.168.33.10]: UNREACHABLE! => {"changed": false, "msg": "host key mismatch for 192.168.33.10", "unreachable": true}
```

then you probably already have SSH host keys for those IPs in your
`~/.ssh/known_hosts`. You can remove them with `ssh-keygen -R
<IP_ADDRESS>`.

Otherwise, just type `yes` when prompted to access ssh host keys if
requested.

To polish things up, it's better to have an ssh-agent running, and add your
keys to it (`ssh-add`).

**NOTE:** We are assuming that you're using Ansible version v2.5+ on your local
machine. If not you should upgrade ansible to v2.5+ before using this
repository (or run under virtualenv).

To check your ansible version use the command `ansible --version`. The output
should be similar to the above:

```none
ansible 2.10.5
  ...
  python version = 3.8.5 (default, Jul 28 2020, 12:59:40) [GCC 9.3.0]
```

Now head to the first step in
[step-01](https://github.com/ludovic-gasc/ansible-tuto-ibmi/tree/master/step-01).
