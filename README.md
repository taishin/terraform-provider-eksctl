# terraform-provider-eksctl

Manage AWS EKS clusters using Terraform and [eksctl](https://github.com/weaveworks/eksctl).

Benefits:

- `terraform apply` to bring up your whole infrastructure.
- No more generating eksctl `cluster.yaml` with Terraform and a glue shell script just for integration between TF and eksctl.

## Installation

**For Terraform 0.12:**

Install the `terraform-provider-eksctl` binary under `.terraform/plugins/${OS}_${ARCH}`, so that the binary is at e.g. `${WORKSPACE}/.terraform/plugins/darwin_amd64/terraform-provider-eksctl`.

You can also install the provider globally under `${HOME}/.terraform.d/plugins/${OS}_${ARCH}`, so that it is available from all the tf workspaces.

**For Terraform 0.13 and later:**

The provider is [available at Terraform Registry](https://registry.terraform.io/providers/mumoshu/eksctl/latest?pollNotifications=true) so you can just add the following to your tf file for installation:

```
terraform {
  required_providers {
    helmfile = {
      source = "mumoshu/eksctl"
      version = "VERSION"
    }
  }
}
```

Please replace `VERSION` with the version number of the provider without the `v` prefix, like `0.3.14`.

## Usage

There is nothing to configure for the provider, so you firstly declare the provider like:

```
provider "eksctl" {}
```

You use `eksctl_cluster` resource to CRUD your cluster from Terraform.

It's almost like writing and embedding eksctl "cluster.yaml" into `spec` attribute of the Terraform resource definition block, except that some attributes like cluster `name` and `region` has dedicated HCL attributes:

```
resource "eksctl_cluster" "primary" {
  name = "primary"
  region = "us-east-2"

  spec = <<EOS
  - name: ng2
    instanceType: m5.large
    desiredCapacity: 1
EOS
```

On `terraform apply`, the provider runs `eksctl create`, `eksctl update` and `eksctl delete` depending on the situation. It uses `eksctl delete nodegroup --drain` for deleting nodegroups for high availability.

On `terraform destroy`, the provider runs `eksctl delete`

The computed field `output` is used to surface the output from `eksctl`. You can use in the string interpolation to produce a useful Terraform output.

## Advanced Features and Use-cases

There's a bunch more settings that helps the app to stay highly available while being recreated, including:

- `kubernetes_resource_deletion_before_destroy`
- `alb_attachment`
- `pods_readiness_check`

It's also highly recommended to include `git` configuration and use `eksctl` which includes https://github.com/weaveworks/eksctl/pull/2274 in order to install Flux in an unattended way, so that the cluster has everything deployed on launch. Otherwise blue-green deployments of the cluster doesn't make sense.

Please see the [existingvpc](/examples/existingvpc) example to see how a fully configured eksctl_cluster resource should look like, and the below references for details of each setting.
### Delete Kubernetes resources before destroy

Use `kubernetes_resource_deletion_before_destroy` blocks.

It is useful for e.g.:

- Stopping Flux so that it won't try to install new manifests to fail while the cluster is being terminated
- Stopping pods whose IP addresses are exposed via a headless service and external-dns before the cluster being down, so that stale pod IPs won't remain in the serviced discovery system

```hcl
resource "eksctl_cluster" "primary" {
  name = "primary"
  region = "us-east-2"

  spec = <<EOS
  - name: ng2
    instanceType: m5.large
    desiredCapacity: 1
EOS

  kubernetes_resource_deletion_before_destroy {
    namespace = "flux"
    kind = "deployment"
    name = "flux"
  }
}
```

## The Goal

My goal for this project is to allow automated canary deployment of a whole K8s cluster via single `terraform apply` run.

That would require a few additional features to this provider, including:

- [x] Ability to attach `eks_cluster` to ALB
- [ ] Analyze ALB metrics (like 2xx and 5xx count per targetgrous) so that we can postpone `terraform apply` before trying to roll out a broken cluster
- [x] Analyze important pods readiness before rolling out a cluster
  - Implemented. Use `pods_readiness_check` blocks.
- [ ] Analyze Datadog metrics (like request success/error rate, background job sucess/error rate, etc.) before rolling out a new cluster.
- [x] Specify default K8s resource manifests to be applied on the cluster
  - [The new kubernetes provider](https://www.hashicorp.com/blog/deploy-any-resource-with-the-new-kubernetes-provider-for-hashicorp-terraform/) doesn't help it. What we need is ability to apply manifests after the cluster creation but before completing update on the `eks_cluster` resource. With the kubernetes provider, the manifests are applied AFTER the `eksctl_cluster` update is done, which isn't what we want.
  - Implemented. Use the `manifests` attribute.
- [ ] Ability to attach `eks_cluster` to NLB

`terraform-provider-eksctl` is my alternative to the imaginary `eksctl-controller`.

I have been long considered about developing a K8s controller that allows you to manage eksctl cluster updates fully declaratively via a K8s CRD. The biggest pain point of that model is you still need a multi-cluster control-plane i.e. a "management" K8s cluster, which adds additional operational/maintenance cost for us.

If I implement the required functionality to a terraform provider, we don't need an additional K8s cluster for management, as the state is already stored in the terraform state and the automation is aleady done with `Atlantis`, Terraform Enterprise, or any CI systems like CircleCI, GitHub Actions, etc.

As of today, [the API is mostly there](https://github.com/mumoshu/terraform-provider-eksctl/blob/master/pkg/resource/cluster/cluster.go#L132-L210), but the implementation of the functionality is still TODO.

## Developing

If you wish to build this yourself, follow the instructions:

```
$ cd terraform-provider-eksctl
$ go build
```

There's also a convenient Make target for installing the provider into the global tf providers directory:

```
$ make install
```

The above will install the provider's binary under `${HOME}/.terraform.d/plugins/${OS}_${ARCH}`.

## Acknowledgement

The implementation of this product is highly inspired from [terraform-provider-shell](https://github.com/scottwinkler/terraform-provider-shell). A lot of thanks to the author!
