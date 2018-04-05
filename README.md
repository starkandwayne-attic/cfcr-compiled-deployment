# CFCR/Kubo Deployment with Compiled Releases

I want it to be fast and fun for people to get started with BOSH and Kubernetes. 

So now you can get started with these commands:

* `bucc up` to use [BUCC](https://github.com/starkandwayne/bucc) to deploy a BOSH environment to your favourite cloud (or locally via VirtualBox)
* `bosh upload-stemcell` with a v3541.X stemcell for your CPI
* `bosh update-cloud-config` with a cloud-config that works with CFAR from `cf-deployment`
* `bosh deploy cfcr.yml` to deploy CFCR/Kubernetes using pre-compiled BOSH releases

Many of the BOSH releases used have been pre-compiled for you, so that's lovely and fast.

The `cfcr.yml` file, and the pre-compiled releases, are built by [Stark & Wayne CI pipeline](https://ci2.starkandwayne.com/teams/cfcommunity/pipelines/cfcr-compiled-deployment).