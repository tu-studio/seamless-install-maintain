# Full Install

Copy your ssh-key over
`ssh-copy-id -i ~/.ssh/private_key user@hostname`

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

you have to specify the vault password, the easiest way to do this, is to specify it in a file and specify that file as `vault_password_file` in the `ansible.cfg`

# specify versions
desired versions are specified in `vars/program_versions.yml`

to overwrite the version of a specific program pass it to the `ansible-playbook` call as `-e "<program>_version=X"`

# encrypt variable
`ansible-vault encrypt_string --name "variable_name"`
