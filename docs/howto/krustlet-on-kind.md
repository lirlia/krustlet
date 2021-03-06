# Running Krustlet on Kubernetes in Docker (KinD)

This how-to guide demonstrates how to boot a Krustlet node in a KinD cluster.

## Prerequisites

You will require a running KinD cluster for this how-to. `kubectl` is also required. See the [how-to
guide for running Kubernetes on KinD](kubernetes-on-kind.md) for more information.

This specific tutorial will be running Krustlet on your host Operating System; however, you can
follow these steps from any device that can start a web server on an IP accessible from the
Kubernetes control plane, including KinD itself.

## Step 1: Create Certificate

Krustlet requires a certificate for securing communication with the Kubernetes API. Because
Kubernetes has its own certificates, we'll need to get a signed certificate from the Kubernetes API
that we can use. First things first, let's create a certificate signing request (CSR):

```shell
$ mkdir -p ~/.krustlet/config
$ cd $_
$ openssl req -new -sha256 -newkey rsa:2048 -keyout krustlet.key -out krustlet.csr -nodes -subj "/C=US/ST=./L=./O=./OU=./CN=krustlet"
Generating a RSA private key
.................+++++
....................................................+++++
writing new private key to 'krustlet.key'
```

This will create a CSR and a new key for the certificate, using `krustlet` as the hostname of the
server.

Now that it is created, we'll need to send the request to Kubernetes:

```shell
$ cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: krustlet
spec:
  request: $(cat krustlet.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
certificatesigningrequest.certificates.k8s.io/krustlet created
```

Once that runs, an admin (that is probably you! at least it should be if you are trying to add a
node to the cluster) needs to approve the request:

```shell
$ kubectl certificate approve krustlet
certificatesigningrequest.certificates.k8s.io/krustlet approved
```

After approval, you can download the cert like so:

```shell
$ kubectl get csr krustlet -o jsonpath='{.status.certificate}' | base64 --decode > krustlet.crt
```

Lastly, combine the key and the cert into a PFX bundle, choosing your own password instead of
"password":

```shell
$ openssl pkcs12 -export -out certificate.pfx -inkey krustlet.key -in krustlet.crt -password "pass:password"
```

## Step 2: Determine the default gateway

The default gateway for most Docker containers (including your KinD host) is generally `172.17.0.1`.
We can use this IP address from the guest Operating System (the KinD host) to connect to the host
Operating System (where Krustlet is running). If this was changed, check `ip addr show docker0` from
the host OS to determine the default gateway.

### Special note: Docker Desktop for Mac

For Docker Desktop for Mac users, [the `docker0` bridge network is unreachable from the host
network](https://docs.docker.com/docker-for-mac/networking/#use-cases-and-workarounds) (and vice
versa). However, the `en0` host network is accessible from within the container.

Because the `en0` network is the default network, Krustlet will bind to this IP address
automatically. You should not need to pass a `--node-ip` flag to Krustlet.

In the event this does not appear to be the case (for example, when the hostname cannot resolve to
this address), check which IP address you have for the `en0` network:

```console
$ ifconfig en0
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
        options=400<CHANNEL_IO>
        ether 78:4f:43:8d:4f:55 
        inet6 fe80::1c20:1e66:6322:6ae9%en0 prefixlen 64 secured scopeid 0x5 
        inet 192.168.1.167 netmask 0xffffff00 broadcast 192.168.1.255
        nd6 options=201<PERFORMNUD,DAD>
        media: autoselect
        status: active
```

In this example, I should use `192.168.1.167`.

## Step 3: Install and run Krustlet

First, install the latest release of Krustlet following [the install guide](../intro/install.md).

Once you have done that, run the following commands to run Krustlet's WASI provider:

```shell
$ krustlet-wasi --node-ip 172.17.0.1 --pfx-password password
```

In another terminal, run `kubectl get nodes -o wide` and you should see output that looks similar to
below:

```
NAME                 STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
kind-control-plane   Ready    master   3m46s   v1.17.0   172.17.0.2    <none>        Ubuntu 19.10   5.3.0-42-generic   containerd://1.3.2
krustlet             Ready    agent    10s     v1.17.0   172.17.0.1    <none>        <unknown>      <unknown>          mvp
```
