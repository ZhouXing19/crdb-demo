## Download the cli with the correct release tag

https://github.com/cockroachlabs/crdb-proxy/actions?query=event%3Arelease, find the correct release tag. Click into it, find `release-cli summary`, and download the cli binary.

## Get the helm charts with the correct release tag

```sh
helm pull lms/lms --version=0.2.3
```

will download the tgz locally. Unzip the tgz file, go into the directory (I renamed it to `helm-molt-lms-0-2-3`) and `helm dependency update`. 


## Start DBs + cdc-sink + LMS + obs
```bash
helm install \
  --namespace default \
  -f helm-molt-lms-0-2-3/values.yaml lms helm-molt-lms-0-2-3
```

To check the status of nodes with `kgp -w`.

## Port-forward

```bash
kubectl port-forward svc/lms 9043:9043 & \
kubectl port-forward svc/lms-orchestrator 4200:4200 & \
kubectl port-forward svc/lms-grafana 3000:80 & \
kubectl port-forward svc/lms-cockroachdb 26257:26257 & \
kubectl port-forward svc/lms-mysql 3306:3306 & \
echo "Run pkill -9 kubectl to stop port-forwarding..."
```

## Run `SELECT VERSION()` on both DB


## Create schema on both mysql and crdb

### MySQL

```bash
mysql -u root -p'password'  -h '0.0.0.0' -P 3306 --database=defaultdb
```

```sql
CREATE TABLE purchase (
  id VARCHAR(36) DEFAULT (uuid()) PRIMARY KEY,
  basket_id VARCHAR(36) NOT NULL,
  member_id VARCHAR(36) NOT NULL,
  amount DECIMAL NOT NULL,
  timestamp TIMESTAMP NOT NULL DEFAULT now()
);

SET @@GLOBAL.ENFORCE_GTID_CONSISTENCY = ON;
SET @@GLOBAL.GTID_MODE = ON;
SET @@GLOBAL.BINLOG_ROW_METADATA = FULL;
```

### CockroachDB

```bash
CREATE TABLE purchase (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  basket_id UUID NOT NULL,
  member_id UUID NOT NULL,
  amount DECIMAL NOT NULL,
  timestamp TIMESTAMP NOT NULL DEFAULT now()
);
```


### Start the APP

``` sh
make deploy_app

kubetail app
```

### Check the count of rows

```sql
SELECT count(*) FROM purchase
```

### Check the log of cdc-sink

```shell
kl $(kubectl get pods --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}' | grep cdcsink)
```


### Cutover begin

```sh
export CLI_ORCHESTRATOR_URL="http://0.0.0.0:4200"

./molt-lms-cli cutover consistent begin

kl $(kubectl get pods --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}' | grep app) --follow

SELECT COUNT(*) FROM purchase

./molt-lms-cli cutover consistent commit
```

### Cleanup

```sh

(toproxy && ./cleanup-k8.sh)

```