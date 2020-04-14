# ceh-odh

To deploy all the components in OpenShift, the simplest way is to login using oc, e.g.:

```shell
$ oc login -u <USER>
```

Next you can create a project for this demo. We will use `ceh`:

```shell
$ oc new-project ceh
```

## OpenDataHub

We start by installing OpenDataHub via its operator. Start by cloning the operator:

```shell
$ git clone https://gitlab.com/opendatahub/opendatahub-operator
$ cd opendatahub-operator
```
Next, we deploy ODH and Seldon's CRD.

```shell
$ oc create -f deploy/crds/opendatahub_v1alpha1_opendatahub_crd.yaml
$ oc create -f deploy/crds/seldon-deployment-crd.yaml
```

Next, create the services and RBAC policy for the service account the operator will run as. This step at a minimum requires namespace admin rights.

```shell
$ oc create -f deploy/service_account.yaml
$ oc create -f deploy/role.yaml
$ oc create -f deploy/role_binding.yaml
$ oc adm policy add-role-to-user admin -z opendatahub-operator
```

Now we can deploy the operator with:

```shell
$ oc create -f deploy/operator.yaml
```

And wait before the pods are ready before continuing. You can verify using:

```shell
$ oc get pods
```

## Kafka

Strimzi is used to provide Apache Kafka on OpenShift. Start by making a copy of `deploy/crds/opendatahub_v1alpha1_opendatahub_cr.yaml`, e.g.

```shell
$ cp deploy/crds/opendatahub_v1alpha1_opendatahub_cr.yaml ceh-models.yaml
```

and edit the following values:

```yaml
# Seldon Deployment
seldon:
  odh_deploy: true

kafka:
  odh_deploy: true
  kafka_cluster_name: odh-message-bus
  kafka_broker_replicas: 3
  kafka_zookeeper_replicas: 3
```

Kafka installation requires special setup, the following steps are to configure Kafka. Add your username to the `kafka_admins` list, by editing `deploy/kafka/vars/vars.yaml`:

```yaml
kafka_admins:
- admin
- system:serviceaccount:<PROJECT>:opendatahub-operator
- <INSERT USERNAME>
```

You can now deploy Kafka using:

```shell
$ cd deploy/kafka/
$ pipenv install
$ pipenv run ansible-playbook deploy_kafka_operator.yaml \
  -e kubeconfig=$HOME/.kube/config \
  -e NAMESPACE=<PROJECT>
```

Deploy the ODH custom resource based on the sample template

```shell
$ oc create -f ceh-models.yaml
```

## Rook / Ceph

This installation of Rook-Ceph assumes your OCP 3.11/4.x cluster has at least 3 worker nodes. Download the files for Rook Ceph v0.9.3 and modify the source Rook Ceph files directly, clone the Rook operator and checkout the v0.9.3 branch. For convenience we also included the modified files.

```shell
$ git clone https://github.com/rook/rook.git
$ cd rook
$ git checkout -b rook-0.9.3 v0.9.3
$ cd cluster/examples/kubernetes/ceph/
```

Edit `operator.yaml` and set the environment variables for `FLEXVOLUME_DIR_PATH` and `ROOK_HOSTPATH_REQUIRES_PRIVILEGED` to allow the Rook operator to use OpenShift hostpath storage.

```yaml
name: FLEXVOLUME_DIR_PATH
  value: "/etc/kubernetes/kubelet-plugins/volume/exec"
name: ROOK_HOSTPATH_REQUIRES_PRIVILEGED
  value: "true"
```

The following steps require cluster wide permissions. Configure the necessary security contexts , and deploy the rook operator, this will create a new namespace, `rook-ceph-system`, and deploy the pods in it.

```shell
$ oc create -f scc.yaml         # configure security context
$ oc create -f operator.yaml    # deploy operator
```

You can verify deployment is ongoing with:

```shell
$ oc get pods -n rook-ceph-system
NAME READY STATUS RESTARTS AGE
rook-ceph-agent-j4zms 1/1 Running 0 33m
rook-ceph-agent-qghgc 1/1 Running 0 33m
rook-ceph-agent-tjzv6 1/1 Running 0 33m
rook-ceph-operator-567f8cbb6-f5rsj 1/1 Running 0 33m
rook-discover-gghsw 1/1 Running 0 33m
rook-discover-jd226 1/1 Running 0 33m
rook-discover-lgfrx 1/1 Running 0 33m
```

Once the operator is ready, you can create a Ceph cluster, and a Ceph object service. The toolbox service is also handy to deploy for checking the health of the Ceph cluster.

> This step takes a couple of minutes, please be patient.

```shell
$ oc create -f cluster.yaml
```

Check the pods and wait for this pods to finish before proceeding.

```shell
$ oc get pods -n rook-ceph
rook-ceph-mgr-a-66db78887f-5pt7l              1/1     Running     0          108s
rook-ceph-mon-a-69c8b55966-mtb47              1/1     Running     0          3m19s
rook-ceph-mon-b-59699948-4zszh                1/1     Running     0          2m44s
rook-ceph-mon-c-58f4744f76-r8prn              1/1     Running     0          2m11s
rook-ceph-osd-0-764bbd9694-nxjpz              1/1     Running     0          75s
rook-ceph-osd-1-85c8df76d7-5bdr7              1/1     Running     0          74s
rook-ceph-osd-2-8564b87d6c-lcjx2              1/1     Running     0          74s
rook-ceph-osd-prepare-ip-10-0-136-154-mzf66   0/2     Completed   0          87s
rook-ceph-osd-prepare-ip-10-0-153-32-prf94    0/2     Completed   0          87s
rook-ceph-osd-prepare-ip-10-0-175-183-xt4jm   0/2     Completed   0          87s
```

