# BOSH Release for Nexus Repository Manager

## How to deploy nexus-boshrelease

A sample manifest is following:

``` yml
---
name: nexus

releases:
- name: nexus
  version: 0.12.0
  url: https://github.com/making/nexus-boshrelease/releases/download/0.12.0/nexus-boshrelease-0.12.0.tgz
  sha1: 09debef2e945a905e089b8d7c445df4d0b32f4e6
- name: openjdk
  version: 8.0.1
  url: https://github.com/making/openjdk-boshrelease/releases/download/8.0.1/openjdk-boshrelease-8.0.1.tgz
  sha1: d02566fb6d974de4b60bf44dc21e56422c7da3fd
  
stemcells:
- alias: xenial
  os: ubuntu-xenial
  version: latest

instance_groups:
- name: nexus
  instances: 1
  vm_type: default
  persistent_disk: default
  stemcell: xenial
  azs: [z1]
  networks:
  - name: default
    static_ips: [((internal_ip))]
  jobs:
  - name: java
    release: openjdk
  - name: nexus
    release: nexus
    properties:
      nexus:
        heap_size: 768M
        max_direct_memory_size: 512M
  - name: nexus-backup
    release: nexus
    
update:
  canaries: 1
  max_in_flight: 1
  serial: false
  canary_watch_time: 1000-60000
  update_watch_time: 1000-60000
```

then,

```
bosh deploy -d nexus nexus.yml -v internal_ip=<your_static_ip>
```

You will be able to access `http://<your_static_ip>:8081`


## How to enable SSL

A sample manifest is following:

``` yml
---
name: nexus

releases:
- name: nexus
  version: 0.12.0
  url: https://github.com/making/nexus-boshrelease/releases/download/0.12.0/nexus-boshrelease-0.12.0.tgz
  sha1: 09debef2e945a905e089b8d7c445df4d0b32f4e6
- name: openjdk
  version: 8.0.1
  url: https://github.com/making/openjdk-boshrelease/releases/download/8.0.1/openjdk-boshrelease-8.0.1.tgz
  sha1: d02566fb6d974de4b60bf44dc21e56422c7da3fd

stemcells:
- alias: xenial
  os: ubuntu-xenial
  version: latest

instance_groups:
- name: nexus
  instances: 1
  vm_type: default
  persistent_disk: default
  stemcell: xenial
  azs: [z1]
  networks:
  - name: default
    static_ips: [((internal_ip))]
  jobs:
  - name: java
    release: openjdk
  - name: nexus
    release: nexus
    properties:
      nexus:
        heap_size: 768M
        max_direct_memory_size: 512M
        ssl_cert: ((nexus_ssl.certificate))
        ssl_key: ((nexus_ssl.private_key))
        ssl_only: true
  - name: nexus-backup
    release: nexus
    
update:
  canaries: 1
  max_in_flight: 1
  serial: false
  canary_watch_time: 1000-60000
  update_watch_time: 1000-60000

variables:
- name: nexus_pkcs12_password
  type: password
- name: nexus_keystore_password
  type: password
- name: default_ca
  type: certificate
  options:
    is_ca: true
    common_name: ca
- name: nexus_ssl
  type: certificate
  options:
    ca: default_ca
    common_name: ((internal_ip))
    alternative_names: 
    - ((internal_ip))
```

then,

```
bosh deploy -d nexus nexus.yml -v internal_ip=<your_static_ip>
```

You will be able to access `https://<your_static_ip>:8443`


## Backup and Restore with [BBR](http://www.boshbackuprestore.io/)

### Backup

```
$ BOSH_CLIENT_SECRET=<BOSH_CLIENT_SECRET> \
  bbr deployment \
  --target <BOSH_TARGET_IP> \
  --username <BOSH_CLIENT> \
  --deployment nexus \
  --ca-cert <PATH_TO_BOSH_SERVER_CERTIFICATE> \
    backup
```

### Restore

```
$ BOSH_CLIENT_SECRET=<BOSH_CLIENT_SECRET> \
  bbr deployment \
  --target <BOSH_TARGET_IP> \
  --username <BOSH_CLIENT> \
  --deployment nexus \
  --ca-cert <PATH_TO_BOSH_SERVER_CERTIFICATE> \
    backup \
  --artifact-path <PATH_TO_ARTIFACT_TO_RESTORE>
```

## How to create stand-alone vm on VirtualBox

Download [nexus.yml](deployment/nexus.yml).

```
$ bosh create-env nexus.yml -v internal_ip=192.168.230.40  --vars-store ./nexus-creds.yml
```

https://192.168.230.40

You can get `admin` user's password as follows:

```
bosh int nexus-creds.yml --path /admin_password
```

## How to develop this bosh release

```
bosh sync-blobs
bosh create-release --name=nexus --force --timestamp-version --tarball=/tmp/nexus-boshrelease.tgz && bosh upload-release /tmp/nexus-boshrelease.tgz  && bosh -n -d nexus deploy manifest.yml -v internal_ip=<nexus static ip> --no-redact
```
