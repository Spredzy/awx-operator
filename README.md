# AWX Operator

An [Ansible AWX](https://github.com/ansible/awx) operator for Kubernetes built with [Operator SDK](https://github.com/operator-framework/operator-sdk) and Ansible.

# Table of Contents

<!--ts-->
* [Purpose](#purpose)
* [Usage](#usage)
  * [Deploying a specific version of AWX](#deploying-a-specific-version-of-awx)
  * [Ingress Types](#ingress-types)
  * [Privilged Tasks](#privileged-tasks)
  * [Database Setup](#database-setup)
    * [External Database](#external-database)
    * [Internal Database](#internal-database)
* [Development](#development)
  * [Testing](#testing)
    * [Testing in Docker](#testing-in-docker)
    * [Testing in Minikube](#testing-in-minikube)
* [Release Process](#release-process)
  * [Build a new release](#build-a-new-release)
  * [Build a new version of the operator yaml file](#build-a-new-version-of-the-operator-yaml-file)
* [Author](#author)
<!--te-->

## Purpose

This operator is meant to provide a more Kubernetes-native installation method for AWX via an AWX Custom Resource Definition (CRD).

Note that the operator is not supported by Red Hat, and is in alpha status. For now, use it at your own risk!

## Usage

This Kubernetes Operator is meant to be deployed in your Kubernetes cluster(s) and can manage one or more AWX instances in any namespace.

First you need to deploy AWX Operator into your cluster:

    kubectl apply -f https://raw.githubusercontent.com/ansible/awx-operator/devel/deploy/awx-operator.yaml

Then you can create instances of AWX, for example:

  1. Make sure the namespace you're deploying into already exists (e.g. `kubectl create namespace ansible-awx`).
  2. Create a file named `my-awx.yml` with the following contents:

     ```
     ---
     apiVersion: awx.ansible.com/v1beta1
     kind: AWX
     metadata:
       name: awx
       namespace: ansible-awx
     spec:
       tower_admin_user: test
       tower_admin_email: test@example.com
       tower_admin_password: changeme
       tower_broadcast_websocket_secret: changeme
     ```

  3. Use `kubectl` to create the awx instance in your cluster:

     ```
     kubectl apply -f my-awx.yml
     ```

After a few minutes, your new AWX instance will be accessible at `http://awx.mycompany.com/` (assuming your cluster has an Ingress controller configured). Log in using the `tower_admin_` credentials configured in the `spec`.


### Deploying a specific version of AWX

To achieve this, please add the following variable under spec within your CR (Custom Resource) file:

```yaml
  tower_image: ansible/awx:15.0.0 # replace this with desired image
```
You may also override any default variables from `roles/awx/defaults/main.yml` using the same process, i.e. by adding those variables within your CR spec.

### Ingress Types

Depending on the cluster that you're running on, you may wish to use an `Ingress` to access AWX, or you may wish to use a `Route` to access your AWX. To toggle between these two options, you can add the following to your AWX CR:

    ---
    spec:
      ...
      tower_ingress_type: Route

OR

    ---
    spec:
      ...
      tower_ingress_type: Ingress
      tower_hostname: awx.mycompany.com

By default, no ingress/route is deployed as the default is set to `none`.

### Privileged Tasks

Depending on the type of tasks that you'll be running, you may find that you need the task pod to run as `privileged`. This can open yourself up to a variety of security concerns, so you should be aware (and verify that you have the privileges) to do this if necessary. In order to toggle this feature, you can add the following to your custom resource:

    ---
    spec:
      ...
      tower_task_privileged: true

If you are attempting to do this on an OpenShift cluster, you will need to grant the `awx` ServiceAccount the `privileged` SCC, which can be done with:

    oc adm policy add-scc-to-user privileged -z awx

Again, this is the most relaxed SCC that is provided by OpenShift, so be sure to familiarize yourself with the security concerns that accompany this action.

### Database Setup

The AWX Operator supports two different scenarios with regard to database setup.

One can either let the Operator manages a PostgreSQL deployment within the same namespace the AWX deployment will take place. Or, one can connect to an already - externally managed - PostgreSQL instance. The decision is made based on the boolean value of the `external_database` ansible variable.

#### External Database

In the scenario when one wants to rely on an external PostgreSQL instance, one needs to provide the necessary information to the operator.

There are three ways one can specify those information. Those way are listed in order of priority in which they are treated.

* Providing a specifically crafted secret

One can create a specially formated secret within the namespace the AWX instance will be deployed into. This secret needs then to be specified as `tower_postgres_configuration_secret` variable in the spec requirements.

```yaml
spec:
  external_database: true
  tower_postgres_configuration_secret: my-awx-external-db-secret
```

The secret format is the following:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: <name-of-the-secret>
  namespace: <namespace>
stringData:
  password: <password>
  username: <username>
  database: <database>
  port: <port>
  host: <host>
```

* Providing variables (not recommended)

If one does not wish to use secret, one can specify the following variables in the spec requirements.

| Variable                | Descritpion                       | Default Value |
| ----------------------- | --------------------------------- | ------------- |
| tower_postgres_user     | Username to connect to PostgreSQL |               |
| tower_postgres_pass     | Password to connect to PostgreSQL |               |
| tower_postgres_database | Database name to connect to       |               |
| tower_postgres_host     | Host to connect to PostgreSQL     |               |
| tower_postgres_port     | Port to connect to PostgreSQL     | 5432          |

The following is an example of an AWX spec section that relie on those variables.

```yaml
spec:
  external_database: true
  tower_postgres_user: awx
  tower_postgres_pass: awx
  tower_postgres_database: awx
  tower_postgres_host: mydb.external.com
  tower_postgres_port: 5432
```

* Relying on '{{ meta.name }}-postgres-configuration' secret

If one wishes not to specify anything and have her data automatically picked up by the operator. One should create a secret named '{{ meta.name }}-postgres-configuration' within the namespace the AWX instance will be deployed in. The secret should look like:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: <awx-instance-name>-postgres-configuration
  namespace: <namespace>
stringData:
  password: <password>
  username: <username>
  database: <database>
  port: <port>
  host: <host>
```

#### Internal Database

If one wants the AWX Operator to manage a PostgreSQL instance along side the AWX instance (this is the default behavior), one can customize the following

| Variable                       | Description                                                       | Default Value                   |
| ------------------------------ | ----------------------------------------------------------------- | ------------------------------- |
| tower_postgres_user            | Username to create on PostgreSQL                                  | awx                             |
| tower_postgres_pass            | Password to associate to the user                                 | awx                             |
| tower_postgres_database        | Database to create on PostgreSQL                                  | awx                             |
| tower_postgres_port            | Port to connect to PostgreSQL                                     | 5432                            |
| tower_postgres_image           | PostgreSQL container to use                                       | postgres:12                     |
| tower_postgres_storage_request | Size of the Persistent Volume desired to be bound to PostgreSQL   | 8Gi                             |
| tower_postgres_storage_class   | Storage class for the Persistent Volume to be bound to PostgreSQL |                                 |
| tower_postgres_data_path       | PostgreSQL mountpoint data path                                   | /var/lib/postgresql/data/pgdata |

For example, if one needs to use a specific storage class for PostgreSQL' storage and expect an heavy use of the instance, one could specify the following

```yaml
---
spec:
  tower_postgres_storage_class: fast-ssd
  tower_postgres_storage_request: 100Gi
```

## Development

### Testing

This Operator includes a [Molecule](https://molecule.readthedocs.io/en/stable/)-based test environment, which can be executed standalone in Docker (e.g. in CI or in a single Docker container anywhere), or inside any kind of Kubernetes cluster (e.g. Minikube).

You need to make sure you have Molecule installed before running the following commands. You can install Molecule with:

    pip install 'molecule[docker]'

Running `molecule test` sets up a clean environment, builds the operator, runs all configured tests on an example operator instance, then tears down the environment (at least in the case of Docker).

If you want to actively develop the operator, use `molecule converge`, which does everything but tear down the environment at the end.

#### Testing in Docker

    molecule test -s test-local

This environment is meant for headless testing (e.g. in a CI environment, or when making smaller changes which don't need to be verified through a web interface). It is difficult to test things like AWX's web UI or to connect other applications on your local machine to the services running inside the cluster, since it is inside a Docker container with no static IP address.

#### Testing in Minikube

    minikube start --memory 8g --cpus 4
    minikube addons enable ingress
    molecule test -s test-minikube

[Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) is a more full-featured test environment running inside a full VM on your computer, with an assigned IP address. This makes it easier to test things like NodePort services and Ingress from outside the Kubernetes cluster (e.g. in a browser on your computer).

Once the operator is deployed, you can visit the AWX UI in your browser by following these steps:

  1. Make sure you have an entry like `IP_ADDRESS  example-awx.test` in your `/etc/hosts` file. (Get the IP address with `minikube ip`.)
  2. Visit `http://example-awx.test/` in your browser. (Default admin login is `test`/`changeme`.)

Alternatively, you can also update the service `awx-service` in your namespace to use the type `NodePort` and use following command to get the URL to access your AWX instance:
```sh
minikube service <serviceName> -n <namespaceName> --url
```

## Release Process

There are a few moving parts to this project:

  1. The Docker image which powers AWX Operator.
  2. The `awx-operator.yaml` Kubernetes manifest file which initially deploys the Operator into a cluster.

Each of these must be appropriately built in preparation for a new tag:

### Build a new release

Run the following command inside this directory:

    operator-sdk build quay.io/ansible/awx-operator:$VERSION

Then push the generated image to Docker Hub:

    docker push quay.io/ansible/awx-operator:$VERSION

### Build a new version of the operator yaml file

Update the awx-operator version:

  - `ansible/group_vars/all`

Once the version has been updated, run from the root of the repo:

    ansible-playbook ansible/chain-operator-files.yml

After it is built, test it on a local cluster:

    minikube start --memory 6g --cpus 4
    minikube addons enable ingress
    kubectl apply -f deploy/awx-operator.yaml
    kubectl create namespace example-awx
    kubectl apply -f deploy/crds/awx_v1beta1_cr.yaml
    <test everything>
    minikube delete

If everything works, commit the updated version, then tag a new repository release with the same tag as the Docker image pushed earlier.

## Author

This operator was originally built in 2019 by [Jeff Geerling](https://www.jeffgeerling.com) and is now maintained by the Ansible Team
