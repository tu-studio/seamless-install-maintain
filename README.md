# Getting started

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt

# run a playbook
ansible-playbook -i <your_inventory>.yml <playbook_name>.yml
```

# Full Install

## Debian installation

Install Debian 12 on all machines. In the installer select or do the following:

1. Language: English
2. Location: Germany
3. Locale: en_US.UTF-8
4. Keyboard: German
5. Select the correct network interface (you'll have to find out)
6. Hostname: kaoruXX for HuFo renderers (renderers)
7. Domain: the correct domain 
    - H0104: empty
    - EN325: ak.tu-berlin.de
    - HuFo: tu-ctrl
8. Root password: leave empty for automatic enabling of sudo. The standard user 
   will be added to the sudoers group automatically. If you are prompted
   a `sudo` password while using the system, just use the one you set in step 11.
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
13. wait for install of base system
14. mirror country: Germany
15. select a mirror (deb.debian.org for example)
16. HTTP proxy: none (or the correct proxy at HuFo)
17. Participate in package usage survey: No
18. Software selection:
    - SSH server
    - standard system utilities
    - On player, video and info PC: choose a desktop environment, currently we
      use KDE (but that might be overkill)
19. Reboot

## Perform installation using Ansible

Ansible playbooks for installation are run with this command:

```bash
ansible-playbook -i <your_inventory>.yml <playbook_name>.yml
```

Whenever a `install/*.yml` file is mentioned in the following, it should be
executed using the command above.

1. Clone the
   [seamless-install-maintain](https://github.com/tu-studio/seamless-install-maintain/tree/main)
   repo.
2. Create `vars/vault.yml` from the `vars/vault.yml.template`, fill it in. If
   regularly switching between locations, create multiple vault files, and just
   symlink them to the currently required one: `ln -sf vault_hufo.yml
   vars/vault.yml`
3. If you use a Dante soundcard, change the path in `vars/vars.yml` to the
   correct path to the driver directory.
4. Add host to Ansible inventory file, follow the provided inventory files for
   guidance
   - Hosts should be grouped into the sections
       - `renderer`
       - `player`,
       - `video_player` 
       - `info_player`
   - Hosts should contain the following variables (if used):
     - `ansible_host`: hostname
     - `ansible_user`: username 
     - `services`: containing a list of services running on this machine,
       available services are
         - `osc-kreuz`
         - `jack-connection-manager`
         - `audio-matrix`
         - `ambisonics`
         - `gui`
         - `showcontrol`
         - `twonder`
         - `location`: the location of this machine for use with the configs,
           at the moment one of `HUFO`, `EN325` or `H0104`
     - `audiodriver`: list of strings or single string of `dante` or `madi`
     - `n_twonders` only necessary if services contains `twonder`, sets up the
       `twonder.target` with the correct number of twonders
5. Roll out your SSH key to PCs: `ansible-playbook install/rollout_ssh_key.yml
   --ask-pass -e "key=<path/to/public/key.pub>"`. It might be necessary to
   install the `sshpass` package for your system.
6. Change the SSH key in the `ansible.cfg` file to point to your private key.
7. If necessary, install `sudo`: `install/install_sudo.yml`
8. Set up btrfs subvolumes (don't do this more than once!):
   `install/setup_btrfs.yml -k`. This needs the `-k` option to ask for the SSH
   connection password, because the moving around of btrfs subvolumes briefly
   moves the `.ssh` directory somewhere else...
9.  When in HuFo: run HuFo specific scripts:
   1. `install/hufo_setup_proxy_server.yml`
   2. `install/hufo_first_run.yml`
   3. `install/hufo_avm_user.yml`
10. start main installation script: `install/full_install.yml`

## Install playback system

The seamless player needs a bit more love. First download the x64 Linux version
of the SWS extension for REAPER from [here](https://sws-extension.org/). The
REAPER version is specified in `program_versions.yml`, consequently it can be
overwritten using `-e` for both `reaper_archive_name` and `reaper_url`

Run these playbooks:

1. `install/setup_player_autologin_and_desktop.yml`
2. `install/setup_player_reaper.yml -e "sws_archive=<path/to/your/sws-x.xx.x.x-Linux-x86_64.tar.xz>"`
3. `install/install_player_showcontrol.yml`
4. `install/install_player_seamless-plugin-suite.yml`
5. only in `H0104`: `install/setup_player_dante_bridge.yml`

Should you have problems with NVIDIA graphics cards and Wayland, run the
playbook `setup_player_nvidia_drivers`

Some steps in REAPER need to be performed manually in the GUI:

#### Set up REAPER remote control

- in REAPER go to `options->Preferences->Control/OSC/web`
- Press "add" to add a new conrol surface of mode `OSC`
- Device Name: Showcontrol or something like that
- Pattern Config:
  - select `(open config directory)`
  - copy `HufoShowControl.ReaperOSC` from the
    [showcontrol](https://github.com/tu-studio/showcontrol) repo there
  - select `(refresh list)`
  - select `HufoShowControl`
- Mode: `Local port [receive only]`
  - local listen port: `8000`
  - local ip: `0.0.0.0` 

#### set up REAPER

- set number of outputs to 64

### Set up remote desktop

VNC with the help of `krfb` is installed using Ansible playbook
`install/setup_player_autologin_and_desktop.yml`, however not started
automatically.

Configure `krfb` on the desktop: System Settings > Sharing > Desktop Sharing > Configure

# Varia

## Specify program versions

Desired versions are specified in `vars/program_versions.yml`, these correspond
to GitHub tags/branches/commits.

To overwrite the version of a specific program pass it to the
`ansible-playbook` call as `-e "<program>_version=X"`

## Encrypt variable

```bash
ansible-vault encrypt_string --name "variable_name"
```

## Additional Ansible scripts not run from full_install

- `install/install_jack-silence-detector`: debugging tool to discover longer
  silences (was used to debug reaper crashes)
- `install/reboot.yml`: used to reboot everything, sometimes used by services
- `install/remove_apt_cdrom_source.yml`: sometimes needed if installation of
  programs on fresh debian installs fails.
- `install/upgrade_system.yml`: performs a system upgrade

## Maintenance playbooks

- `maintain/pull_videos.yml`: pulls all videos from the video players to your
  local machine
- `maintain/rollout_videos.yml`/`maintain/rollout_info_text.yml`: roll out the
  video/info video files of the desired piece to all video/info players, has to
  be modified before use:
  1. Add video file name to `maintain/templates/playlist.txt.j2`, `{{ video_id
     }}` will be replaced with the index of this video player (1-6).
  2. Change `project_source` to point to the folder containing the videos on
     your local machine.
  3. Add a block like this to the tasks in `rollout_videos.yml`:
    ``` yaml
        - name: "Copy <new_project> onto the server"
        copy:
            src: "{{ project_source }}/<new_project>/cool_filename-0{{ video_id }}.mp4"
            dest: "{{ target_content_dir  }}"
            owner: kiosk
            group: avm
            mode: "u=rwx,g=rwx,o=rx"
    ```
