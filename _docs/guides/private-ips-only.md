---
title: Private IPs Only
category: Guides
order: 4
---

Dump from DEP-31, needs to be cleaned up eventually. Joaquin Lopez wrote these. CS should clean up based on experience
in the field

> **Note:** these steps complement [Internal Training about VPC Install/Deployment]() (these steps do not replace that page)

## Overview

With the right settings in the manifest, the deployer will now create a "bastion instance", allowing you to access a deployer install and deploy to infrastructure using only private ips.
The process is to do the initial provision step on your workstation (or any other machine you have access to), and then do the deploy step on the bastion instance. Once the bastion instance is created, you can use it to do everything (including re-run the provision step for upgrades and such).

## Deployer Setup

You'll need to setup the deployer on some machine accessible to you.

Since you will be creating a bastion instance for future upgrades, I suggest using your own workstation if possible.

Refer to the usual instructions.

### Docker for Mac Note

If you're using Docker for Mac, you may need to add the '/domino' path to docker to use our standard Deployer Docker Image instructions. Our usual run_docker.sh script tries to mount /domino/, which isn't in the list of paths Docker for Mac allows you to mount by default.

Go to the "Docker" menu on the menu bar, choose "File Sharing", hit the "+" sign and add "/domino" to the list. Click "Apply & Restart" and the Docker instructions should work.

## Manifest

You'll want to make sure the following is set:

```yaml
provisioning:
  aws:
    util_subnet_cidr_block: '__UTIL_SUBNET_HERE__'
    use_nat: 'true'
  private_ips_only: 'true'
```

Be mindful that private_ips_only is under "provisioning", not "aws".

This will end up creating a NAT gateway in the util subnet and provision a bastion intance, in addition to the normal assets.

### Important Change to Note

If `provisioning.aws.use_elb` is 'true', it will create a **PUBLIC** elb even with private_ips_only on.

Previously, with private_ips_only, it turned into a private ip elb. Now, we have use_int_elb, so private_ips_only + private elb configs should
have use_int_elb instead of use_elb.

The use case is to have all servers private, but still have secured public access.

### Deploy Part 1: Origin Story

Once you have your deployer setup created, you can run the provision step. Optionally, you can run only the portion of the provision step that creates the vpc, nat gateway and bastion instance, and then finish the provision step by rerunning it on the bastion instance.

You can get the bastion/deployer instance IP either from the aws console, or from the terraform output:

```
20170728-205229: DEPLOYER_PUBLIC_IP = 1.2.3.4
```

Once you've done your provision, you can rsync your local deployer directory to the bastion instance (from outside docker):

```bash
rsync -e "ssh -i keys/domino" --rsync-path="sudo rsync" -ai /domino ubuntu@1.2.3.4:/
```

### Deployer Part 2: Rebirth

ssh into your bastion instance, become root, install docker w/ install_docker.sh

Once docker is installed, you should be able to use the usual run_deployer.sh after doing a docker login to quay.io

### Deployer Part 3: The Revenge

Add the newly created bastion/deployer instance Private IP to the manifest (along with Domino VPN, and any customer specific CIDR blocks), Re-run the provision step, either to finish (if you chose to only setup the bastion instance) or as a sanity check.

Then, run the deploy step as you normally would. It should ssh into the instances, install salt and highstate as it normally would. At this point, your bastion instance should be functionally equivalent to most deployer box instances that you're used to.

You're done!
