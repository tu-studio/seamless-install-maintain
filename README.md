# Full Install

Copy your ssh-key over
`ssh-copy-id -i ~/.ssh/private_key user@hostname`
Change the ssh-key in the ansible.cfg file to point to your private key.

Change the password in the vault.yml file in the `vars` directory with the password of the server you want to install on.

``` yaml
ansible_become_pass: sudo_password_here
```

Setup the btrfs filesystem using

``` bash
ansible-playbook -i H0104_inventory.yml install/install_sudo.yml
ansible-playbook -i H0104_inventory.yml install/setup_btrfs.yml -k
```

This has to be done using ssh-login via password. In order to use the -k option it might be necessary to install the `sshpass` package for your system.

Start the full install using

``` bash
ansible-playbook -i H0104_inventory.yml install/full_install.yml
```

## specify versions

desired versions are specified in `vars/program_versions.yml`

to overwrite the version of a specific program pass it to the `ansible-playbook` call as `-e "<program>_version=X"`

## encrypt variable

`ansible-vault encrypt_string --name "variable_name"`
