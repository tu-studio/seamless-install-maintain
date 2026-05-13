# Full Install

## initialize rendering machines
<details>
<summary>  
<b> open for our Debian setup</b>
</summary>

Install Debian 12 on all machines

In the Installer select or do the following
1. Language: English
2. Location: Germany
3. Locale: en_US.UTF-8
4. Keyboard: German
5. Select the correct network interface (you'll have to find out)
6. Hostname: kaoruXX for HuFo renderers (renderers)
7. Domain: the correct domain (empty in H0104/ak.tu-berlin.de in EN325/tu-ctrl at HuFo)
8. Root password: leave empty for automatic enabling of sudo/adding of the standard user to sudoers group
9. User: tu-studio
10. Full name: tu-studio
11. User password: use a good password here friend
12. Partitioning method: Manual (Guided partitioning manual)
  - /dev/nvme0n1
  - yes new partition table
  - 1.0 MB free space is done automatically
  - 512.0 MB EFI System Partition
    - Use as: EFI System Partition
    - Bootable flag: on
  - 850.0 GB brtfs / (root)
    - mount options: defaults
    - label: system
    - Bootable flag: off
  - 32 GB swap
    - Use as: swap area
    - Bootable flag: off
  - Finish partitioning and write changes to disk
13. wait for install of Base system
14. mirror country: Germany
15. select a mirror (deb.debian.org for example)
16. HTTP proxy: none (or the correct proxy at HuFo)
17. Participate in package usage survey: No
18. Software selection: SSH server, standard system utilities, ON PLAYER PC ONLY: choose a desktop environment, currently we use KDE (but that might be overkill)
19. Reboot

</details>

## perform installation using ansible
ansible playbooks for installation are run with this command:
`ansible-playbook -i <your_inventory>.yml <playbook_name>.yml`.

whenever a `install/*.yml` file is mentioned in the following, it should be executed using the command above.

1. clone this repo
2. create `vars/vault.yml` from the `vars/vault.yml.template`, fill it in. if regularly switching between locations create multiple vault files, and just symlink them to the currently required one: `ln -sf vault_hufo.yml vars/vault.yml`
3. if you use a dante soundcard change the path in `vars/vars.yml` to the correct path to the driver directory
4. add host to ansible inventory file, follow the provided inventory files for guidance
   - hosts should be grouped into the sections `renderer`, `player`, `video_player` and `info_player`
   - hosts should contain the following variables (if used):
     - `ansible_host`: hostname
     - `ansible_user`: username 
     - `services` containing a list of services running on this machine, available services are `osc-kreuz`, `jack-connection-manager`, `audio-matrix`, `ambisonics`, `gui`, `showcontrol` and `twonder`
     - `location`: the location of this machine for use with the configs, at the moment one of `HUFO`, `EN325` or `H0104`
     - `audiodriver`: list of strings or single string of `dante` or `madi`
     - `n_twonders` only necessary if services contains `twonder`, sets up the `twonder.target` with the correct number of twonders
5. rollout your ssh key to pcs: `ansible-playbook install/rollout_ssh_key.yml --ask-pass -e "key=<path/to/public/key.pub>"`. it might be necessary to install the `sshpass` package for your system.
6. change the ssh-key in the ansible.cfg file to point to your private key.
7. if necessary install sudo: `install/install_sudo.yml`
8. setup btrfs subvolumes (don't do this more than once!): `install/setup_btrfs.yml -k`. This needs the `-k` option to ask for the ssh connection password, because the moving around of btrfs subvolumes briefly moves the `.ssh` directory somewhere else...
9.  when in hufo: run hufo specific scripts:
   1. `install/hufo_setup_proxy_server.yml`
   2. `install/hufo_first_run.yml`
   3. `install/hufo_avm_user.yml`
10. start main install script: `install/full_install.yml`

## install seamless player
The seamless player needs a bit more love.
first download the x64 linux version of the SWS extension for REAPER from [here](https://sws-extension.org/). The REAPER version is specified in `program_versions.yml`, consequently it can be overwritten using `-e` for both `reaper_archive_name` and `reaper_url`


run these playbooks:
1. `install/setup_player_autologin_and_desktop.yml`
2. `install/setup_player_reaper.yml -e "sws_archive=<path/to/your/sws-x.xx.x.x-Linux-x86_64.tar.xz>"`
3. `install/install_player_showcontrol.yml`
4. `install/install_player_seamless-plugin-suite.yml`
5. only in `H0104`: `install/setup_player_dante_bridge.yml`

should you have problems with nvidia graphics cards and wayland runt the playbook `setup_player_nvidia_drivers`



some steps in REAPER need to be performed manually on the GUI:

#### setup REAPER remote control

- in reaper go to `options->Preferences->Control/OSC/web`
- press "add" to add a new conrol surface of mode `OSC`
- Device Name: Showcontrol or something like that
- Pattern Config:
  - select `(open config directory)`
  - copy `HufoShowControl.ReaperOSC` from the Showcontrol repo there
  - select `(refresh list)`
  - select `HufoShowControl`
- Mode: `Local port [receive only]`
  - local listen port: `8000`
  - local ip: `0.0.0.0` 

#### setup REAPER

- set number of outputs to 64

### setup remote desktop

vnc with the help of `krfb` is installed using ansible playbook `install/setup_player_autologin_and_desktop.yml`, however not started automatically.

configure `krfb` on the desktop: System Settings > Sharing > Desktop Sharing > Configure



# specify program versions

desired versions are specified in `vars/program_versions.yml`, these correspond to github tags/branches/commits

to overwrite the version of a specific program pass it to the `ansible-playbook` call as `-e "<program>_version=X"`

# encrypt variable

`ansible-vault encrypt_string --name "variable_name"`



# additional ansible scripts not run from full_install
- `install/install_jack-silence-detector`: debugging tool to discover longer silences (was used to debug reaper crashes)
- `install/reboot.yml`: used to reboot everything, sometimes used by services
- `install/remove_apt_cdrom_source.yml`: sometimes needed if installation of programs on fresh debian installs fails.
- `install/upgrade_system.yml`: performs a system upgrade

# maintenance playbooks
- `maintain/pull_videos.yml`: pulls all videos from the video players to your local machine
- `maintain/rollout_videos.yml`/`maintain/rollout_info_text.yml`: roll out the video/info video files of the desired piece to all video/info players, has to be modified before use:
  1. add video file name to `maintain/templates/playlist.txt.j2`, {{ video_id }} will be replaced with the index of this video player (1-6)
  2. put the video files into the `project_source` directory (or change the `project_source` directory to your video directory, whatever makes you happy)
  3. add a block like this to the tasks in `rollout_videos.yml`:
    ``` yaml
        - name: "Copy <new_project> onto the server"
        copy:
            src: "{{ project_source }}/<new_project>/cool_filename-0{{ video_id }}.mp4"
            dest: "{{ target_content_dir  }}"
            owner: kiosk
            group: avm
            mode: "u=rwx,g=rwx,o=rx"
    ```
