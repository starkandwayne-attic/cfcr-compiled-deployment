# CFCR/Kubo Deployment with Compiled Releases

I want it to be fast and fun for people to get started with BOSH and Kubernetes. 

So now you can get started with these commands:

* `bucc up` to use [BUCC](https://github.com/starkandwayne/bucc) to deploy a BOSH environment to your favourite cloud (or locally via VirtualBox)
* `bosh upload-stemcell` with a v3541.X stemcell for your CPI
* `bosh update-cloud-config` with a cloud-config that works with CFAR from `cf-deployment`
* `bosh deploy cfcr.yml` to deploy CFCR/Kubernetes using pre-compiled BOSH releases

Many of the BOSH releases used have been pre-compiled for you, so that's lovely and fast.

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