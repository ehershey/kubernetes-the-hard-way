# Smoke Test

In this lab you will complete a series of tasks to ensure your Kubernetes cluster is functioning correctly.

## Data Encryption

In this section you will verify the ability to [encrypt secret data at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted).

Create a generic secret:

```
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Print a hexdump of the `kubernetes-the-hard-way` secret stored in etcd:

```
external_ip=$(aws ec2 describe-instances --filters \
  "Name=tag:Name,Values=controller-0" \
  "Name=instance-state-name,Values=running" \
  --output text --query 'Reservations[].Instances[].PublicIpAddress')

ssh -i kubernetes.rsa ubuntu@${external_ip} \
 "sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
```

> output

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a ef a1 50 52 26 2b e2  |:v1:key1:..PR&+.|
00000050  09 8b 3b c5 24 1c 0e 37  bc 41 dd 36 f6 f8 9d ba  |..;.$..7.A.6....|
00000060  77 17 c5 da b8 bc 9f e9  2b 09 86 94 ed 2f f9 04  |w.......+..../..|
00000070  f6 f8 f0 f7 d8 dc 65 86  cb c0 a7 04 07 03 8f 03  |......e.........|
00000080  52 2f 75 5e e6 ec 4f 37  b7 81 85 0b f4 57 6a 72  |R/u^..O7.....Wjr|
00000090  b1 8c 2f 13 94 9b 28 32  f4 f8 47 db f6 25 39 f1  |../...(2..G..%9.|
000000a0  de 42 7e 5e 80 d4 80 8c  23 1f d0 2e ee 23 15 40  |.B~^....#....#.@|
000000b0  6e c8 c7 5f 85 27 51 cd  89 a0 e3 f8 db bf 23 18  |n.._.'Q.......#.|
000000c0  05 d0 40 c8 97 a7 4e 63  4c 5f f0 41 3d 1c cd 74  |..@...NcL_.A=..t|
000000d0  4f 1b fb b7 6d bf 9a 24  90 a3 5c 25 e9 4d ec 63  |O...m..$..\%.M.c|
000000e0  a4 44 9f 10 c8 dd 8c 8f  e0 3a 65 68 d1 b3 5c fc  |.D.......:eh..\.|
000000f0  dc d6 fb fe ce 64 f7 6e  a9 ee 26 11 24 a9 81 72  |.....d.n..&.$..r|
00000100  09 cc cb 45 30 07 49 9d  16 ab 98 54 59 b5 b4 30  |...E0.I....TY..0|
00000110  56 8f 8a 40 da 24 2d 57  b2 4f 86 64 07 26 df d7  |V..@.$-W.O.d.&..|
00000120  01 70 e9 0a f8 d3 f8 72  72 de 0f 86 58 8e 29 85  |.p.....rr...X.).|
00000130  fd 49 e3 83 4f d7 cd ee  db d7 86 13 cf 67 9b 65  |.I..O........g.e|
00000140  70 1b f2 54 d6 72 b4 04  8a 9c 5e 04 c6 11 67 66  |p..T.r....^...gf|
00000150  c4 6c f9 c1 dd 05 d9 5f  16 0a                    |.l....._..|
0000015a
```

The etcd key should be prefixed with `k8s:enc:aescbc:v1:key1`, which indicates the `aescbc` provider was used to encrypt the data with the `key1` encryption key.

## Deployments

In this section you will verify the ability to create and manage [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Create a deployment for the [nginx](https://nginx.org/en/) web server:

```
kubectl create deployment nginx --image=nginx
```

List the pod created by the `nginx` deployment:

```
kubectl get pods -l app=nginx
```

> output

```
NAME                     READY   STATUS    RESTARTS   AGE
nginx-6799fc88d8-tq99b   1/1     Running   0          10s
```

### Port Forwarding

In this section you will verify the ability to access applications remotely using [port forwarding](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/).

Retrieve the full name of the `nginx` pod:

```
POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath="{.items[0].metadata.name}")
```

Forward port `8080` on your local machine to port `80` of the `nginx` pod:

```
kubectl port-forward $POD_NAME 8080:80
```

> output

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

In a new terminal make an HTTP request using the forwarding address:

```
curl --head http://127.0.0.1:8080
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.23.1
Date: Thu, 01 Sep 2022 22:56:06 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 19 Jul 2022 14:05:27 GMT
Connection: keep-alive
ETag: "62d6ba27-267"
Accept-Ranges: bytes
```

Switch back to the previous terminal and stop the port forwarding to the `nginx` pod:

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### Logs

In this section you will verify the ability to [retrieve container logs](https://kubernetes.io/docs/concepts/cluster-administration/logging/).

Print the `nginx` pod logs:

```
kubectl logs $POD_NAME
```

> output

```
...
127.0.0.1 - - [01/Sep/2022:22:56:06 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.79.1" "-"
```

### Exec

In this section you will verify the ability to [execute commands in a container](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container).

Print the nginx version by executing the `nginx -v` command in the `nginx` container:

```
kubectl exec -ti $POD_NAME -- nginx -v
```

> output

```
nginx version: nginx/1.23.1
```

## Services

In this section you will verify the ability to expose applications using a [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Expose the `nginx` deployment using a [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) service:

```
kubectl expose deployment nginx --port 80 --type NodePort
```

> The LoadBalancer service type can not be used because your cluster is not configured with [cloud provider integration](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider). Setting up cloud provider integration is out of scope for this tutorial.

Retrieve the node port assigned to the `nginx` service:

```
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

Create a firewall rule that allows remote access to the `nginx` node port:

```
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITYGROUP} \
  --protocol tcp \
  --port ${NODE_PORT} \
  --cidr 0.0.0.0/0
```

Retrieve the external IP address of a worker instance:

```
INSTANCE_NAME=$(kubectl get pod $POD_NAME --output=jsonpath='{.spec.nodeName}')


```

Make an HTTP request using the external IP address and the `nginx` node port:

```
curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

> output

```
HTTP/1.1 200 OK
Server: nginx/1.23.1
Date: Thu, 01 Sep 2022 22:59:45 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 19 Jul 2022 14:05:27 GMT
Connection: keep-alive
ETag: "62d6ba27-267"
Accept-Ranges: bytes
```

Next: [Cleaning Up](14-cleanup.md)
