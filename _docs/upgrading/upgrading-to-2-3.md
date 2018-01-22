---
title: Upgrading to 2.3
category: Upgrading
order: 2
---

Domino version 2.3 adds several user-level, phased over several patches:

- 2.3.4-d1
    - Manifest wizard
    - Deployer version checks
- 2.3.3-d1
    - Static manifest checker

## Manifest Changes

Starting in v2.3.4-d1, a new manifest entry is added to help verify that the Deployer, Application Artifact, and the manifest itself are all of the same, compatible version.

As the deploy engineer, you should add the following to the top of the manifest to be updated:

```
meta:
  manifest-version: 2.3
```





