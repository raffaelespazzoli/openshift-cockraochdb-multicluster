# CockroachDB on OCP

## CockroachDB on Single OCP cluster

Deploy cert-manager

```shell
oc new-project cert-manager
oc apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.0/cert-manager.yaml
```

Deploy local CA

```shell
oc new-project cockroachdb
oc apply -f ./cert-issuer.yaml -n cockroachdb
```

```shell
helm template cockroachdb ./cockroachdb-prep -n cockroachdb -f single-cluster-values.yaml | oc apply -f - -n cockroachdb
helm template cockroachdb ./cockroachdb -n cockroachdb -f single-cluster-values.yaml | oc apply -f - -n cockroachdb
```

### Verify the cluster

```shell
export tools_pod=$(oc get pods | grep tools | awk '{print $1}')
oc exec $tools_pod -c tools -n cockroachdb -- /cockroach/cockroach init --certs-dir=/crdb-certs --host cockroachdb-0.cockroachdb
oc exec $tools_pod -c tools -n cockroachdb -- /cockroach/cockroach node status --certs-dir=/crdb-certs --host cockroachdb-0.cockroachdb
```

## Emulating 3 clusters

Deploy cert-manager

```shell
oc new-project cert-manager
oc apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.0/cert-manager.yaml
```

Deploy cert-utils operator

```shell
oc new-project cert-utils-operator
oc apply -f ./cert-utils-operator.yaml
```

Create namespaces

```shell
oc new-project cockroachdb-1
oc new-project cockroachdb-2
oc new-project cockroachdb-3
```

Create CAs

```shell
oc apply -f ./cert-issuer.yaml -n cockroachdb-1
oc get secret rootca -n cockroachdb-1 -o jsonpath='{.data.ca\.crt}' | base64 -d > /tmp/ca.crt
oc get secret rootca -n cockroachdb-1 -o jsonpath='{.data.tls\.key}' | base64 -d > /tmp/tls.key
oc get secret rootca -n cockroachdb-1 -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/tls.crt

oc create secret tls rootca -n cockroachdb-2 --cert=/tmp/tls.crt --key=/tmp/tls.key
oc create secret tls rootca -n cockroachdb-3 --cert=/tmp/tls.crt --key=/tmp/tls.key
oc patch secret rootca -n cockroachdb-2 --patch '{"data":{"ca.crt":"'$(cat /tmp/ca.crt | base64 -w 0)'"}}'
oc patch secret rootca -n cockroachdb-3 --patch '{"data":{"ca.crt":"'$(cat /tmp/ca.crt | base64 -w 0)'"}}'
oc apply -f cert-issuer-secondary-sites.yaml -n cockroachdb-2
oc apply -f cert-issuer-secondary-sites.yaml -n cockroachdb-3
```

Create certs and tools container

```shell
helm template cockroachdb ./cockroachdb-prep -n cockroachdb-1 -f three-clusters-emulation-values.yaml --set conf.locality=namespace=cockroachdb-1 | oc apply -f - -n cockroachdb-1
helm template cockroachdb ./cockroachdb-prep -n cockroachdb-2 -f three-clusters-emulation-values.yaml --set conf.locality=namespace=cockroachdb-2 | oc apply -f - -n cockroachdb-2
helm template cockroachdb ./cockroachdb-prep -n cockroachdb-3 -f three-clusters-emulation-values.yaml --set conf.locality=namespace=cockroachdb-3 | oc apply -f - -n cockroachdb-3
```

create crdb cluster

```shell
helm template cockroachdb ./cockroachdb -n cockroachdb-1 -f three-clusters-emulation-values.yaml --set conf.locality=namespace=cockroachdb-1 | oc apply -f - -n cockroachdb-1
helm template cockroachdb ./cockroachdb -n cockroachdb-2 -f three-clusters-emulation-values.yaml --set conf.locality=namespace=cockroachdb-2 | oc apply -f - -n cockroachdb-2
helm template cockroachdb ./cockroachdb -n cockroachdb-3 -f three-clusters-emulation-values.yaml --set conf.locality=namespace=cockroachdb-3 | oc apply -f - -n cockroachdb-3
```

### Start and Verify the cluster

```shell
export tools_pod=$(oc get pods -n cockroachdb-1 | grep tools | awk '{print $1}')
oc exec $tools_pod -c tools -n cockroachdb-1 -- /cockroach/cockroach init --certs-dir=/crdb-certs --host cockroachdb-0.cockroachdb
oc exec $tools_pod -c tools -n cockroachdb-1 -- /cockroach/cockroach node status --certs-dir=/crdb-certs --host cockroachdb-0.cockroachdb
oc exec $tools_pod -c tools -n cockroachdb-1 -- /cockroach/cockroach sql --execute='CREATE USER raffa WITH PASSWORD raffa;' --certs-dir=/crdb-certs --host cockroachdb-0.cockroachdb
oc exec $tools_pod -c tools -n cockroachdb-1 -- /cockroach/cockroach sql --execute='GRANT admin TO raffa WITH ADMIN OPTION;' --certs-dir=/crdb-certs --host cockroachdb-0.cockroachdb
```

