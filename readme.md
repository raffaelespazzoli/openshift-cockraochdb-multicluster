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

## Emulating 3 clusters

```shell
oc new-project cockroachdb-1
oc new-project cockroachdb-2
oc new-project cockroachdb-3
helm template cockroachdb ./cockroachdb-emulation -n cockroachdb-1 -f three-clusters-emulation-values.yaml | oc apply -f - -n cockroachdb-1
helm template cockroachdb ./cockroachdb-emulation -n cockroachdb-2 -f three-clusters-emulation-values.yaml | oc apply -f - -n cockroachdb-2
helm template cockroachdb ./cockroachdb-emulation -n cockroachdb-3 -f three-clusters-emulation-values.yaml | oc apply -f - -n cockroachdb-3
oc exec cockroachdb-0 -n cockroachdb-1 -- /cockroach/cockroach init --insecure
```

## Verify the cluster

```shell
oc exec cockroachdb-0 -n cockroachdb-1 -- /cockroach/cockroach nodes status --insecure
```

## CockroachDB on Multiple OCP clusters

Assuming you have three cluster under context $CLUSTER1, $CLUSTER2, CLUSTER3, run the following

```shell
oc --context $CLUSTER1 new-project cockroach
```
