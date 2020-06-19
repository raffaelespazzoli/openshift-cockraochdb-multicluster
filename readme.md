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

## Verify the cluster

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

## Start and Verify the cluster

```shell
export tools_pod=$(oc get pods -n cockroachdb-1 | grep tools | awk '{print $1}')
oc exec $tools_pod -c tools -n cockroachdb-1 -- /cockroach/cockroach init --certs-dir=/crdb-certs --host cockroachdb-0.cockroachdb
oc exec $tools_pod -c tools -n cockroachdb-1 -- /cockroach/cockroach node status --certs-dir=/crdb-certs --host cockroachdb-0.cockroachdb
```
