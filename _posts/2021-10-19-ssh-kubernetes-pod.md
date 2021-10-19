---
title: "Run an OpenSSH server as a bastion on a Kubernetes Pod"
layout: post
date: 2021-10-19 23:00
headerImage: false
tag:
  - kubernetes
  - open-source

category: blog
author: zainpatel
description: "Ever needed a convenient bastion/jump-host to something hidden away in your internal private subnets? Have a Kubernetes cluster up and running already? This blog post walks you through how to spin up an OpenSSH server in a pod for easy SSH port-tunneling"
---

I should probably caveat this up-front with the fact that this goes against the raison-d'être of Kubernetes slightly, probably isn't best practice and you most likely shouldn't do this in production. That said -- I have this running in production and am fairly pleased with it and don't feel _too_ ashamed of myself.

## Introduction

Let me outline my setup for some context. I run a Kubernetes cluster using [EKS](https://aws.amazon.com/eks/) which runs the backend, frontend and some other small services for an application I've built. This cluster sits on a public subnet of my [VPC](https://aws.amazon.com/vpc/) and my database which hosts all of the application data sits on a private subnet in [RDS](https://aws.amazon.com/rds/), comfortably nestled away from the world.

I often need access to this database, either for analytics or debugging purposes, on my local machine. This would typically call for an EC2 instance running on the same public subnet, with the appropriate security group configurations to allow it to talk to my database and let me in. Given that all of my security group configurations are done in Terraform, I don't want to encode this ephermeral bastion into my Terraform (and I also maintain Terraform for Azure, which means more work - and I'm lazy). It just felt like the wrong level to do things at, when I had an entire EKS cluster to operate within.

I spent a good half-week searching (I'm a Kubernetes newbie) for how to implement the solution I had in my head, which was to run an SSH server on a Kubernetes pod (due to being within the VPC, had access to the RDS endpoint) and then use SSH local forwarding to connect to the server in that pod, and forward the RDS port down to my local machine. In an ideal world, I'd have been able to spin up an empty pod and run `kubectl port-forward -L 5432:endpoint:5432` or something. That's a bit of a lie -- the reason that that isn't ideal is that it requires `kubectl` installed on my machine (fine for me, less so for my product manager) and also means that various tools that allow a database connection to specify an `ssh` connection no longer work (e.g PyCharm's database viewer).

## Implementation

First off, I needed to spin a pod up that runs an [OpenSSH](https://www.openssh.com) server, preferrably with the `authorized_keys` baked into the image already. To do this I used this super simple Dockerfile from [corbinu/ssh-server](https://github.com/corbinu/ssh-server).

1. Clone the repository down:

```
git clone https://github.com/corbinu/ssh-server.git
```

2. Add the relevant public keys to the `authorized_keys` file at the root of the repository (create it if required)

```
echo "public_key_1" > authorized_keys
echo "public_key_2" >> authorized_keys
```

3. Modify the configuration within `sshd_config` to ensure that the SSH connection doesn't time out after 1 minute (as it does by default):

```
ClientAliveInterval 30
ClientAliveCountMax 5
```

for a timeout of `30 * 5` seconds. Modify as appropriate.

4. Build the docker image, tag it with your relevant repository tag (I use Amazon [ECR](https://aws.amazon.com/ecr/)) and push it to an appropriate registry

```
docker build -t account_id.amazon.ecr.io/repo/app-ssh-server:latest .
docker push account_id.amazon.ecr.io/repo/app-ssh-server:latest
```

Alternatively, you could leave it as-is and simply use the `corbinu/ssh-server` image from DockerHub, but will need to configure the `authorized_keys` by `kubectl exec -it <pod-name> -- /bin/bash` and editing the file directly, and will mean that you aren't able to configure `sshd_config`

5. Spin up a Kubernetes pod with your pre-baked image:

```
kubectl run app-ssh-server --image account_id.amazon.ecr.io/repo/app-ssh-server:latest --port 22 --restart Never --namespace app-maintenance
```

6. Expose this Kubernetes pod to the outside world using a Load Balancer (your cluster must be configured to talk to some Cloud Provider to create the necessary resources):

```
kubectl expose pod app-ssh-server --namespace app-maintenance --port 22
```

7. If you monitor your services with `kubectl get svc -n app-maintenance`, you should see a service dedicated to your pod with an external IP provisoned.

8. For best security practices, edit the configuration of this created service to whitelist certain IP addresses

```
kubectl edit svc/app-ssh-server --namespace app-maintenance
```

using (for example) `sourceLoadBalancerRage` under the `spec` key.

9. Connect to your SSH server using `ssh <external-ip>` and optionally forward some ports using `-L` or the `LocalForward` configuration
