---
title: Upgrading to 2.4
category: Upgrading
order: 1
---

## Overview

Domino version 2.4 introduces two major technolgy changes to the Deployer:

### Containerization

Domino 2.4.0 introduces the option to deploy Domino application assets as Docker images. This change

To enable this behavior, the following changes will need to be made to your manifest:

```yaml
add:
  some:
    docs: 'here'
```


### Kubeadm

A TWELVE STEP PROGRAM FOR UPGRADING A k8s-master TO kubeadm-masterworker

1. Comment out or remove the k8s-master instance block

(vim shortcut: ctrl-v, hit down until all lines are selected, 'I' (capitalized), '#', escape, down)

    #    k8s-master:
    #      services:
    #        k8s-master:
    #      instance_config:
    #        root_volume_size: 8
    #        data_volume_size: 1
    #        encrypt_ebs_volumes: 'false'
    #        instance_type: 'm4.large'
    #        instance_profile_name: 'domino_kubernetes'

2. Add a kube-mw block

kubeadm-masterworker requires either CentOS/RHEL 7 or Ubuntu 16.04.

Our example here will use 16.04.

* The only service should be 'kubeadm-masterworker'
* Must override ami. Our 16.04 ami in us-west-2 is: ami-0def3275

Example:

    kube-mw:
      services:
          kubeadm-masterworker:
      instance_config:
        ami: 'ami-0def3275'
        root_volume_size: 20
        data_volume_size: 8
        docker_volume_size: 50
        encrypt_ebs_volumes: 'false'
        instance_type: 'm4.xlarge'

3. Add a kubernetes.enabled block and set docker 1.12.3

    kubernetes:
      enabled: 'true'

    docker:
      version: '1.12.3'

This will _NOT_ upgrade existing docker installs.

4. Reprovision

*AWS: Re-provision*

* This will DESTROY the existing k8s-master instance
* This will create a NEW kubeadm-masterworker instance

*ON-PREM: Re-install the machine / use a new vm*

* If we aren't getting a new machine/VM, then the customer
  should reinstall or reimage the machine.

5. Remove salt keys for k8s-master

    ./bin/domino-deployment stagename ssh --instance k8s-master sudo salt-key -d stagename-k8s-master

6. Install salt on our masterworker

    ./bin/domino-deployment deploy stagename --instances kube-mw --reset-salt --deploy-phases install-salt

* "--reset-salt" forces salt (re)installation
* "--instances kube-mw" restricts salt installation to our new instance.
  Without this, salt would get reset across the deployment, which will
  cause any stopped executors to be unable to highstate when started.
  If you run without this, be sure to terminate all stopped executors.

7. Deploy

Just a standard deploy:

    ./bin/domino-deployment deploy stagename

8. Remove hard-coded k8s-master url from central config

Remove this key from central config:

    common::com.cerebro.domino.executor.kubernetes.serverUrl

9. Restart frontends/dispatcher

10. Build a v2 environment!
11. Accept a higher power
12. Get wasted


