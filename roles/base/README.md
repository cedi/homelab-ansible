# Ansible Role Base

This ansible role handles the base setup of all my hosts.

## Supported Operating Systems

* Debian

## Role Variables

Available variables are listed below, along with default values (see [defaults/main.yml]):

### base__system_packages

```yaml
base__system_packages: []
```

System packages to install.
This value will be set with the OS specific value in the [vars/] directory.

### base__additional_package_repositories: []

```yaml
base__additional_package_repositories: []
#  - key:
#      url: https://raw.githubusercontent.com/eza-community/eza/main/deb.asc
#      name: gierens.asc
#    repo: deb [signed-by=/etc/apt/keyrings/gierens.asc] http://deb.gierens.de stable main
#    install_package: eza
```

Additional system packages to install from custom repositories.
This value will be set with the OS specific value in the [vars/] directory.

### base__locales

```yaml
base__locales:
    - "en_US.UTF-8"
```

Locales to generate

### base__lang

```yaml
base__lang: "en_US.UTF8"
```

### base__home_dir

```yaml
base__home_dir: "/home"
```

### base__installer_scripts

```yaml
base__installer_scripts:
- url: https://raw.githubusercontent.com/warrensbox/terraform-switcher/release/install.sh
    name: tfswitch-install.sh
    creates: /usr/local/bin/tfswitch
- url: https://raw.githubusercontent.com/ajeetdsouza/zoxide/main/install.sh
    name: zoxide_install.sh
    args: "--bin-dir /usr/local/bin"
    creates: /usr/local/bin/zoxide
- url: https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh
    name: nvm_install.sh
    creates: /home/pi/.config/nvm
- url: https://ohmyposh.dev/install.sh
    name: ohmypos_install.sh
    args: "-d /usr/local/bin"
    creates: /usr/local/bin/oh-my-posh
- url: https://sh.rustup.rs
    name: rust_install.sh
    args: "-y"
    creates: /home/pi/.cargo/bin/cargo
```

Custom programs to install by downloading the installer script.

### base__installer_from_git

```yaml
base__installer_from_git:
- repo: <https://github.com/udhos/update-golang.git>
    dest: /home/{{ ansible_user }}/.update_golang/
    command: "./update-golang.sh"
```

Custom programs to install by cloning a git-repository and running the installer script.

## Example Playbook

```yaml
---
- hosts: all
  gather_facts: true
  become: true
  roles:
    - role: base
```

## License

MIT / BSD

[defaults/main.yml]: defaults/main.yml
[vars/]: ./vars/
