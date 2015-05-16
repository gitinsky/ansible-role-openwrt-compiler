# ansible-role-openwrt-compiler

## Description

This role is designed for compiling custom openwrt firmwares with docker containers based on docker images created with the the [```openwrt-builder```](https://github.com/gitinsky/ansible-role-openwrt-builder) role.

## Requirements

- ```/root/openwrt-1209-builder/run/run.sh``` for 12.09 builds
- ```/root/openwrt-1407-builder/run/run.sh``` for 14.07 builds
- ```/compile/logs/openwrt-1407-builder-raw.done``` (or ```/compile/logs/openwrt-1209-builder-raw.done``` ) should be generated inside the container after the build

Here’s the sample run.sh file for 14.07:

```bash
#!/bin/bash
if test -n "$1"
then
  cp -v /compile/config/$1 /compile/openwrt-1407/.config
fi
cp -v /compile/config/010-https_uci_options_heartbeat.patch  /compile/openwrt-1407/feeds/oldpackages/net/coova-chilli/patches/
cp -v /compile/config/011-apple_captive_portal_support_1.3.0 /compile/openwrt-1407/feeds/oldpackages/net/coova-chilli/patches/

chown compile:compile /compile/openwrt-1407/.config
chown compile:compile /compile/openwrt-1407/bin
cd /compile/openwrt-1407 && su compile -c "time make V=99"      2>&1 | tee -a /compile/logs/make.log
date > /compile/logs/openwrt-1407-builder-raw.done
while true
do
  sleep 60
done

```

Logs could be tracked at ```/root/openwrt-{{ open_wrt_release_version }}-builder-{{ open_wrt_arch }}/logs:/compile/logs/make.log```

## Variables

- ```open_wrt_release_version```: could be ```1407``` or ```1209```
- ```open_wrt_arch```: at least ```ar7xxx-ar9xxx``` is a valid one, for more options or for adding your own read [here](https://github.com/gitinsky/ansible-role-openwrt-builder#updating-role-with-more-architecture-specific-images).

## Examples

Here’s the sample playbook:

```yaml
- hosts: openwrt-builders
  sudo: yes
  tasks:
       - name: create directories
       file:
         dest: "{{ item }}"
         state: directory
         mode: 0755
       with_items:
         - /root/openwrt-1209-builder/config
         - /root/openwrt-1407-builder/config
         - /root/openwrt-1407-builder/run/
         - /root/openwrt-1209-builder/run/

     - name: put patches and files
       template:
         src: "templates/{{ item.file }}"
         dest: "{{ item.dst }}"
         mode: "{{ item.mode }}"
       with_items:
         - { file: 010-https_uci_options_heartbeat.patch              , dst: /root/openwrt-1209-builder/config/ , mode: "u=rw,g=rw,o=rw" }
         - { file: 12.09/011-apple_captive_portal_support.patch       , dst: /root/openwrt-1209-builder/config/ , mode: "u=rw,g=rw,o=rw" }
         - { file: 12.09/Makefile                                     , dst: /root/openwrt-1209-builder/config/ , mode: "u=rw,g=rw,o=rw" }
         - { file: 12.09/run.sh                                       , dst: /root/openwrt-1209-builder/run/    , mode: "u=rwx,g=rw,o=rw" }
         - { file: 010-https_uci_options_heartbeat.patch              , dst: /root/openwrt-1407-builder/config/ , mode: "u=rw,g=rw,o=rw" }
         - { file: 14.07/011-apple_captive_portal_support_1.3.0.patch , dst: /root/openwrt-1407-builder/config/ , mode: "u=rw,g=rw,o=rw" }
         - { file: 14.07/run.sh                                       , dst: /root/openwrt-1407-builder/run/    , mode: "u=rwx,g=rw,o=rw" }


- hosts: openwrt-builders
  sudo: yes
  roles:
    - { role: openwrt-compiler, open_wrt_release_version: 1407, open_wrt_arch: ar7xxx-ar9xxx }

```