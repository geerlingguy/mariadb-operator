# MariaDB Operator for Kubernetes

[![Build Status](https://travis-ci.com/geerlingguy/mariadb-operator.svg?branch=master)](https://travis-ci.com/geerlingguy/mariadb-operator)

This is a MariaDB Operator, which makes management of MariaDB instances (or clusters) running inside Kubernetes clusters easy. It was built with the [Operator SDK](https://github.com/operator-framework/operator-sdk) using [Ansible](https://www.ansible.com/blog/ansible-operator).

## Usage

This Kubernetes Operator is meant to be deployed in your Kubernetes cluster(s) and can manage one or more MariaDB database instances or clusters in any namespace.

First you need to deploy MariaDB Operator into your cluster:

    kubectl apply -f https://raw.githubusercontent.com/geerlingguy/mariadb-operator/master/deploy/mariadb-operator.yaml

Then you can create instances of MariaDB in any namespace, for example:

  1. Create a file named `my-database.yml` with the following contents:

     ```
     ---
     apiVersion: mariadb.mariadb.com/v1alpha1
     kind: MariaDB
     metadata:
       name: example-mariadb
       namespace: example-mariadb
     spec:
       masters: 1
       replicas: 0
       mariadb_password: CHANGEME
       mariadb_image: mariadb:10.4
       mariadb_pvc_storage_request: 1Gi
     ```

  2. Use `kubectl` to create the MariaDB instance in your cluster:

     ```
     kubectl apply -f my-database.yml
     ```

> You can also deploy `MariaDB` instances into other namespaces by changing `metadata.namespace`, or deploy multiple `MariaDB` instances into the same namespace by changing `metadata.name`.

### Connecting to the running database

Once the database instance has been deployed and initialized (this can take up to a few minutes), you can connect to it using the MySQL CLI from within the same namespace as your database instance, for example:

    kubectl -n example-mariadb run -it --rm mysql-client --image=arey/mysql-client --restart=Never -- -h example-mariadb-0.example-mariadb.example-mariadb.svc.cluster.local -u db_user -pCHANGEME -D db

Other applications can connect to the database instance from within the same namespace using the following connection parameters:

  - Host: `example-mariadb-0.example-mariadb.example-mariadb.svc.cluster.local`
  - Username: `db_user`
  - Password: `CHANGEME`
  - Database: `db`

### Exposing an instance to the outside world

You can also expose a database instance to the outside world by adding a `Service` to connect to port `3360`:

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: mariadb-master
      namespace: example-mariadb
    spec:
      type: NodePort
      selector:
        statefulset.kubernetes.io/example-mariadb: example-mariadb-0
      ports:
      - protocol: TCP
        port: 3360
        targetPort: 3360

Once you create that service in the same namespace, you can connect to MariaDB via the NodePort assigned to the service.

## Development

### Release Process

There are a few moving parts to this project:

  1. The Docker image which powers MariaDB Operator.
  2. The `mariadb-operator.yaml` Kubernetes manifest file which initially deploys the Operator into a cluster.

Each of these must be appropriately built in preparation for a new tag:

#### Build a new release of the Operator for Docker Hub

Run the following command inside this directory:

    operator-sdk build geerlingguy/mariadb-operator:0.0.3

Then push the generated image to Docker Hub:

    docker push geerlingguy/mariadb-operator:0.0.3

#### Build a new version of the `mariadb-operator.yaml` file

Verify the `build/chain-operator-files.yml` playbook has the most recent version/tag of the Docker image, then run the playbook in the `build/` directory:

    ansible-playbook chain-operator-files.yml

After it is built, test it on a local cluster:

    minikube start
    minikube addons enable ingress
    kubectl apply -f deploy/mariadb-operator.yaml
    kubectl create namespace example-mariadb
    kubectl apply -f deploy/crds/mariadb_v1alpha1_mariadb_cr.yaml
    <test everything>
    minikube delete

If everything is deployed correctly, commit the updated version and push it up to GitHub, tagging a new repository release with the same tag as the Docker image.

### Testing

#### Local tests with Molecule and KIND

Ensure you have the testing dependencies installed (in addition to Docker):

    pip install docker molecule openshift jmespath

Run the local molecule test scenario:

    molecule test -s test-local

#### Local tests with minikube

TODO.

## Authors

This operator is maintained by [Jeff Geerling](https://www.jeffgeerling.com), author of [Ansible for DevOps](https://www.ansiblefordevops.com) and [Ansible for Kubernetes](https://www.ansibleforkubernetes.com).
