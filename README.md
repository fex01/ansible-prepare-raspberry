prepare-raspberry
=========

Bootstrap and customize a Raspbian Lite system

### It will:
* rename user
  * rename the default user *pi* to *user_name*
  * set a new user password *user_password*
  * rename the home dir */home/pi* to */home/{{ user_name }}*
  * create a symlink */home/pi* linking to */home/{{ user_name }}*
* rename host
  * change the hostname to *host_name*
  * update `/etc/hosts`
* configure locale
  * set timezone to *localization_timezone*
* activate auto-updates / -upgrades
* SSH config
  * set sensible defaults for the SSH server
  * write public SSH keys to *user_name*'s *authorized_keys* file
  * disable SSH *root* login
  * enable SSH strict mode
  * disable X11 forwarding
  * disable SSH password login (role will fail if not at least one key is provided in *ssh_public_keys*)
  * allow SSH access only to *user_name*
  * add an SSH banner
  * option to set additional SSH settings
* install and configure a firewall (ufw)


Requirements
------------

* Role will fail if *ssh_public_keys* is empty (at least one key is needed to enable passwordless SSH authentication)
* *ansible_host* or *inventory_hostname* has to be set to the hosts IP address so that you can connect to your host before and after changing the hostname.
* `gather_facts: false`, otherwise reruns will run into trouble since we change SSH port & username (facts will be gathered after the correct connection details have been determined)
* Setup
  ### prepare SD card
    * download the [Raspbian Lite image](https://www.raspberrypi.org/downloads/)
    * write image on SD card (instructions at [raspberrypi.org](https://www.raspberrypi.org/documentation/installation/installing-images/README.md))
    * enable ssh by creating a file called *ssh* on the boot partition
    * [optional] enable WIFI by creating a file called *wpa_supplicant.conf* on the boot partition with the following content:
    ```
    network={
            ssid="your ssid"
            psk="your password"
    }
    ```
    * eject SD card & insert it into your Raspberrypi

  ### enable key-based ssh auth for pi
    * [optional] generate ssh public key on your ansible machine: `ssh-keygen -o`
    * [optional] delete old host key (name): `ssh-keygen -f "/home/<your user name>/.ssh/known_hosts" -R "raspberrypi"`
    * copy ssh public key to your Pi (password raspberry): `ssh-copy-id pi@raspberrypi`


Role Variables
--------------

```prepare-raspberry/defaults/main``` contains variables which should be overwritten to customize this role:
```yml
# These defaults mention only the most commonly overwritten variables of dependencies.
# For additional options have a look into the dependency roles defaults.

# rename-user
# Change to the username of your choice
user_name: "link"
# Password for your user
user_password_hash: "zelda"

#rename-host
# Change to the hostname of your choice
host_name: "hyrule"

# localization
# Set your timezone
localization_timezone: 'Europe/Berlin'
# Default locale. See `man 7 locale`
localization_default_locale: 'en_US.UTF-8'

# apt
# upgrade system: safe | full | dist
apt_upgrade: full
# Do “apt-get upgrade –download-only” every n-days (0=disable)
apt_download_upgradeable_packages: 1
# Do “apt-get autoclean” every n-days (0=disable)
apt_auto_clean_interval: 1
# Split the upgrade into the smallest possible chunks so that
# they can be interrupted with SIGUSR1. This makes the upgrade
# a bit slower but it has the benefit that shutdown while a upgrade
# is running is possible (with a small delay)
apt_unattended_upgrades_minimal_steps: yes
# Send email to this address for problems or packages upgrades
# If empty or unset then no email is sent, make sure that you
# have a working mail setup on your system. A package that provides
# 'mailx' must be installed. E.g. "user@example.com"
apt_mails: []
# Automatically reboot *WITHOUT CONFIRMATION*
# if the file /var/run/reboot-required is found after the upgrade
apt_unattended_upgrades_automatic_reboot: yes
# If automatic reboot is enabled and needed, reboot at the specific
# time instead of immediately
# Values: now | 02:00 | ...
apt_unattended_upgrades_automatic_reboot_time: now

# SSH
# grant SSH access only for
ssh_users: 
  - "{{ user_name }}"
# Set a custom ssh port
ssh_port: 22
# Required field, list of ssh public keys to update ~/.authorized_keys. 
# Note: If you don't provision your user with keys, your user will no longer be able to access the host via SSH.
# !!!The first key has to be the key ansible is using!!!
ssh_public_keys: []
# String to present when connecting to host over ssh
ssh_banner: "Welcome to {{ user_name }}'s {{ host_name }} server\n"
# Basic sshd config will be done, use this to set additional values.
# Syntax:
#   - var_name: "<settings name>"
#     value: "<settings value>"
# For details about setting names and values check /etc/ssh/sshd_config.
ssh_sshd_config_settings:
  - var_name: "MaxAuthTries"
    value: "3"
  - var_name: "PubkeyAuthentication"
    value: "yes"
  - var_name: "AuthorizedKeysFile"
    value: ".ssh/authorized_keys"
  - var_name: "IgnoreUserKnownHosts"
    value: "yes"
  - var_name: "IgnoreRhosts"
    value: "yes"
  - var_name: "AllowAgentForwarding"
    value: "no"
  - var_name: "AllowTcpForwarding"
    value: "no"
  - var_name: "GatewayPorts"
    value: "no"
  - var_name: "PermitTTY"
    value: "yes"
  - var_name: "Protocol"
    value: "2"

# ufw
ufw_rules:
  - { rule: "allow", port: "{{ ssh_port }}", proto: "tcp", comment: 'SSH' }
```


Dependencies
------------
* [arillso.localization](https://galaxy.ansible.com/arillso/localization)
* [weareinteractive.apt](https://github.com/weareinteractive/ansible-apt)
* [weareinteractive.users](https://github.com/weareinteractive/ansible-users)
* [weareinteractive.ufw](https://github.com/weareinteractive/ansible-ufw)
The list of dependency roles can also be found in `prepare-raspberry/meta/requirements.yml` and be install with `ansible-galaxy install -r requirements.yml`


Example Playbook
----------------

```yml
- name: My Playbook
  hosts: raspberrypi
  gather_facts: false
  become: true
  vars:
    - user_name: link
    - ssh_port: 394586
    - ssh_ansible_key: ssh-rsa XXXXXX[...]XXX ansible_user@ansible-machine.local
  
  tasks:
  - include_role:
      name: prepare-raspberry
```


License
-------

MIT