Connecting to the ui

```shell
export ui_url=$(oc get route cockroachdb -n cockroachdb-1 -o jsonpath='{.spec.host}')
echo $ui_url
```

connect to $ui_url user raffa/raffa

## Deploy on three clusters

### Create the clusters

here we create the clusters with hive. Other means of creating the clusters are perfectly fine. the only thing that matters is that at the end the context are correct set.

Deploy the hive operator

```shell
```

Variable preparation

replace where appropriately with your values.

```shell
export ssh_key=$(cat ~/.ssh/ocp_rsa | sed 's/^/  /')
export ssh_pub_key=$(cat ~/.ssh/ocp_rsa.pub)
export pull_secret=$(cat ~/git/openshift-enablement-exam/4.0/config/pullsecret.json)
export aws_id=$(cat ~/.aws/credentials | grep aws_access_key_id | cut -d'=' -f 2)
export aws_key=$(cat ~/.aws/credentials | grep aws_secret_access_key | cut -d'=' -f 2)
```

Deploy clusters

```shell
oc new-project crdb-clusters
envsubst < ./hive-values.yaml > /tmp/hive-values.yaml
helm upgrade clusters ./hive-clusters -i -n crdb-clusters  -f /tmp/hive-values.yaml
```

wait for the clusters to be deployed:

```shell
watch oc get clusterdeployment -n crdb-clusters
```

this will take around 40 minutes.

Set the kubeconfig context variables for the newly create cluster

```shell
export pwd1=$(oc get secret $(oc get clusterdeployment raffa1 -n crdb-clusters -o jsonpath='{.spec.clusterMetadata.adminPasswordSecretRef.name}') -n crdb-clusters -o jsonpath='{.data.password}' | base64 -d)
export pwd2=$(oc get secret $(oc get clusterdeployment raffa2 -n crdb-clusters -o jsonpath='{.spec.clusterMetadata.adminPasswordSecretRef.name}') -n crdb-clusters -o jsonpath='{.data.password}' | base64 -d)
export pwd3=$(oc get secret $(oc get clusterdeployment raffa3 -n crdb-clusters -o jsonpath='{.spec.clusterMetadata.adminPasswordSecretRef.name}') -n crdb-clusters -o jsonpath='{.data.password}' | base64 -d)
export apiurl1=$(oc get clusterdeployment raffa1 -n crdb-clusters -o jsonpath='{.status.apiURL}')
export apiurl2=$(oc get clusterdeployment raffa2 -n crdb-clusters -o jsonpath='{.status.apiURL}')
export apiurl3=$(oc get clusterdeployment raffa3 -n crdb-clusters -o jsonpath='{.status.apiURL}')

export context_cluster_control=$(oc config current-context)

oc login -u kubeadmin -p ${pwd1} --insecure-skip-tls-verify=true ${apiurl1}
export context_cluster1=$(oc config current-context)

oc login -u kubeadmin -p ${pwd2} --insecure-skip-tls-verify=true ${apiurl2}
export context_cluster2=$(oc config current-context)

oc login -u kubeadmin -p ${pwd3} --insecure-skip-tls-verify=true ${apiurl3}
export context_cluster3=$(oc config current-context)
```

## Deploy submariner

```shell
git clone https://github.com/submariner-io/submariner-charts
```

Deploy the broker

```shell
helm update submariner-broker ./submariner-charts/submariner-k8s-broker --kube-context ${context_cluster_control} -i --create-namespace -f ./values-sm-broker.yaml -n submariner-broker
```

Deploy submariner

```shell
export broker_api=$(oc --context ${context_cluster_control} get infrastructure cluster -o jsonpath='{.status.apiServerURL}')
export broker_ca=$(oc --context ${context_cluster_control} get secret $(oc --context ${context_cluster_control} get sa default -n default -o yaml | grep token | awk '{print $3}') -n default -o jsonpath='{.data.ca\.crt}' | base64 -d | sed 's/^/    /')
for context in ${context_cluster1} ${context_cluster2} ${context_cluster3}; do
  cluster_cidr=$(oc --context ${context} get network cluster -o jsonpath='{.status.clusterNetwork[0].cidr}')
  cluster_service_cidr=$(oc --context ${context} get network cluster -o jsonpath='{.status.serviceNetwork[0]}')
  cluster_service_cidr=$(oc --context ${context} get infrastructure cluster -o jsonpath='{.status.infrastructureName}')

  helm update submariner ./submariner-charts/submariner --kube-context ${context} -i --create-namespace -f ./values-sm.yaml -n submariner
done
```
