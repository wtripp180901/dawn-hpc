# capi-helm-fluxcd-config

This repository contains configuration for managing a [Cluster API](https://cluster-api.sigs.k8s.io/)
on [OpenStack](https://www.openstack.org/) cluster using [Flux CD](https://fluxcd.io/).

## Usage

First, create a copy of the repository (like a fork but detached).

```sh
# Clone the repository
git clone https://github.com/stackhpc/capi-helm-fluxcd-config.git my-cluster-config
cd my-cluster-config

# Rename the origin remote to upstream so that we can pull changes in future
git remote rename origin upstream

# Add the new origin remote and push the initial commit
git remote add origin <url>
git push -u origin main
```

Create a cluster configuration under the `./clusters` directory, following the
[example](./clusters/example/).

Part of this process involves creating an application credential in OpenStack and updating the
`credentials.yaml` file. During the bootstrap process, this file will be encrypted using
[kubeseal](https://github.com/bitnami-labs/sealed-secrets) and pushed to the git repository.
Be careful to **NEVER** commit the unencrypted version.

Once you are happy with your cluster configuration, you can bootstrap the cluster.

First, create a Python environment with the required dependencies installed, e.g.:

```sh
python3 -m venv ./.venv
source .venv/bin/activate
pip install -U pip
pip install -r requirements.txt
```

Then run the bootstrap command for the cluster that you created:

```sh
./bin/manage bootstrap my-cluster
```

Once the bootstrap is complete, you will have a cluster that can be managed by making changes
to the git repository.
