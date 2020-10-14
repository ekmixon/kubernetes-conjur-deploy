# kubernetes-conjur-deploy

This repository contains scripts for automating the deployment of Conjur
followers to a Kubernetes or OpenShift environment. These scripts can also be
used to deploy a full cluster with Master and Standbys for testing and demo
purposes but this is not recommended for a production deployment of Conjur.

**This repo supports CyberArk DAP v10+**. To deploy Conjur OSS, please use
the [Conjur OSS helm chart](https://github.com/cyberark/conjur-oss-helm-chart).

---

# Setup

The Conjur deployment scripts pick up configuration details from local
environment variables. The setup instructions below walk you through the
necessary steps for configuring your environment and show you which variables
need to be set before deploying.

All environment variables can be set/defined with:
- `bootstrap.env` file if deploying the follower to Kubernetes or OpenShift
- `dev-bootstrap.env` for all other configurations.

Edit the values per instructions below, source the appropriate file and run
`0_check_dependencies.sh` to verify.

The Conjur appliance image can be loaded with `_load_conjur_tarfile.sh`. The script uses environment variables to locate the tarfile image and the value to use as a tag once it's loaded.

### Conjur Configuration

_Note: If you are using Conjur v4, please use [v4_support](https://github.com/cyberark/kubernetes-conjur-deploy/tree/v4_support)
branch of this repo!_

#### Appliance Image

You need to obtain a Docker image of the Conjur appliance and push it to an
accessible Docker registry. Provide the image and tag like so:

```
export CONJUR_APPLIANCE_IMAGE=<tagged-docker-appliance-image>
```

You will also need to provide an ID for the Conjur authenticator that will later
be used in [Conjur policy](https://developer.conjur.net/policy) to provide your
apps with access to secrets through Conjur:

```
export AUTHENTICATOR_ID=<authenticator-id>
```

This ID should describe the cluster in which Conjur resides. For example, if
you're hosting your dev environment on GKE you might use `gke/dev`.

#### Follower Seed

You will need to provide a follower seed file generated from your Conjur Master.
The seed can be generated by SSH-ing into your Master and running:

*NOTE: If you are running this code to deploy a follower that will run in a separate
cluster from the master, you _must_ force-generate the follower certificate manually
before creating the seed to prevent the resulting certificate from omitting the
future in-cluster subject altname included.*

To generate a follower seed with the appropriate in-cluster subject altname for followers
that are not in the same cluster as master, we first will need to issue a certificate
on master. Skip this step if master is collocated with the follower.
```
$ evoke ca issue --force <follower_external_fqdn> conjur-follower.<conjur_namespace_name>.svc.cluster.local
```

We now need to create the seed archive with the proper information:

```
$ evoke seed follower <follower_external_fqdn> > /tmp/follower-seed.tar
```

If you are on the same node as the master container, you can also export the seed with:
```
$ sudo docker exec <container_id> evoke seed follower <follower_external_fqdn> > /tmp/follower-seed.tar
```
Note: the exported seed file will not be copied to host properly if you use `-t` flag with the
`docker exec` command.

Copy the resulting seed file from the Conjur master to your local filesystem and
set the following environment variable to point to it:

```
export FOLLOWER_SEED=path/to/follower/seed/file
```

The deploy scripts will copy the seed to your follower pods and use it to
configure them as Conjur followers.

*Important note*: Make sure to delete any copies of the seed after use. It
contains sensitive information and can always be regenerated on the Master.

### Platform Configuration

If you are working with OpenShift, you will need to set:

```
export PLATFORM=openshift
export OSHIFT_CLUSTER_ADMIN_USERNAME=<name-of-cluster-admin> # system:admin in minishift
export OSHIFT_CONJUR_ADMIN_USERNAME=<name-of-conjur-namespace-admin> # developer in minishift
```

Otherwise, `$PLATFORM` variable will default to `kubernetes`.

Before deploying Conjur, you must first make sure that you are connected to your
chosen platform with a user that has the `cluster-admin` role. The user must be
able to create namespaces and cluster roles.

#### Conjur Namespace

Provide the name of a namespace in which to deploy Conjur:

```
export CONJUR_NAMESPACE_NAME=<my-namespace>
```

#### The `conjur-authenticator` Cluster Role

Conjur's Kubernetes authenticator requires the following privileges:

- [`"get"`, `"list"`] on `"pods"` for confirming a pod's namespace membership
- [`"create"`, `"get"`] on "pods/exec" for injecting a certificate into a pod

The deploy scripts include a manifest that defines the `conjur-authenticator`
cluster role, which grants these privileges. Create the role now (note that
your user will need to have the `cluster-admin` role to do so):

```
# Kubernetes
kubectl apply -f ./kubernetes/conjur-authenticator-role.yaml

# OpenShift
oc apply -f ./openshift/conjur-authenticator-role.yaml
```

### Docker Configuration

[Install Docker](https://www.docker.com/get-docker) from version 17.05 or higher on your local machine if you
do not already have it.


#### Kubernetes

You will need to provide the domain and any additional pathing for the Docker
registry from which your Kubernetes cluster pulls images:

```
export DOCKER_REGISTRY_URL=<registry-domain>
export DOCKER_REGISTRY_PATH=<registry-domain>/<additional-pathing>
```

Note that the deploy scripts will be pushing images to this registry so you will
need to have push access.

If you are using a private registry, you will also need to provide login
credentials that are used by the deployment scripts to create a [secret for
pulling images](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#create-a-secret-in-the-cluster-that-holds-your-authorization-token):

```
export DOCKER_USERNAME=<your-username>
export DOCKER_PASSWORD=<your-password>
export DOCKER_EMAIL=<your-email>
```

Please make sure that you are logged in to the registry before deploying.

### Running Kubernetes locally

You can now deploy a local development environment for Kubernetes using [Docker Desktop](https://www.docker.com/products/docker-desktop).
See our [contributing guide][CONTRIBUTING.md] to learn how!

#### OpenShift

OpenShift users should make sure the [integrated Docker registry](https://docs.okd.io/latest/install_config/registry/deploy_registry_existing_clusters.html)
in your OpenShift environment is available and that you've added it as an
[insecure registry](https://docs.docker.com/registry/insecure/) in your local
Docker engine. You must then specify the path to the OpenShift registry like so:

```
export DOCKER_REGISTRY_PATH=docker-registry-<registry-namespace>.<routing-domain>
```

Please make sure that you are logged in to the registry before deploying.

### Running OpenShift in Minishift

You can use Minishift to run OpenShift locally in a single-node cluster. Minishift provides a convenient way to test out Conjur deployments on a laptop or local machine and also provides an integrated Docker daemon from which to stage and push images into the OpenShift registry. The `./openshift` subdirectory contains two files:
 * `_minishift-boot.env` that defines environment variables to configure Minishift, and
 * `_minishift-start.sh` to startup Minishift.
The script assumes VirtualBox as the hypervisor but others are supported. See https://github.com/minishift/minishift for more information.

Steps to startup Minishift:

1. Ensure VirtualBox is installed
1. `cd openshift`
1. Run `./minishift-start.sh`
1. `source minishift.env` to gain use of the internal Docker daemon
1. `cd ..`
1. Use `dev-bootstrap.env` for your variable configuration
1. Run `./start`

---

# Usage

## Deploying Conjur Follower

Ensure that `bootstrap.env` has the `FOLLOWER_SEED` variable set to the seed
file created [here](#follower-seed) or a URL to the seed service.

If master key encryption is used in the cluster, `CONJUR_DATA_KEY` must be set to
the path to a file that contains the encryption key to use when configuring the
follower.

By default, the follower will store all data within the container.  If
`FOLLOWER_USE_VOLUMES` is set to `true`, the follower will use host volumes (not
persistent volumes) for `/var/log/conjur`, `/var/log/nginx` and
`/var/lib/postgresql/10`.

After verifying this setting, source `./bootstrap.env` and then run `./start` to
execute the scripts necessary to have the follower deployed in your environment.

---

# Test App Demo

The [kubernetes-conjur-demo repo](https://github.com/conjurdemos/kubernetes-conjur-demo)
deploys test applications that retrieve secrets from Conjur and serves as a
useful reference when setting up your own applications to integrate with Conjur.


# Contributing

We welcome contributions of all kinds to this repository. For instructions on how to get started and descriptions of our development workflows, please see our [contributing
guide][contrib].

[contrib]: CONTRIBUTING.md

# License

This repository is licensed under Apache License 2.0 - see [`LICENSE`](LICENSE) for more details.

