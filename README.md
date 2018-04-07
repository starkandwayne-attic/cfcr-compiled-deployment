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

## Thanks

This project is only possible thanks to the CFCR team (heavily sponsored by Pivotal, VMWare, and Google). The `ops-files` and `helper` folders are taken from the upstream [kubo-deployment](https://github.com/cloudfoundry-incubator/kubo-deployment) project's `manifests` folder. The `cfcr.yml` file is also originally sourced from [manifests/cfcr.yml](https://github.com/cloudfoundry-incubator/kubo-deployment/blob/master/manifests/cfcr.yml), but modified with pre-compiled final releases so everything "just works" out of the box.

Once kubo-deployment project starts maintaining a stable `master` branch and the `manifests/cfcr.yml` file includes pre-compiled final releases then this repository will not be necessary. Although the walk-thrus may still be valuable as they are different from https://docs-cfcr.cfapps.io/.

## Walk thru

This section will include a walk thru for various CPI/IaaS/Clouds.

### AWS

The prerequisites for this walk thru are:

* an AWS account
* an admin/poweruser IAM user with API keys
* a VPC and subnet
* security groups to allow ingress traffic on ports 22, 443, and 25555; as well as all traffic between VMs using the same security group

#### Deploy BOSH/BUCC to AWS

To provision a BOSH environment with Credhub/UAA/Concourse, I recommend [BUCC](https://github.com/starkandwayne/bucc). It makes it easy and provides you with helper commands to login to BOSH and CredHub.

NOTE: see comments below about performing this walk-thru upon an existing AWS VM rather than your local computer. See YouTube video [Walk thru using BUCC/BOSH](https://www.youtube.com/watch?v=aDg9kuv6hjg) to learn more about production AWS environments with jumpboxes, private subnets, and BUCC.

```plain
mkdir ~/workspace
cd ~/workspace
git clone https://github.com/starkandwayne/bucc
source <(~/workspace/bucc/bin/bucc env)
bucc up --cpi aws
```

This will download the `bosh` CLI and then create stub `~/workspace/bucc/vars.yml` where you fill in your AWS credentials, networking etc.

```plain
installing bosh cli '2.0.48' into: /Users/drnic/workspace/bucc/bin/
We just copied a template in /Users/drnic/workspace/bucc/vars.yml please adjust it to your needs
```

When you're done run `bucc up` again to bring up your BOSH environment:

```plain
bucc up
```

This will take some time depending a lot on your upload/download Internet speed. It will take:

* a few minutes or hours to download about 1G of pre-compiled BOSH releases
* a few minutesto compile the AWS CPI locally
* a minute or so to provision an AWS VM
* a few minutes or hours to upload the 1G of BOSH releases to the VM
* a few minutes to compile the AWS CPI upon your new VM
* a few minutes for the BOSH/UAA/CredHub/Concourse processes to start up

This sequence will be a lot faster if you perform it upon another AWS VM (download and upload times will reduce to almost zero).

When its complete, re-source your environment to setup various `$BOSH` environment variables for your BOSH environment:

```plain
source <(~/workspace/bucc/bin/bucc env)
bosh env
```

#### Setup AWS BOSH

The `bucc up` command creates or upgrades your BOSH environment. It does not populate it with stemcells or cloud-config (nor CFCR clusters or other deployments). So, let's setup our BOSH environment for CFCR.

A stemcell is BOSH terminology for AWS AMI in your AWS region. The `cfcr.yml` includes pre-compiled packages (hurray for not having to watch compilation) that require any `3541.X` stemcell:

```plain
bosh upload-stemcell https://bosh.io/d/stemcells/bosh-aws-xen-hvm-ubuntu-trusty-go_agent?v=3541.10
```

Setting up your cloud-config is more difficult to simplify for a walk-thru. But let's try.

There is a very simple AWS cloud-config provided by [bosh-deployment](https://github.com/cloudfoundry/bosh-deployment/).

```plain
bosh update-cloud-config ~/workspace/bucc/src/bosh-deployment/aws/cloud-config.yml \
  -l ~/workspace/bucc/vars.yml
```

This creates a cloud-config with a network `default` (the same subnet you used for your BOSH environment), and vm_types `default` (`m4.large`) and `large` (`m4.xlarge`). These are pretty good for us to get started with our CFCR deployment.

But they don't match the requirements of `cfcr.yml`. This file references `vm_types` named `small`, `small-highmem`, and `minimal`.

You have a choice: Either we can:

1. update our `cloud-config` to add these additional `vm_types`, or
1. update our `cfcr.yml` deployment manifest to use the available `vm_types`

Fortunately, this project (which originates from upstream [kubo-deployment](https://github.com/cloudfoundry-incubator/kubo-deployment)) contains an operator file to pick different `vm_types`. So I choose option 2. We will modify our `cfcr.yml` via operator files.

#### Deploy CFCR to AWS

You do not need to create or upload any BOSH releases. `bosh deploy cfcr.yml` will do everything for you.

```plain
cd ~/workspace
git clone https://github.com/starkandwayne/cfcr-compiled-deployment

export BOSH_DEPLOYMENT=cfcr
bosh deploy cfcr-compiled-deployment/cfcr.yml \
  -o cfcr-compiled-deployment/ops-files/vm-types.yml \
  -v master_vm_type=default \
  -v worker_vm_type=large
```

I ran these commands at 10:35am. It finished at around 10:50am, and created a full-functional 1-master, 3-worker cluster of Kubernetes:

```plain
$ bosh instances
Deployment 'cfcr'

Instance                                     Process State  AZ  IPs
master/129f5313-976b-4f89-89c3-dfcb3f207274  running        z1  10.10.2.5
worker/032096e3-84b0-46d5-9b94-2e0910ebbb1f  running        z2  10.10.2.7
worker/3a40714e-dcdb-47a1-b698-4ea434192b6b  running        z3  10.10.2.8
worker/ee08db00-3bc1-4b29-aef4-c207151bf26c  running        z1  10.10.2.6
```

#### Using CFCR on AWS

Once the deployment is running, you can setup your `kubectl` CLI to connect and authenticate you.

Download and install the `kubectl` CLI [using the instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl).

Firstly, to download the `credhub` CLI and login to CredHub API:

```plain
bucc credhub
```

To get the randomly generated Kubernetes API admin password from Credhub:

```plain
admin_password=$(bosh int <(credhub get -n "bucc/cfcr/kubo-admin-password" --output-json) --path=/value)
```

Next, get the dynamically assigned IP address of the `master/0` instance:

```plain
master_host=$(bosh int <(bosh instances --json) --path /Tables/0/Rows/0/ips)
```

Finally, setup your local `kubectl` configuration:

```plain
export BOSH_DEPLOYMENT=cfcr
cluster_name="cfcr:${BOSH_ENVIRONMENT}:${BOSH_DEPLOYMENT}"
user_name="cfcr:${BOSH_ENVIRONMENT}:${BOSH_DEPLOYMENT}-admin"
context_name="cfcr:${BOSH_ENVIRONMENT}:${BOSH_DEPLOYMENT}"

kubectl config set-cluster "${cluster_name}" \
  --server="https://${master_host}:8443" \
  --insecure-skip-tls-verify=true
kubectl config set-credentials "${user_name}" --token="${admin_password}"
kubectl config set-context "${context_name}" --cluster="${cluster_name}" --user="${user_name}"
kubectl config use-context "${context_name}"
```

To confirm that you are connected and configured to your Kubernetes cluster:

```plain
$ kubectl get all
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.200.1   <none>        443/TCP   17m
```

#### Example Kubernetes Usage

To deploy the [Kubernetes tutorial Guestbook application](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/):

```plain
cd ~/workspace
git https://github.com/kubernetes/examples kubernetes-examples
kubectl create -f kubernetes-examples/guestbook/redis-master-deployment.yaml
kubectl create -f kubernetes-examples/guestbook/redis-master-service.yaml
kubectl create -f kubernetes-examples/guestbook/redis-slave-deployment.yaml
kubectl create -f kubernetes-examples/guestbook/redis-slave-service.yaml
kubectl create -f kubernetes-examples/guestbook/frontend-deployment.yaml
kubectl create -f kubernetes-examples/guestbook/frontend-service.yaml
```

Watch the pods/containers come up:

```plain
kubectl get all
```

By default we will be able to access our frontend application by connecting directly to the workers.

```plain
$ kubectl get service frontend
NAME       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
frontend   NodePort   10.100.200.145   <none>        80:30841/TCP   4m

$ bosh instances
Instance                                     Process State  AZ  IPs
master/391853a3-0dd0-40ad-9e34-9ddc58c956c0  running        z1  10.10.2.5
worker/9fc2f8d9-c8ac-427e-8818-d2e3d00636b4  running        z3  10.10.2.8
worker/b53b3e64-ceef-4980-9145-832c637d4932  running        z1  10.10.2.6
worker/e85c1c25-17e7-4e81-87bf-7fcd7849cbb7  running        z2  10.10.2.7
```

In the example above, the frontend service is available on port `:30841` on each worker. Pick a worker IP a try to ping it:

```plain
curl 10.10.2.8:30841
```

#### Privileged containers

Some Kubernetes deployments required privileged containers. There is an operator file `allow-privileged-containers.yml` to enable this across your CFCR cluster:

```plain
bosh deploy cfcr-compiled-deployment/cfcr.yml \
  -o cfcr-compiled-deployment/ops-files/vm-types.yml \
  -v master_vm_type=default \
  -v worker_vm_type=large \
  -o cfcr-compiled-deployment/ops-files/allow-privileged-containers.yml
```

You can now deploy the `kubernetes-examples/staging/elasticsearch` deployment which requires privileged containers:

```plain
cd ~/workspace
kubectl create -f kubernetes-examples/staging/elasticsearch/service-account.yaml
kubectl create -f kubernetes-examples/staging/elasticsearch/es-svc.yaml
kubectl create -f kubernetes-examples/staging/elasticsearch/es-rc.yaml
```

Watch the `es` pod come up:

```plain
kubectl get pods
```

#### Setup TCP routing via Cloud Foundry routing mesh

Check that your Cloud Foundry has both HTTP and TCP domains. If missing TCP domains, [follow the instructions](https://docs.cloudfoundry.org/adminguide/enabling-tcp-routing.html) for setting them up.

```plain
$ cf domains
Getting domains in org system as admin...
name                   status   type
apps.mycompany.com     shared
tcp.mycompany.com      shared   tcp
```

We will also be changing how we interact with the Kubernetes API. Instead of using https://IP:8443 we will access it through the Cloud Foundry TCP routing hostname and the selected port; such as https://tcp.mycompany.com:8443

So we need need to have some certificates regenerated to include the new hostname. Delete them from Credhub:

```plain
cd ~/workspace
bucc bosh
bucc credhub

export BOSH_ENVIRONMENT=bucc
export BOSH_DEPLOYMENT=cfcr
credhub delete -n /${BOSH_ENVIRONMENT}/${BOSH_DEPLOYMENT}/tls-kubernetes
credhub delete -n /${BOSH_ENVIRONMENT}/${BOSH_DEPLOYMENT}/tls-kubelet
```

```plain
cfcr-compiled-deployment/helper/cf-routing-vars.sh > cf-vars.yml
```

```plain
bosh deploy cfcr-compiled-deployment/cfcr.yml \
  -o cfcr-compiled-deployment/ops-files/vm-types.yml \
  -v master_vm_type=default \
  -v worker_vm_type=large \
  -o cfcr-compiled-deployment/ops-files/allow-privileged-containers.yml \
  -o cfcr-compiled-deployment/ops-files/cf-routing.yml \
  -l cf-vars.yml
```

This will create a new instance called `route-sync` that will provide the facility to watch your Kubernetes services for tags and publish the required HTTP or TCP routes.

```plain
Task 1640 | 07:32:49 | Creating missing vms: route-sync/59fd5320-b7bf-4240-af48-e7cd36b69e7f (0) (00:01:05)
```

We can now re-configure `kubectl` to use the new hostname and its matching certificate (rather than use the smelly `--insecure-skip-tls-verify` flag).

First, get the randomly generated Kubernetes API admin password from CredHub again:

```plain
admin_password=$(bosh int <(credhub get -n "${BOSH_ENVIRONMENT}/${BOSH_DEPLOYMENT}/kubo-admin-password" --output-json) --path=/value)
```

Next, get your TCP hostname from your `cf-vars.yml` (e.g. `tcp.apps.mycompany.com`):

```plain
master_host=$(bosh int cf-vars.yml --path /kubernetes_master_host)
```

Then, store the root certificate in a temporary file:

```plain
tmp_ca_file="$(mktemp)"
bosh int <(credhub get -n "${BOSH_ENVIRONMENT}/${BOSH_DEPLOYMENT}/tls-kubernetes" --output-json) --path=/value/ca > "${tmp_ca_file}"
```

Finally, setup your local `kubectl` configuration:

```plain
cluster_name="cfcr:${BOSH_ENVIRONMENT}:${BOSH_DEPLOYMENT}"
user_name="cfcr:${BOSH_ENVIRONMENT}:${BOSH_DEPLOYMENT}-admin"
context_name="cfcr:${BOSH_ENVIRONMENT}:${BOSH_DEPLOYMENT}"

kubectl config set-cluster "${cluster_name}" \
  --server="https://${master_host}:8443" \
  --certificate-authority="${tmp_ca_file}" \
  --embed-certs=true
kubectl config set-credentials "${user_name}" --token="${admin_password}"
kubectl config set-context "${context_name}" --cluster="${cluster_name}" --user="${user_name}"
kubectl config use-context "${context_name}"
```

Confirm that the `:8443` TCP route and certificate for Kubernetes API are working:

```plain
kubectl get all
```

#### Register service to HTTP or TCP router

To register an HTTP route https://myelastic.apps.mycompany.com to route to your Elastic HTTP API:

```plain
kubectl label service elasticsearch http-route-sync=myelastic
```

The route will take a few moments to appear:

```plain
$ curl -k https://myelastic.apps.mycompany.com
502 Bad Gateway: Registered endpoint failed to handle the request.
$ curl -k https://myelastic.apps.mycompany.com
{
  "name" : "f6a77df3-a1ae-42a6-b749-87e3a7e88906",
  "cluster_name" : "myesdb",
  "cluster_uuid" : "ErRYBEU8QHChXeOV0NOhsA",
  "version" : {
    "number" : "5.6.2",
    "build_hash" : "57e20f3",
    "build_date" : "2017-09-23T13:16:45.703Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.1"
  },
  "tagline" : "You Know, for Search"
}
```

To assign yourself public port :9300 on the TCP routing tier `tcp.mycompany.com:9300`:

```plain
kubectl label service elasticsearch tcp-route-sync=9300
```

#### AWS Cloud Provider

```plain
mkdir -p bucc/operators
cp cfcr-compiled-deployment/ops-files/iaas/aws/bosh/tags.yml \
  bucc/operators/
bucc up
```

Kubernetes can be configured to communicate with AWS.

```plain
cd ~/workspace
bosh deploy cfcr-compiled-deployment/cfcr.yml \
  -o cfcr-compiled-deployment/ops-files/vm-types.yml \
  -v master_vm_type=default \
  -v worker_vm_type=large \
  -o cfcr-compiled-deployment/ops-files/allow-privileged-containers.yml \
  -o cfcr-compiled-deployment/ops-files/iaas/aws/cloud-provider.yml \
  -o cfcr-compiled-deployment/ops-files/iaas/aws/add-master-credentials.yml \
  -o cfcr-compiled-deployment/ops-files/iaas/aws/add-worker-credentials.yml \
  -v aws_access_key_id_master=$(bosh int bucc/vars.yml --path /access_key_id) \
  -v aws_access_key_id_worker=$(bosh int bucc/vars.yml --path /access_key_id) \
  -v aws_secret_access_key_master=$(bosh int bucc/vars.yml --path /secret_access_key) \
  -v aws_secret_access_key_worker=$(bosh int bucc/vars.yml --path /secret_access_key)
```

#### Deploy another CFCR cluster

Why have one CFCR/Kubernetes cluster, when you can have many? Each will be independently deployed, configured, and upgradable over time.

Each BOSH deployment needs a unique name. Currently `cfcr.yml` has a hard-coded `name: cfcr`. We can change this with an operator file.

The `cfcr-compiled-deployment/ops-files` do not provide a simple operator to do this, so let's create a new one to make it easy to create lots of CFCR clusters.

```bash
cat > ~/workspace/cfcr-set-name.yml <<YAML
- type: replace
  path: /name
  value: ((deployment_name))
YAML
```

To create a new `cfcr2` deployment we use the command above, plus our new operator file and `deployment_name` variable:

```plain
cd ~/workspace
export BOSH_DEPLOYMENT=cfcr2

bosh deploy cfcr-compiled-deployment/cfcr.yml \
  -o cfcr-compiled-deployment/ops-files/vm-types.yml \
  -v master_vm_type=default \
  -v worker_vm_type=large \
  -o cfcr-set-name.yml \
  -v deployment_name=$BOSH_DEPLOYMENT
```

Follow the [Using CFCR on AWS](#using-cfcr-on-aws) instructions to register and login to Kubernetes API with `kubectl`, where `BOSH_DEPLOYMENT=cfcr2`.

Throw the new cluster away when you're ready:

```plain
bosh -d cfcr2 delete-deployment
```

### Cleanup BOSH on AWS

One day you might want to discard everything we've just created together. Ok.

First, delete your deployments:

```plain
source <(~/workspace/bucc/bin/bucc env)
bosh deployments
bosh delete-deployment -d cfcr
```

Next, tell your BOSH environment to clean up all orphaned disks/volumes so that you can stop paying AWS for them:

```plain
bosh clean-up --all
```

Finally, use `bucc down` to drop your BOSH environment VM:

```plain
bucc down
```