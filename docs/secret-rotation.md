# Secret Rotation

This brief guide outlines all the steps required to rotate the OpenStack application credential for a FluxCD GitOps-managed CAPI Helm cluster.

## Contents

### [Pre-requisites](#pre-requisites-1)
- [Applications & Software](#applications--software)

### [Outline](#outline-1)
- [Cluster Maintenance](#cluster-maintenance)
  - Basics of what is done in a standard secret rotation.
- [Security Threat](#security-threat)
  - What to do in the case of a compromised secret.

### [Horizon Application Credentials](#horizon-application-credentials-1)
- [Create/Delete Credentials](#createdelete-credentials)
  - How to create & delete application credentials within Horizon.

### [Processing Secrets](#processing-secrets-1)
- [Encrypting Credentials](#encrypting-new-credential)
  - Using Kubeseal to encrypt the newly created credentials.
- [Updating Secrets](#updating-secrets)
  - How to rotate new secrets using FluxCD's GitOps.
- [Deleting Old Secrets](#deleting-old-secrets)
  - Some housekeeping steps post secret rotation.
- [Rotating Compromised Secrets](#rotating-compromised-secrets)
  - Summary for the rotation of secrets posing a security threat.

### [Validate Configuration](#validate-configuration-1)
- [Pod Status](#pod-status)
  - How to check the state of pods post secret rotation.
- [Sonobuoy](#sonobuoy)
  - Brief outline on validating the Kubernetes configuration.


# Pre-Requisites

This document applies to FluxCD-managed Cluster API (CAPI) clusters deployed using the tooling found in this repository (see the repo's main [README.md](../README.md) for an introduction).

The following are required to rotate the cloud credential on an existing cluster:

  * The `kubeconfig` for the target cluster (written to `clusters/<cluster-name>/kubeconfig` in the local repo checkout on initial cluster deployment)
  * An OpenStack account with access to the target project
  * [git](https://git-scm.com/)
  * [kubeseal](https://github.com/bitnami-labs/sealed-secrets?tab=readme-ov-file#kubeseal)
  * [flux CLI](https://fluxcd.io/flux/cmd/)


# Outline

The overall idea is to take advantage of Kubeseal's encryption, allowing us to upload secrets to our repositories; meaning that after a branch containing the new secrets is merged into the branch being tracked by FluxCD, the FluxCD GitOps controllers on the cluster will update the current cluster deployment to reflect the newly merged branch.

Although the basic outline will remain the same, there are some variations in the preliminary actions depending on the situation and reason why secrets are being rotated.

The two main categories fall under **cluster maintenance** and **security threat**, and the differences in these situations will be further explained below.

### Cluster Maintenance

It is often a good idea and good security practice to rotate passwords and secrets, as it reduces the chances of them existing long enough to be exploited or end up in the wrong hands; often many secrets will have an expiration date precisely for this reason.

Therefore, in the case of the user simply updating secrets for the purpose of cluster maintenance it is sufficient to replace the current Kubeseal encrypted secret with an updated, `kubeseal`-encrypted secret via a GitHub pull request. The old application credential should then be deleted from OpenStack.

###	Security Threat

In the event that the existence of a secret is a cause for concern, such as the current credentials being leaked, then the protocol is largely the same apart from needing to delete the current application credential first before any other step. At which point the compromised secret is no longer valid and anyone using it won't be able to do anything with it.

# Horizon Application Credentials

### Create/Delete Credentials

The first step is to access the OpenStack Horizon dashoard in order to either create a new application credential or to delete the old, compromised secret.

Application credentials can be managed using the OpenStack CLI ([relevant docs](https://docs.openstack.org/keystone/2024.2/user/application_credentials.html)); however, for the sake of simplicity, the following steps will outline how to do this through the Horizon dashboard user interface.

Once logged into Horizon, head to the _Identity_ tab on the left, then _Application Credentials_.

> [!IMPORTANT]
> If the application credential been compromised, this is the point at which the it will need to be deleted.

In the case of creating a new application credential, please remember to download the `clouds.yaml` once the configuration steps are complete; this will be the only time you will be able to do so, otherwise a new app cred will need to be created.

At this point, the newly created secret requires encrypting before it can be rotated into the existing cluster.

# Processing Secrets

### Encrypting New Credential

Once the new secret's `clouds.yaml` has been downloaded, create a new file at ``clusters/<cluster-name>/credentials.yaml` (or replace it if it already exists) with the following structure:

```
apiVersion: v1
kind: Secret
metadata:
  name: <MAKE SURE THIS MATCHES “cloudCredentialsSecretName:” IN configmap.yaml>
  namespace: capi-self
  annotations:
    # Allow the sealed secret controller to take over this secret after bootstrapping
    sealedsecrets.bitnami.com/managed: "true"
  labels:
    # Tell the cluster-api-addon-provider to watch this resource for
    # changes and update any Manifest or HelmRelease addon resources
    # which refer to it whenever this secret is changed / rotated.
    addons.stackhpc.com/watch: "true"
stringData:
  clouds.yaml: |
    <INSERT THE CONTENTS OF YOUR NEW SECRET'S CLOUDS.YAML, PAYING ATTENTION TO THE
    INDENTATIONS>
```

Once the newly created **clouds.yaml**'s contents have been copied into **credentials.yaml**, it should look something like this:

```
apiVersion: v1
kind: Secret
metadata:
  name: <MAKE SURE THIS MATCHES “cloudCredentialsSecretName:” IN configmap.yaml>
  namespace: capi-self
  annotations:
    # Allow the sealed secret controller to take over this secret after bootstrapping
    sealedsecrets.bitnami.com/managed: "true"
  labels:
    # Tell the cluster-api-addon-provider to watch this resource for
    # changes and update any Manifest or HelmRelease addon resources
    # which refer to it whenever this secret is changed / rotated.
    addons.stackhpc.com/watch: "true"
stringData:
  clouds.yaml: |
    clouds:
      openstack:
        auth:
          auth_url: ...
          application_credential_id: ...
          application_credential_secret: ...
        interface: ...
        identity_api_version: 3
        auth_type: ...
```

Next, make sure that the terminal's current working directory contains the new `credentials.yaml`, this should be `clusters/<cluster-name>/` from the repo root, then run the following Kubeseal command to encrypt the secret:

```sh
kubeseal \
  --kubeconfig kubeconfig \
  --format yaml \
  --controller-name sealed-secrets \
  --controller-namespace sealed-secrets-system \
  --secret-file credentials.yaml  \
  --sealed-secret-file credentials-sealed.yaml
```

> [!NOTE]
> This will create a new file named `credentials-sealed.yaml` containing the encrypted contents. The unencrypted `credentials.yaml` file is [gitignored](https://github.com/stackhpc/capi-helm-fluxcd-config/blob/main/.gitignore#L7-L8) to avoid accidentally pushing the unencrypted secret to the remote git repo.

### Updating Secrets

With the new secret encrypted, it is time to change the sealed secret which is on the Kubernetes cluster.

The process of changing the secret over is as simple as committing and pushing the new **credentials-sealed.yaml** file to the remote git repository which is being tracked by FluxCD.

> [!TIP]
> It is highly recommended to `git switch -c` to a new branch when pushing the new sealed credentials so that a pull request can be reviewed before merging to the tracked branch.

When the updated **credentials-sealed.yaml** file has been merged into the FluxCD tracked repository branch, FluxCD will try to update the **sealed secret** on the cluster once it recognises that there are differences between the cluster's current configuration and the one in the remote git repository.

> [!NOTE]
> The time interval between FluxCD reconciliations of current cluster state vs git repo state is set in `components/cluster/helmrelease.yaml` under the `spec.interval` variable.

This should result in the sealed secret on the Kubernetes cluster containing the new secret. The `sealed-secrets` controller on the cluster will then convert this to a standard k8s secret which can be [decoded](https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/#decoding-secret) using `kubectl` to check that the contents has been updated correctly.

Once the sealed secret has been updated on your Kubernetes cluster it is time to do some housekeeping by deleting the old application credential (using Horizon or the OpenStack CLI).

###	Rotating Compromised Secrets

As mentioned above, an application credential which is a security concern should be deleted before any other step; after which a new application credential, which will replace it, can be created.

> [!IMPORTANT]
> Don't forget to download the `clouds.yaml` file after creating the new application credential!

```
apiVersion: v1
kind: Secret
metadata:
  name: <MAKE SURE THIS MATCHES “cloudCredentialsSecretName:” IN configmap.yaml>
  namespace: capi-self
  annotations:
    # Allow the sealed secret controller to take over this secret after bootstrapping
    sealedsecrets.bitnami.com/managed: "true"
  labels:
    # Tell the cluster-api-addon-provider to watch this resource for
    # changes and update any Manifest or HelmRelease addon resources
    # which refer to it whenever this secret is changed / rotated.
    addons.stackhpc.com/watch: "true"
stringData:
  clouds.yaml: |
    clouds:
      openstack:
        auth:
          auth_url: ...
          application_credential_id: ...
          application_credential_secret: ...
        interface: ...
        identity_api_version: 3
        auth_type: ...
```

Then, like the previous secret rotation, use Kubeseal to encrypt the new secret:

```sh
kubeseal \
  --kubeconfig kubeconfig \
  --format yaml \
  --controller-name sealed-secrets \
  --controller-namespace sealed-secrets-system \
  --secret-file credentials.yaml  \
  --sealed-secret-file credentials-sealed.yaml
```

Again, this file should then be committed and pushed to a new remote branch which can then be reviewed and merged into the branch which FluxCD is tracking for changes. After which, once per interval, FluxCD will compare the current configuration with the branch on GitHub, and upon noticing any differences will attempt to update the current cluster's configuration to reflect the changes.

# Validate Configuration

It is always a good idea to test your deployments for any potential errors, which can lead to more issues further down the line. Therefore, it is worthwhile to validate the Kubernetes cluster both with a diagnostic tool and by checking the pods' ```STATUS```.

###	Pod Status

A quick and simple way of checking the results of the secret rotation is to check the ```STATUS``` and, if necessary, logs of the pods. This can be done by running

```sh
kubectl get pods -A
```

then checking for any pods which may be labelled with ```Error``` or ```CrashLoopBackOff```. If this is the case check the logs of that pod by running

```sh
kubectl logs <pod-name> -n <namespace>
```

In particular, it is worth checking the logs of the various pods in the `openstack-system` and `capo-system` namespaces since these are the processses which actually use the new OpenStack application credential to interact with the OpenStack APIs. By default, Kubernetes does not update the internal state of pods which mount secrets (or config maps); therefore, in order for the aforementioned pods to pick up the new application credentials secret it may be necessary to restart the pods. This can be achieved with

```sh
kubectl -n capo-system rollout restart deployment/capo-controller-manager
kubectl -n openstack-system rollout restart deployment/openstack-cinder-csi-controllerplugin
kubectl -n openstack-system rollout restart daemonset/openstack-cloud-controller-manager
kubectl -n openstack-system rollout restart daemonset/openstack-cinder-csi-nodeplugin
```

For general help with debugging Cluster API clusters please visit our debugging guide [here](https://github.com/azimuth-cloud/capi-helm-charts/blob/main/charts/openstack-cluster/DEBUGGING.md).

###	**Sonobuoy**

The official documentation can be found [here](https://sonobuoy.io/docs/v0.57.1/), however a brief outline of testing your cluster with Sonobuoy is included here.

Once Sonobuoy has been installed, change to the `clusters/<cluster-name>` directory. From there, a **single** conformance test can be initiated with:

```
sonobuoy run --kubeconfig kubeconfig --wait --mode quick
```

Once complete, export the results with:

```
results=$(sonobuoy retrieve)
```

Then inspect them with:

```
sonobuoy results $results
```

This will display various information, however, all that is of interest at this point are the details under **Run Details** which should, hopefully, report 100% **Node** and **Pod** health.

The ```results``` command has many useful options, for which further information can be found [here](https://sonobuoy.io/docs/v0.57.1/results/).
