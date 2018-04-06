# CFCR/Kubo Deployment with Compiled Releases

I want it to be fast and fun for people to get started with BOSH and Kubernetes.

How fun and fast? Try out:

```plain
bosh -d cfcr deploy <(curl -L https://raw.githubusercontent.com/starkandwayne/cfcr-compiled-deployment/master/cfcr.yml)
```

The only requirements are an existing BOSH environment with v3541.X stemcell and a cloud-config similar to [cf-deployment](https://github.com/cloudfoundry/cf-deployment).

Many of the BOSH releases used have been pre-compiled for you, so its even more lovely and fast.

```plain
$ bosh instances -d cfcr

Deployment 'cfcr'

Instance                                     Process State  AZ  IPs
master/a957bed5-8e09-46f5-94a8-ef2964cfebdb  running        z1  10.10.1.10
worker/169c76e7-76b9-4546-949c-89e7630c9bed  running        z2  10.10.1.12
worker/2e5a58aa-5e7d-4e8d-9918-a5400cd6f278  running        z1  10.10.1.11
worker/e3ad4443-2d67-47ba-b661-5f68929bd711  running        z3  10.10.1.13
```

The `cfcr.yml` file, and the pre-compiled releases, are built by [Stark & Wayne CI pipeline](https://ci2.starkandwayne.com/teams/cfcommunity/pipelines/cfcr-compiled-deployment).