Edit `object.yaml` and replace port `80` with `8080`:

```yaml
gateway:
  # type of the gateway (s3)
  type: s3
  # A reference to the secret in the rook namespace where the ssl certificate is stored
  sslCertificateRef:
  # The port that RGW pods will listen on (http)
  port: 8080
```

And then run:

```shell
$ oc create -f toolbox.yaml
$ oc create -f object.yaml
```

You can check the deployment progress, as previously, with:

```shell
$ oc get pods -n rook-ceph
rook-ceph-mgr-a-5b6fcf7c6-cx676 1/1 Running 0 6m56s
rook-ceph-mon-a-54d9bc6c97-kvfv6 1/1 Running 0 8m38s
rook-ceph-mon-b-74699bf79f-2xlzz 1/1 Running 0 8m22s
rook-ceph-mon-c-5c54856487-769fx 1/1 Running 0 7m47s
rook-ceph-osd-0-7f4c45fbcd-7g8hr 1/1 Running 0 6m16s
rook-ceph-osd-1-55855bf495-dlfpf 1/1 Running 0 6m15s
rook-ceph-osd-2-776c77657c-sgf5n 1/1 Running 0 6m12s
rook-ceph-osd-3-97548cc45-4xm4q 1/1 Running 0 5m58s
rook-ceph-osd-prepare-ip-10-0-138-84-gc26q 0/2 Completed 0 6m29s
rook-ceph-osd-prepare-ip-10-0-141-184-9bmdt 0/2 Completed 0 6m29s
rook-ceph-osd-prepare-ip-10-0-149-16-nh4tm 0/2 Completed 0 6m29s
rook-ceph-osd-prepare-ip-10-0-173-174-mzzhq 0/2 Completed 0 6m28s
rook-ceph-rgw-my-store-d6946dcf-q8k69 1/1 Running 0 5m33s
rook-ceph-tools-cb5655595-4g4b2 1/1 Running 0 8m46s
```

Next, you will need to create a set of S3 credentials, the resulting credentials will be stored in a secret file under the `rook-ceph` namespace. There isn’t currently a way to cross-share secrets between OpenShift namespaces, so you will need to copy the secret to the namespace running Open Data Hub operator. To do so, run:

```shell
$ oc create -f object-user.yaml
```

Next we are going to retrieve the secrets using

```shell
$ oc get secrets -n rook-ceph rook-ceph-object-user-my-store-my-user -o json
```

Create a secret in your deployment namespace that includes the secret and key for S3 interface. Make sure to copy the `accesskey` and `secretkey` from the command output above and replace it in the file available in this repo in `deploy/ceph/s3-secretceph.yml.`. Then you should run:

```shell
$ oc create -n ccfd -f deploy/ceph/s3-secretceph.yaml
```

### Route

From the Openshift console, create a route to the rook service, `rook-ceph-rgw-my-store`, in the `rook-ceph` namespace to expose the endpoint. This endpoint url will be used to access the S3 interface from the example notebooks.

```shell
$ oc expose -n rook-ceph svc/rook-ceph-rgw-my-store
```

## Fraud detection model

Deploy fraud detection fully trained model by using `deploy/model/eh-seldon-models.json` in this repository:

```shell
$ oc create -n <NAMESPACE> -f deploy/model/ceh-seldon-models.json
```

Check and make sure the model is created, this step will take a couple of minutes.

```shell
$ oc get seldondeployments
$ oc get pods
```

Create a route to the model by using `deploy/model/ceh-seldon-models-route.yaml` in this repo:

```shell
$ oc create -n <NAMESPACE> -f deploy/model/ceh-seldon-models-route.yaml
```

Enable Prometheus metric scraping by editing `ceh-seldon-models-ceh-seldon-models` service from the portal and adding these two lines under annotations:

```yaml
apiVersion: v1
kind: Service
metadata:
annotations:
    prometheus.io/path: /prometheus
    prometheus.io/scrape: 'true'
```

## Upload data to Rook-Ceph

Make sure to decode the key and secret copied from the rook installation by using the following commands:

```shell
$ base64 -d
<Paste secret>
[Ctrl-D]
```

From a command line use the aws tool to upload the file to `rook-ceph` data store:

```shell
$ aws configure
```

Only enter key and secret, leave all other fields as default. Check if connection is working using the route created previously (you can use `oc get route -n rook-ceph`):

```shell
$ aws s3 ls --endpoint-url <ROOK_CEPH_URL>
```

Create a bucket to upload the file:

```shell
$ aws s3api create-bucket --bucket data --endpoint-url <ROOK_CEPH_URL>
```

Download the training dataset `data.csv` file (available [here](https://raw.githubusercontent.com/ruivieira/ceh-seldon-models/master/data/data.csv)) and upload it using:

```shell
$ wget -O data.csv https://raw.githubusercontent.com/ruivieira/ceh-seldon-models/master/data/data.csv
$ aws s3 cp data.csv s3://data/OPEN/uploaded/data.csv --endpoint-url <ROOK_CEPH_URL> --acl public-read-write
```

You can verify the file is uploaded using:

```shell
$ aws s3 ls s3://data/OPEN/uploaded/ --endpoint-url <ROOK_CEPH_URL>
```

## Notebook

Set the following variables S3 credentials as the Base64 encoded previously:

- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

Set `ENDPOINT_URL` as the exposed Ceph URL previously.