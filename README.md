# coopgo/peertube-k8s

Deploy [Peertube](https://joinpeertube.org/) on Kubernetes using [Kustomize](https://kustomize.io/).

Made by [COOPGO](https://coopgo.fr).

Inspired by [Peertube on Kubernetes](https://forge.extranet.logilab.fr/open-source/peertube-on-kubernetes/).

## Assumptions 

We made some assumptions for the deployment that make this version different from `Peertube on Kubernetes` :

- This repository is a Kustomize base that you can extend (with ingress, etc...) by implementing overlays (instead of providing .example files). See [Off The Shelf Application in Kustomize documentation](https://kubectl.docs.kubernetes.io/guides/config_management/offtheshelf/).
- We wanted to use [the new native object storage settings available starting version 3.4](https://docs.joinpeertube.org/admin-remote-storage?id=native-object-storage) instead of s3fs. We defined a classical persistent storage in the base configuration, and specific configuration such as object storage environment variables can be added in an the overlay. (see instructions below) You could even use overlays to replace persistent storage by s3fs, but we'll not address that.
- We use external PostgreSQL database and SMTP server (we don't install Postfix in the base deployment)
- At [COOPGO](https://coopgo.fr), we store our configuration in git repositories, and we don't want to store plain text credentials inside. Instead we use [Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets). Secrets configuration will be delegated to overlays (You can still use secretGenerator there if you want).

## Instructions

First off, initiate your Kustomize overlay by referencing our base (or, better, fork our base and reference your fork, but examples will be based on our Kustomize base)

In a repository, create `kustomization.yaml` :

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: peertube

bases:
- https://github.com/coopgo/peertube-k8s/
```

Here we are, we have a peertube K8S configuration ! ;) (not yet in fact, you have to "kustomize" it with your own parameters -database credentials, ingress hostnames, etc...-, it was too simple right ? ;))

### Environment variables

To change most of the parameters of Peertube deployment, you can store environment variables in a "secret" inside your Kubernetes namespace. This can be done like in [Peertube on Kubernetes](https://forge.extranet.logilab.fr/open-source/peertube-on-kubernetes/) using Kustomize secretGenerator. (copy their kustomization.yaml example file, append to the one we just created, and adapt environment variables to your desired values : if you do that, you can skip the next paragraphs and go directly to the Ingress section).

This is good for simple deployment purpose, where you do this once, or if you can store your files in a trusted location you're the only one to use. But if you want to, let's say, store your configuration in a Git repository accessible by others (for example, in your team at work, or with other teams in bigger organizations), that's bad (very very bad) in terms of security (everyone will know your database credentials, object storage access and secret keys, etc...). Instead, we use Bitnami Sealed Secrets and store the sealed secret file in the repository. This is not a lesson about Bitnami Sealed Secrets, so if you want to use it, we [let you go on their Git and documentation for installation instructions](https://github.com/bitnami-labs/sealed-secrets)).

Once installed (both the controller on your cluster and the `kubeseal` command locally), you can create a sealed secret. First, create a generic secret file with your environment variables, named "peertube-secret" (in a file called peertube-secret.yaml for this example. Do not store this in Git, you can even do that in another location like /tmp. Once the sealed secret created, we can forget it or keep it really safe in a trusted place). 

Example of a generic secret serving as a base for the sealed secret : 

(in this example, we configure object storage using PEERTUBE_OBJECT_STORAGE_XXXXXX environment variables : remove them or set PEERTUBE_OBJECT_STORAGE_ENABLED to false if you don't need it. Even with object storage enabled, it will still store your videos in a persistent volume, so it's different from using s3fs as done by `Peertube on Kubernetes` -see Peertube documentation for more details on how native object storage works-)

```yaml
apiVersion: v1
kind: Secret
metadata:
  # Note how the Secret is named
  name: peertube-secret
  namespace: peertube
type: bootstrap.kubernetes.io/generic
stringData:
  POSTGRES_USER: "peertube"
  POSTGRES_PASSWORD: "peertube"
  POSTGRES_DB: "peertube"
  PEERTUBE_DB_USERNAME: "peertube"
  PEERTUBE_DB_PASSWORD: "peertube"
  PEERTUBE_DB_SSL: "true"
  PEERTUBE_DB_HOSTNAME: "your-postgre-host"
  PEERTUBE_DB_PORT: "12345"
  PEERTUBE_WEBSERVER_HOSTNAME: "peertube.your-server.com"
  PEERTUBE_SMTP_USERNAME: "example@your-server.com"
  PEERTUBE_SMTP_PASSWORD: "smtppassword"
  PEERTUBE_SMTP_HOSTNAME: "mail.your-server.com"
  PEERTUBE_SMTP_PORT: "587"
  PEERTUBE_SMTP_FROM: "noreply@your-server.com"
  PEERTUBE_SMTP_TLS: "false"
  PEERTUBE_SMTP_DISABLE_STARTTLS: "false"
  PEERTUBE_ADMIN_EMAIL: "you@your-server.com"
  PEERTUBE_OBJECT_STORAGE_ENABLED: "true"
  PEERTUBE_OBJECT_STORAGE_ENDPOINT: "s3.fr-par.scw.cloud"
  PEERTUBE_OBJECT_STORAGE_REGION: "fr-par"
  PEERTUBE_OBJECT_STORAGE_STREAMING_PLAYLISTS_BUCKET_NAME: "yourbucket"
  PEERTUBE_OBJECT_STORAGE_STREAMING_PLAYLISTS_PREFIX: "streaming/"
  PEERTUBE_OBJECT_STORAGE_VIDEOS_BUCKET_NAME: "yourbucket"
  PEERTUBE_OBJECT_STORAGE_VIDEOS_PREFIX: "videos/"
  AWS_ACCESS_KEY_ID: "S3ACCESSKEY"
  AWS_SECRET_ACCESS_KEY: "S3SECRETKEY"
  PEERTUBE_INSTANCE_NAME: "Your server name"
  PEERTUBE_INSTANCE_DESCRIPTION: "Your server description"
```

And now, generate the sealed secret :

```
$ cat peertube-secret.yaml | kubeseal --controller-namespace kube-system --controller-name sealed-secrets-controller --format yaml > sealed-secret.yaml
```

Your sealed secret with encrypted environment variables will be in the file `sealed-secret.yaml`

Copy this file in your Kustomize directory

Add the resources reference of this file to the `kustomization.yaml` file :

```yaml
[...]

resources:
- sealed-secret.yaml
```

### Ingress

You also need to override Ingress configuration with your hostname. For this, you can patch the default ingress settings from the base configuration :

Add this to `kustomization.yaml` :

```yaml
[...]

patchesJson6902:
  - path: ingress-patch.yaml
    target:
      group: networking.k8s.io
      version: v1
      kind: Ingress
      name: peertube
```

And create `ingress-patch.yaml` in the same folder :

```yaml
- op: replace
  path: /spec/rules/0/host
  value: peertube.your-server.com
- op: replace
  path: /spec/tls/0/hosts/0
  value: peertube.your-server.com
```

### Other configurations

You should have a pretty good configuration right now. But you can go further. For example, if you want to adapt production configurations with settings not available as environment variables (for example, to disable the `tracker`) by [adapting the default configuration through the production.yaml file](https://github.com/Chocobozzz/PeerTube/blob/develop/config/production.yaml.example) and using a ConfigMap :

Copy the content of the file locally in `production.yaml` and adapt values.

Use `configMapGenerator` to generate the configmap from Kustomize. In `kustomization.yaml`, add :

```yaml
[...] 

configMapGenerator:
- name: peertube-production-yaml-configmap
  files:
  - production.yaml
```

Create a new patch to use this configmap to override /app/config/production.yaml in the deployed container. Create `deployment-patch.yaml` :

```yaml
- op: add
  path: /spec/template/spec/volumes/-
  value: 
    name: peertube-config
    configMap:
      name: peertube-production-yaml-configmap
      items:
        - key: production.yaml
          path: production.yaml
- op: add
  path: /spec/template/spec/containers/0/volumeMounts/-
  value:
    name: peertube-config
    mountPath: /app/config/production.yaml
    subPath: production.yaml
```

Add this patch to your `kustomization.yaml` "patchesJson6902" list, just after the ingress patch :

```yaml
patchesJson6902:
  - path: ingress-patch.yaml
    target:
      group: networking.k8s.io
      version: v1
      kind: Ingress
      name: peertube
  - path: deployment-patch.yaml
    target:
      group: apps
      version: v1
      kind: Deployment
      name: peertube
```

You can do many more customizations, for example by extending persistent volume claim storage, etc... Just look at [Kustomize documentation](https://kubectl.docs.kubernetes.io/guides/config_management/) for every possibilities.

### Apply your Kustomize configuration

You can now use this configuration to deploy to Kubernetes.

If it is not done yet, create namespace `peertube` (if you want another namespace, change "namespace: yournamespacename" in `kustomize.yaml` and the namespace name in the following command) :

```
$ kubectl create ns peertube
```

And finally, apply your configuration using the "-k" (Kustomize) option :

```
kubectl apply -k .
```


### Full kustomization.yaml example (if you followed our instructions exactly) :

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: peertube

bases:
- https://github.com/coopgo/peertube-k8s/

resources:
- sealed-secret.yaml

configMapGenerator:
- name: peertube-production-yaml-configmap
  files:
  - production.yaml

patchesJson6902:
  - path: ingress-patch.yaml
    target:
      group: networking.k8s.io
      version: v1
      kind: Ingress
      name: peertube
  - path: deployment-patch.yaml
    target:
      group: apps
      version: v1
      kind: Deployment
      name: peertube
```

## Licence

`coopgo/peertube-k8s` is released under GNU LGPL v2.1 licence (We simply followed the licence from [Peertube on Kubernetes](https://forge.extranet.logilab.fr/open-source/peertube-on-kubernetes/) as we took inspiration from it)