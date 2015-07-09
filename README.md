# ansible-role-openwrt-compiler

## Description

This role is designed for compiling custom openwrt firmwares with docker containers based on docker images created with the the [```openwrt-builder```](https://github.com/gitinsky/ansible-role-openwrt-builder) role.

## Requirements

- ```{{ openwrt_config_dir }}/openwrt-1209-builder/run/run.sh``` for 12.09 builds
- ```{{ openwrt_config_dir }}/openwrt-1407-builder/run/run.sh``` for 14.07 builds
- ```/compile/logs/openwrt-1407-builder-raw.done``` (or ```/compile/logs/openwrt-1209-builder-raw.done``` ) should be generated inside the container after the build
- ```/compile/logs/make.log``` should exist. It's a good practice to recreate it on each new build, see example below. Final task of this role checks this file for make errors.

## Variables

- ```open_wrt_release_version```: could be ```1407``` or ```1209```
- ```open_wrt_arch```: at least ```ar71xx``` is a valid one, for more options or for adding your own read [here](https://github.com/gitinsky/ansible-role-openwrt-builder#updating-role-with-more-architecture-specific-images).

## Paths

The following volumes are mounted to the containter:

* ```{{ openwrt_config_dir }}/openwrt-{{ open_wrt_release_version }}-builder-{{ open_wrt_arch }}_{{ open_wrt_subtarget }}/result``` to ```/compile/openwrt-{{ open_wrt_release_version }}/bin```
* ```{{ openwrt_config_dir }}/openwrt-{{ open_wrt_release_version }}-builder-{{ open_wrt_arch }}_{{ open_wrt_subtarget }}/logs``` to ```/compile/logs"
* ```{{ openwrt_config_dir }}/openwrt-{{ open_wrt_release_version }}-builder/run``` to ```/compile/run```
* ```{{ openwrt_config_dir }}/openwrt-{{ open_wrt_release_version }}-builder/config``` to ```/compile/config```

Container start command is ```/compile/run/run.sh {{ open_wrt_arch }}_{{ open_wrt_subtarget }}.config```.

So for ar71xx on 14.09 the following files are required:

- ```{{ openwrt_config_dir }}/openwrt-1407-builder/config/ar71xx_generic.config```
- ```{{ openwrt_config_dir }}/openwrt-1407-builder/run/run.sh``` (see example below)

Log file could be found at ```{{ openwrt_config_dir }}/openwrt-1407-builder-ar71xx/logs/make.log``` if you write it to ```/compile/logs/make.log``` in your ```run.sh```.
Firmwares and packages should occure in the following folder: ```{{ openwrt_config_dir }}/openwrt```.

In fact firmwares are build in ```{{ openwrt_config_dir }}/openwrt-1407-builder/result``` and then moved to ```{{ openwrt_config_dir }}/openwrt/{{ open_wrt_release_version }}/{{ open_wrt_arch }}/{{ open_wrt_subtarget }}/``` with rsync.

## Examples

Here’s the sample playbook:

```yaml
- name: create directories
  file:
    dest: "{{ item }}"
    state: directory
    mode: 0755
  with_items:
    - /root/openwrt/openwrt-1209-builder/config
    - /root/openwrt/openwrt-1407-builder/config
    - /root/openwrt/openwrt-1407-builder/run
    - /root/openwrt/openwrt-1209-builder/run
  tags:
    - ar71xx
    - ramips
    - x86

- name: put patches and files
  synchronize:
    src: "templates/{{ item.src }}/"
    dest: "/root/openwrt/{{ item.dst }}"
    archive: false
    recursive: true
    delete: "{{ item.delete}}"
    set_remote_user: no
    perms: yes
    checksum: true
    rsync_opts:
        --exclude=.git
        --exclude=.DS_Store
  with_items:
    - { src: 12.09/config, dst: openwrt-1209-builder/config , delete: yes }
    - { src: 14.07/config, dst: openwrt-1407-builder/config , delete: yes }
    - { src: 12.09/run,    dst: openwrt-1209-builder/run    , delete: no  }
    - { src: 14.07/run,    dst: openwrt-1407-builder/run    , delete: no  }
  tags:
    - ar71xx
    - ramips
    - x86

- hosts: openwrt-builders
sudo: yes
roles:
- { role: openwrt-compiler, open_wrt_release_version: 1407, open_wrt_arch: ar71xx, open_wrt_subtarget: generic,  tags: ['ar71xx']   }
- { role: openwrt-compiler, open_wrt_release_version: 1407, open_wrt_arch: ramips, open_wrt_subtarget: mt7620a,  tags: ['ramips']   }
- { role: openwrt-compiler, open_wrt_release_version: 1407, open_wrt_arch: x86   , open_wrt_subtarget: generic,  tags: ['x86'   ]   }
- { role: openwrt-compiler, open_wrt_release_version: 1407, open_wrt_arch: ar71xx, open_wrt_subtarget: mikrotik, tags: ['mikrotik'] }

```

Here’s the sample ```run.sh```:

```bash
#!/bin/bash
if test -n "$1"
then
  cp -v /compile/config/$1 /compile/openwrt-1407/.config
fi
cp -v /compile/config/010-https_uci_options_heartbeat.patch        /compile/openwrt-1407/feeds/oldpackages/net/coova-chilli/patches/
cp -v /compile/config/011-apple_captive_portal_support_1.3.0.patch /compile/openwrt-1407/feeds/oldpackages/net/coova-chilli/patches/

test -f /compile/logs/make.log && mv -v /compile/logs/make.log /compile/logs/make.log.1

mkdir -vp /compile/openwrt-1407/files/
cp -rv /compile/config/files/* /compile/openwrt-1407/files

# get architecture from config file
arch=$(grep '^CONFIG_TARGET_BOARD' /compile/openwrt-1407/.config|cut -d '"' -f 2| tee /dev/stderr)
# Remove compiled files to prevent dependency error
rm -rf /compile/openwrt-1407/bin/$arch
if [[ "$arch" != "x86"  ]]; then
  rm -v /compile/openwrt-1407/files/etc/config/network
fi

cd /compile/openwrt-1407

find /compile/config/patches/$arch/ -type f | {
  while read patch
  do
    patch -p0 < bin/4to8lzma.patch 2>&1 | tee -a /compile/logs/make.log
  done
}

chown compile:compile /compile/openwrt-1407/.config
chown compile:compile /compile/openwrt-1407/bin
su compile -c "git pull"       2>&1 | tee -a /compile/logs/make.log
su compile -c "time make V=99" 2>&1 | tee -a /compile/logs/make.log
date > /compile/logs/openwrt-1407-builder-raw.done
while true
do
  sleep 60
done
```

This run.sh generates log file at Logs could be tracked at ```{{ openwrt_config_dir }}/openwrt-{{ open_wrt_release_version }}-builder-{{ open_wrt_arch }}_{{ open_wrt_subtarget }}/logs:/compile/logs/make.log```

## Find out ```arch``` and ```subtarget``` for ```.config``` files

```bash
ls *.config| {
  while read conf
  do
    arch=$(grep '^CONFIG_TARGET_BOARD' $conf|cut -d '"' -f 2)
    target=$(grep --color -E "^\s{0,}CONFIG_TARGET_${arch}_[a-z,0-9]{1,}=y" $conf| awk -F '[_=]' {'print $4'})
    echo "$conf: $arch $target"
  done
} | column -t
```
