
## Introduction
In this example we will be deploying a kafka cluster with ERP and MDS enabled. We will be using OAuth for authentication and authorization. We will be using keycloak as IDP. We will be using RBAC for authorization.

## Pre-requisite

1. Set Tutorial Home
Navigate to the folder.

```bash
export TUTORIAL_HOME=$(pwd)
```

2. Create namespace 

```bash
kubectl create namespace operator
```

3. Other tools

This scenario workflow requires the following CLI tools to be available on the machine you are using:

- openssl
- cfssl

## Install keycloak



### Deploy keycloak
```bash
kubectl apply -f $TUTORIAL_HOME/keycloak/keycloak_deploy.yaml
```

### Validate keycloak is successfully deployed

```bash
 while true; do kubectl port-forward service/keycloak 8080:8080 -n operator; done;
 // Add entry in /etc/hosts
 127.0.0.1    keycloak
 // in second terminal
 curl --location 'http://keycloak:8080/realms/sso_test/protocol/openid-connect/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--header 'Authorization: Basic c3NvbG9naW46S2JMUmloMUh6akRDMjY3UGVmdUtVN1FJb1o4aGdIREs=' \
--data-urlencode 'grant_type=client_credentials'
```

## Deploy Confluent for Kubernetes

Set up the Helm Chart:

```
helm repo add confluentinc https://packages.confluent.io/helm
```

Install Confluent For Kubernetes using Helm:

```
helm upgrade --install operator confluentinc/confluent-for-kubernetes --namespace operator
```
  
Check that the Confluent For Kubernetes pod comes up and is running:

```
kubectl get pods --namespace operator
```
## Create Secret Objects

### Configure auto-generated certificates

Confluent For Kubernetes provides auto-generated certificates for Confluent Platform
components to use for TLS network encryption. You'll need to generate and provide a
Certificate Authority (CA).

Generate a CA pair to use:

```
openssl genrsa -out $TUTORIAL_HOME/ca-key.pem 2048

openssl req -new -key $TUTORIAL_HOME/ca-key.pem -x509 \
  -days 1000 \
  -out $TUTORIAL_HOME/ca.pem \
  -subj "/C=US/ST=CA/L=MountainView/O=Confluent/OU=Operator/CN=TestCA"
```

Create a Kubernetes secret for the certificate authority:

```
kubectl create secret tls ca-pair-sslcerts \
  --cert=$TUTORIAL_HOME/ca.pem \
  --key=$TUTORIAL_HOME/ca-key.pem -n operator
```

### Secret Object Credentials
```bash
     kubectl create secret generic credential \
       --from-file=plain-users.json=$TUTORIAL_HOME/creds/creds-kafka-sasl-users.json \
       --from-file=plain.txt=$TUTORIAL_HOME/creds/creds-client-kafka-sasl-user.txt \
       --namespace operator
```
### Secret Object MDS
```bash
kubectl create secret generic mds-token \
  --from-file=mdsPublicKey.pem=$TUTORIAL_HOME/creds/mds-publickey.txt \
  --from-file=mdsTokenKeyPair.pem=$TUTORIAL_HOME/creds/mds-tokenkeypair.txt \
  --namespace operator
```

## Deploy CP components

1. Create jass config secret

```bash
    kubectl create -n operator secret generic oauth-jass --from-file=oauth.txt=oauth_jass.txt
```
2. apply cp_components.yaml
    
```bash
kubectl apply -f cp_components.yaml
```

## Testing

1. Copy updated kafka.properties in /tmp

```bash
kubectl cp -n operator kafka.properties  kafka-0:/tmp/kafka.properties
```

2. Do Shell
   ```bash
   kubectl -n operator exec kafka-0 -it /bin/bash
   ```
3. Run topic command against token listener 9073 as with rbac we spawn a new listener
   ```bash
   kafka-topics --bootstrap-server kafka.operator.svc.cluster.local:9073 --topic test-topic --create --replication-factor 3 --command-config /tmp/kafka.properties
    kafka-topics --bootstrap-server kafka.operator.svc.cluster.local:9073 --topic test-topic --describe --command-config /tmp/kafka.properties
   ```
4. Validate operator should not throw errors for kafka-rest-class controller, as these logs indicate operator client is able to successfully connect with ERP
5. Get an access token
   ```bash
curl --request POST --url 'http://keycloak:8080/realms/sso_test/protocol/openid-connect/token' \
    --header 'Content-Type: application/x-www-form-urlencoded' --data-urlencode 'grant_type=client_credentials' \
    --data-urlencode 'client_id=ssologin'      --data-urlencode 'client_secret=KbLRih1HzjDC267PefuKU7QIoZ8hgHDK'
   ```
6. Test Curl requests against rest proxy at 8082, after port forwarding
```bash
    curl -k --location 'https://localhost:8082/v3/clusters' \
    --header 'Accept: application/json' \
    --header 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICIwTE1NczNseUtIeWRMRDlZRUsycjhVaGttdE13NE9IcXpmczZZRU9xNE9zIn0.eyJleHAiOjE3MjY0OTk3MzAsImlhdCI6MTcyNjQ5OTQzMCwianRpIjoiY2UzOTdiOWQtNzcwNS00YzFjLTkwYjYtMzJkNzc1YmIyZWJhIiwiaXNzIjoiaHR0cDovL2tleWNsb2FrOjgwODAvcmVhbG1zL3Nzb190ZXN0Iiwic3ViIjoiNDNhZDg4MjktMTA1Ni00ZDYwLTlmOGYtYmNkOTk0ZDI0YWI4IiwidHlwIjoiQmVhcmVyIiwiYXpwIjoic3NvbG9naW4iLCJhY3IiOiIxIiwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbImRlZmF1bHQtcm9sZXMtc3NvX3Rlc3QtMSJdfSwic2NvcGUiOiJwcm9maWxlIGVtYWlsIiwiZW1haWxfdmVyaWZpZWQiOmZhbHNlLCJjbGllbnRIb3N0IjoiMTAuMjQ0LjAuMTQiLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiJzZXJ2aWNlLWFjY291bnQtc3NvbG9naW4iLCJjbGllbnRBZGRyZXNzIjoiMTAuMjQ0LjAuMTQiLCJjbGllbnRfaWQiOiJzc29sb2dpbiJ9.bB7VRBFOX4UYsa8W32PtQ4hJvQBhf5WD7q60qkQDSAvQZQ0Fj3p7cUkRIsFbmE8GUhZ1AlS2iX0iIWflepmJ3fkO0MzG29VXCoHZ7raalj1uuwIRWiBjUjLRlBOnneFpCAoTAqYiZ4EDWOGvKL_eUD_4GVpoS8mRaEhk7pJD0KRoxePCYzHP4YAyj-ZwRH3IiBpwWsXdIwhMo4EKeNKngxbRG2M_hXD5utnQYWgSpYkqXTwsWgrPvpQc9sWoTv8VVGgDZEawT2wNZCDZFLYXFhtZEdobIHXFljixvIlztN2kjDBf8Ty9-oNuYVh0aEGEG6AcpYNtzCBMItMjH3K4ig'    
    ```
   Note: Token passes here are issued by IDP like keycloak, and are valid for 5 minutes.