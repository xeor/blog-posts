---
title: Kubernetes pod - read-once config
date: 2021-01-29
summary: Make sure the pod can only read a configuration only one time
tags: ['kubernetes', 'security']
githubFiles: true
---

## Table of Contents

## Introduction

Sometimes you need to store sensitive information in a configuration-file in kubernetes, but you don't want the information laying around.

This post is about an idea for how to fix this problem using some tricks:

* Using an encrypted config stored in a persistant volume
* Making the password for the encrypted config available when starting the container, but then remove it after decryption
* Remove the decrypted config file once it is read
* Leaving a running app started with the decrypted config, but without any way of getting to that config if you have access inside the container.

Everything needed is in this post github-files <g-link :to=$page.post.githubFiles>here</g-link>.

Also note that we use `alpine` with `s6-overlay`. This trick should also work with other distroes without too much changes, but some changes are needed.

:::tip Heads-up
We use the fact that you usually don't have access using `SYS_PTRACE` inside a container, hence, there is no way to debug the running application to get to the unencrypted config.
:::

### Prerequisites

* Knowledge about containers, alpine and some s6-overlay
* An application that doesnt care if it's config is deleted after it is started.

## Actual content

### The container

There is little you would need to do to the Containerfile. What's important is to include gnupg, s6-overlay and some scripts.
Below is a full example of a containerfile that should work out of the box.

:::tip
Dockerfile and Containerfile are mostly the same.. This works using docker as well, just use `docker build -f Containerfile`.
:::

```dockerfile{codeTitle: "In Containerfile"}{file=true}
```

The example-image used in the `k8s-resources.yaml` below was built and uploaded to docker-hub like this:

```bash{outputLines: 2-6,8-10}
docker build -f Containerfile -t xeor/k8s-read-once-example:latest .
Sending build context to Docker daemon  10.75kB
Step 1/6 : FROM alpine
# Lots of lines #
Successfully built 53daa1df30ae
Successfully tagged xeor/k8s-read-once-example:latest
docker push xeor/k8s-read-once-example:latest
The push refers to repository [docker.io/xeor/k8s-read-once-example]
# More lines #
1: digest: sha256:fd68012fa17907af7d22fbb693f982c0dfddeadd5ea7ddb8d0de1e550f6b4e7f size: 947
```

Inside the `root` directory, we need to put some helper scripts..

#### /usr/bin/secret_encrypt

This is a simple helper-script to encrypt a file `/config/secrets.yaml` into the file `/config/secrets.yaml.enc`. This will be helpfull to test and bootstrap your setup, and in case you want to change the data in the encrypted file.

```shell{codeTitle: "In root/usr/bin/secret_encrypt"}{file=true}
```

#### /usr/bin/secret_decrypt

Used for decryption when the container starts.
It should be called at container start to make sure the decrypted version of the config is ready when the appliation reads it.

```shell{codeTitle: "In root/usr/bin/secret_decrypt"}{file=true}
```

#### /usr/bin/secret_delete_waiter

This file might need to be modified to your need. This version is tailored to work with the inotify utility that comes with busybox.
See `/etc/cont-init.d/00-secrets-handler` below for how it is used. It's single goal is work with inotifyd to delete a file once it is read.

```shell{codeTitle: "In root/usr/bin/secret_delete_waiter"}{file=true}
```

#### /etc/cont-init.d/00-secrets-handler

Placing this script inside `cont-init.d` when using `s6-overlay` makes it start before every service.
It's goal is simple,

1. Decrypt the secret
2. Put a watcher in the background that will delete the config-file once the application is started.

```shell{codeTitle: "In root/etc/cont-init.d/00-secrets-handler"}{file=true}
```

### Kubernetes

Now that we have an idea how the image should be modified, lets start with kubernetes.

