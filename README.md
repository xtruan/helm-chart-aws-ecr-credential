# aws-ecr-credential

AWS ECR requires a password refreshed every 12 hours in order to pull images. This chart provides a mechanism to integrate Kubernetes with AWS ECR by automating this process.

Simply deploy this chart to your Kubernetes cluster and you will be able to pull and run images from your AWS ECR (Elastic Container Registry) in your cluster.

# Quickstart

Run the following command to register a shared AWS ECR secret alongside a service account:

```sh
$ export AWS_ECR_REGISTRY=<account>.dkr.ecr.<region>.amazonaws.com
$ export AWS_ACCESS_KEY_ID=<>
$ export AWS_SECRET_ACCESS_KEY=<>
$ kubectl create namespace ecr-credential
$ helm install register-aws-ecr-credential . \
  --set "mode=register" \
  --set-string "aws.ecrRegistry=$AWS_ECR_REGISTRY" \
  --set "aws.accessKeyId=$AWS_ACCESS_KEY_ID" \
  --set "aws.secretAccessKey=$AWS_SECRET_ACCESS_KEY" \
  --set "awsSecret=ecr-credential-secret" \
  --set "awsSecretNamespace=ecr-credential" \
  --set "refreshAccount=ecr-credential-refresh"
```

After running, there should be a secret and a service account:
```sh
kubectl -n my-admin-namespace get secrets,serviceaccounts
NAME                                        TYPE                                  DATA   AGE
secret/default-token-7t22j                  kubernetes.io/service-account-token   3      53s
secret/ecr-credential-secret                Opaque                                3      24s
secret/ecr-credential-refresh-token-fq2b6   kubernetes.io/service-account-token   3      24s

NAME                                    SECRETS   AGE
serviceaccount/default                  1         53s
serviceaccount/ecr-credential-refresh   1         24s
```

Next, create a job+cron to populate the Docker registry secret in another namespace:

```sh
$ kubectl create namespace my-user-namespace
$ helm install refresh-image-pull-secret-for-my-user-namespace . \
  --set "mode=refresh" \
  --set "awsSecret=ecr-credential-secret" \
  --set "awsSecretNamespace=ecr-credential" \
  --set "refreshAccount=ecr-credential-refresh" \
  --set "targetSecret=ecr-credential-secret" \
  --set "targetNamespace=my-user-namespace"
```

After running, there should be a job+cronjob and a secret in the user namespace:
```sh
$ kubectl -n my-admin-namespace get jobs,cronjobs
NAME                                                  COMPLETIONS   DURATION   AGE
job.batch/refresh-image-pull-secret-for-my-user-job   1/1           5s         39s

NAME                                                       SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/refresh-image-pull-secret-for-my-user-cron   * */8 * * *   False     0        <none>          39s
$ kubectl -n my-user-namespace get secrets
NAME                    TYPE                                  DATA   AGE
default-token-cmp42     kubernetes.io/service-account-token   3      11m
ecr-credential-secret   kubernetes.io/dockerconfigjson        1      2m34s
```

This image pull secret can then be used in the standard way:

Example:
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      imagePullSecrets:
      - name: ecr-credential-secret
      containers:
        - name: node
          image: node:latest
```
