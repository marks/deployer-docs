---
title: central-config
category: Commands
order: 3
---

The `central-config` subcommand holds helpers related to managing a deployment's central configuration database.

## dump

Usage: `bin/domino-deployment central-config dump [stage name]`

Dump this deployment's central config to local disk.

> **Note:** Central config keys known to be secrets are automatically redacted from this dump.

The output of this command is a YAML file placed into `deploys/[stage]/central-config-[stage].[timestamp].yaml`.

The output has the following format:

```yaml
---
key: 'application.secret'
namespace: 'common'
name: null
value: null  # Redacted because this is a secret
---
key: 'com.cerebro.domino.apiHost'
namespace: 'common'
name: null
value: 'https://cydep98422290.deploy-team-sandbox.domino.tech'
---
key: 'com.cerebro.domino.aws.instanceProfileName'
namespace: 'common'
name: null
value: 'cydep98422290-executor'
---
key: 'securesocial.ssl'
namespace: 'role'
name: 'dispatcher'
value: 'true'
---
key: 'securesocial.ssl'
namespace: 'role'
name: 'frontend'
value: 'true'
---
```

## gen-target

Usage: `bin/domino-deployment central-config gen-target [stage]`

Given the deployment's manifest file, generates what would have been created as the initial Central Config values has
 this been run as a fresh install.

This "target" central config dump is output to `deploys/[stage]/target-central-config.yaml`.  This file has the same
format as the output of the `dump` subcommand, to make for easy command line diffing.