We will use [minikube](https://minikube.sigs.k8s.io/docs/start/), but should work fine on a normal kubernetes cluster as well.
Just keep in mind that the default `/config` persistant volume we create uses a volume on the node itself.

#### Create the resources

This file is used to create the kubernetes resources for the example. Nothing magical, but see comments inline in the file for more info.

```yaml{codeTitle: "In k8s-resources.yaml"}{file=true}
```

To get everything running, creating the pod and the secret defined.

```bash{outputLines: 2-3}
kubectl apply -f k8s-resources.yaml
pod/pod-with-secret created
secret/encryption-key created
```

Check the logs of the container after it has started up. You might want to wait a second or two, maybe even check the status while you wait with `kubectl get pods --watch pod-with-secret` (^c to exit the status-view)

```bash{outputLines: 2-13}
kubectl logs pod-with-secret 
[s6-init] making user provided files available at /var/run/s6/etc...exited 0.
[s6-init] ensuring user provided files have correct perms...exited 0.
[fix-attrs.d] applying ownership & permissions fixes...
[fix-attrs.d] done.
[cont-init.d] executing container initialization scripts...
[cont-init.d] 00-secrets-handler: executing... 
/config/secrets.yaml.enc dont exists. Halting so you can inspect manually.

If this is the first time you start this container, and you want to initialize the encrypted config:
  * Make sure the unencrypted config (/config/secrets.yaml) exists.
  * Use the 'secret_encrypt' command to make /config/secrets.yaml.enc
  * Kill/restart the container
```

#### Creating the initial secrets.yaml

When the pod is created, we will have an empty `/config` folder. This is where our config lives, both the encrypted and the temporary unencrypted version.

Enter the container to create the secrets.yaml and encrypt it.. Use `kubectl exec -it pod-with-secret -- sh` to enter the container before executing the following inside the container.

```bash{outputLines: 3-5,7,10-15}{promptUser: "root"}{promptHost: "pod-with-secret"}{codeTitle: "Inside the container"}
echo "secret: 1234" > /config/secrets.yaml
secret_encrypt  # This works since /etc/init-secrets/secretfile_pw exists
gpg: directory '/root/.gnupg' created
gpg: keybox '/root/.gnupg/pubring.kbx' created
All done, deleted /etc/init-secrets/secretfile_pw
ls /config
secrets.yaml      secrets.yaml.enc
rm /config/secrets.yaml
cat /config/secrets.yaml.enc
-----BEGIN PGP MESSAGE-----

jA0ECQMK2zgjW8e3yeH/0koB6uIXd8Z58VaVP2sAPq9YfNrczEyMskP0sfZQRPW6
oUi5JUFFP3GQqGJvccTlRJK6LsRlID0E59fATRXIx77HiD8Ykzhp633q4w==
=bOCo
-----END PGP MESSAGE-----
exit
```

#### Trying it out

We can now recreate the resources to see how it behaves. Note that the `/config` volume is persistant.

:::tip
Normally, you could just kill the pod and it will be restarted by kubernetes, but we are running a `Kind` of `Pod`, which is just a dumb pod. When they are restarted automatically, they are running under something else, like an `ReplicaSet` or `Deployment`.
:::

```bash{outputLines: 2-3,5-6}
kubectl delete -f k8s-resources.yaml
pod "pod-with-secret" deleted
secret "encryption-key" deleted
kubectl apply -f k8s-resources.yaml 
pod/pod-with-secret created
secret/encryption-key created
```

Check the logs to see that it now decrypted our file and deleted the temp-pw-file.
```bash{outputLines: 2-16}
kubectl logs pod-with-secret
[s6-init] making user provided files available at /var/run/s6/etc...exited 0.
[s6-init] ensuring user provided files have correct perms...exited 0.
[fix-attrs.d] applying ownership & permissions fixes...
[fix-attrs.d] done.
[cont-init.d] executing container initialization scripts...
[cont-init.d] 00-secrets-handler: executing... 
gpg: directory '/root/.gnupg' created
gpg: keybox '/root/.gnupg/pubring.kbx' created
gpg: AES256.CFB encrypted data
gpg: encrypted with 1 passphrase
All done, deleted /etc/init-secrets/secretfile_pw
[cont-init.d] 00-secrets-handler: exited 0.
[cont-init.d] done.
[services.d] starting services
[services.d] done.
```

Enter the new container (`kubectl exec -it pod-with-secret -- sh`) when it has started.
The secrets.yaml file can now only be read once before it is deleted.
```bash{outputLines: 2,4,6}
ls /config/
secrets.yaml      secrets.yaml.enc
cat /config/secrets.yaml
secret: 1234
cat /config/secrets.yaml
cat: can't open '/config/secrets.yaml': No such file or directory
```

Notice that we where only able to read the file once. This is the same as if the application reads it.

The magic that deletes it is the `inotify` process running in the background started by `00-secrets-handler`.

:::important
You should make sure that if you app dies, the pod also dies (is killed). Doing the above will remove the ability for the app to restart itself.
:::
