# ansible-role-openwrt-compiler

## Description

This role is designed for compiling custom openwrt firmwares with docker containers based on docker images created with the the [```openwrt-builder```](https://github.com/gitinsky/ansible-role-openwrt-builder) role.

## Requirements

- ```/root/openwrt-1209-builder/run/run.sh``` for 12.09 builds
- ```/root/openwrt-1407-builder/run/run.sh``` for 14.07 builds
- ```/compile/logs/openwrt-1407-builder-raw.done``` (or ```/compile/logs/openwrt-1209-builder-raw.done``` ) should be generated inside the container after the build

## Variables

- ```open_wrt_release_version```: could be ```1407``` or ```1209```
- ```open_wrt_arch```: at least ```ar71xx``` is a valid one, for more options or for adding your own read [here](https://github.com/gitinsky/ansible-role-openwrt-builder#updating-role-with-more-architecture-specific-images).

## Paths

The following volumes are mounted to the containter:

* ```/root/openwrt-{{ open_wrt_release_version }}-builder/result``` to ```/compile/openwrt-{{ open_wrt_release_version }}/bin```
* ```/root/openwrt-{{ open_wrt_release_version }}-builder-{{ open_wrt_arch }}/logs``` to ```/compile/logs"
* ```/root/openwrt-{{ open_wrt_release_version }}-builder/run``` to ```/compile/run```
* ```/root/openwrt-{{ open_wrt_release_version }}-builder/config``` to ```/compile/config```

Container start command is ```/compile/run/run.sh {{ open_wrt_arch }}.config```.

So for ar71xx on 14.09 the following files are required:

- ```/root/openwrt-1407-builder/config/ar71xx.config```
- ```/root/openwrt-1407-builder/run/run.sh``` (see example below)

Log file could be found at ```/root/openwrt-1407-builder-ar71xx/logs/make.log``` if you write it to ```/compile/logs/make.log``` in your ```run.sh```.
Firmwares and packages should occure in the following folder: ```/root/openwrt-1407-builder/result```.

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
    - { role: openwrt-compiler, open_wrt_release_version: 1407, open_wrt_arch: ar71xx }

```

Here’s the sample ```run.sh```:

```
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

This run.sh generates log file at Logs could be tracked at ```/root/openwrt-{{ open_wrt_release_version }}-builder-{{ open_wrt_arch }}/logs:/compile/logs/make.log```
