# HomeLab Ansible

This is the Ansible Playbook I am using to deploy my Homelab.

To run the deployment use

```sh
ansible-playbook playbooks/site.yml --ask-pass
```

This playbook uses [`geerlingguy.node_exporter`] and [`artis3n.tailscale`] which can be installed from the [requirements.yml] using

```sh
ansible-galaxy install -r  requirements.yml
```

[requirements.yml]: ./requirements.yml
[`artis3n.tailscale`]: https://github.com/artis3n/ansible-role-tailscale
[`geerlingguy.node_exporter`]: https://github.com/geerlingguy/ansible-role-node_exporter
