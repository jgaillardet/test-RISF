# RISF

## Table of Contents

[Install docker & k3s](#install-docker-and-k3s)  
[Image build & deployments](#image-build-and-deployments)  
[PKI Setup](#pki-setup)  
[Expose our services with Traefik](#expose-our-services-with-traefik)  

## Install docker and k3s

First, let's install Docker using the [official doc](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository).  
Then we add the current user to the docker group:

```bash
╰─ sudo usermod -aG docker $USER
```

You can now logout and login again for the usermod command to take effect.  
Now we will install k3s, retrieve the kubeconfig and make an alias `k` for kubectl as I'm lazy.

```bash
╰─ curl -sfL https://get.k3s.io | sh -s - --docker
╰─ mkdir -p ~/.kube
╰─ export KUBECONFIG=~/.kube/config
╰─ sudo k3s kubectl config view --raw > "$KUBECONFIG"
╰─ chmod 600 "$KUBECONFIG"
╰─ echo 'export KUBECONFIG=~/.kube/config' >> ~/.zshrc
╰─ echo 'alias k=kubectl >> ~/.zshrc
╰─ source ~/.zshrc
```

## Image build and deployments

I'm used to deny writes on the root filesystem in Kubernetes as it allow us to control where the files are writen (emptyDir, Persitentvolume ...).  
By doing so it also eliminates disk alerts on worker nodes and reduce the RUN.  
I will build the RISF image and generate a conf for both deployment accordingly.

### RISF image build

Let's build a rootless Nginx image that will serve our custom index.html.
You can check the [Dockerfile](image-risf/Dockerfile) and the nginx [configuration](image-risf/files/conf/nginx.conf):

```bash
╰─ docker build image-risf -t hello-risf:1.0.0
[+] Building 6.0s (9/9) FINISHED                                                                         docker:default
 => [internal] load build definition from Dockerfile                                                               0.0s
 => => transferring dockerfile: 294B                                                                               0.0s
 => [internal] load metadata for docker.io/library/nginx:1.27.4                                                    2.9s
 => [internal] load .dockerignore                                                                                  0.0s
 => => transferring context: 2B                                                                                    0.0s
 => [1/4] FROM docker.io/library/nginx:1.27.4@sha256:9d6b58feebd2dbd3c56ab5853333d627cc6e281011cfd6050fa4bcf2072c  2.6s
 => => resolve docker.io/library/nginx:1.27.4@sha256:9d6b58feebd2dbd3c56ab5853333d627cc6e281011cfd6050fa4bcf2072c  0.0s
 => => sha256:9d6b58feebd2dbd3c56ab5853333d627cc6e281011cfd6050fa4bcf2072c9496 10.27kB / 10.27kB                   0.0s
 => => sha256:28edb1806e63847a8d6f77a7c312045e1bd91d5e3c944c8a0012f0b14c830c44 2.29kB / 2.29kB                     0.0s
 => => sha256:7cf63256a31a4cc44f6defe8e1af95363aee5fa75f30a248d95cae684f87c53c 28.22MB / 28.22MB                   0.4s
 => => sha256:b52e0b094bc0e26c9eddc9e4ab7a64ce0033c3360d8b7ad4ff4132c4e03e8f7b 8.58kB / 8.58kB                     0.0s
 => => sha256:bf9acace214a6c23630803d90911f1fd7d1ba06a3083f0a62fd036a6d1d8e274 43.95MB / 43.95MB                   0.8s
 => => sha256:513c3649bb1480ca9a04c73f320b6b5a909e24e4ac18ae72fd56b818241d6730 626B / 626B                         0.8s
 => => extracting sha256:7cf63256a31a4cc44f6defe8e1af95363aee5fa75f30a248d95cae684f87c53c                          1.2s
 => => sha256:d014f92d532d416c7b9eadb244f14f73fdb3d2ead120264b749e342700824f3c 957B / 957B                         0.9s
 => => sha256:943ea0f0c2e42ccacc72ac65701347eadb2b0cb22828fac30f1400bba3d37088 1.21kB / 1.21kB                     1.1s
 => => sha256:103f50cb3e9f200431b555078cce5e8df3db6ddc2e54d714a10b994e430e98a3 1.40kB / 1.40kB                     1.2s
 => => sha256:9dd21ad5a4a6a856d82bb6bb6147c30ad90a9768c3651c55775354e7649bc74d 406B / 406B                         1.0s
 => => extracting sha256:bf9acace214a6c23630803d90911f1fd7d1ba06a3083f0a62fd036a6d1d8e274                          0.7s
 => => extracting sha256:513c3649bb1480ca9a04c73f320b6b5a909e24e4ac18ae72fd56b818241d6730                          0.0s
 => => extracting sha256:d014f92d532d416c7b9eadb244f14f73fdb3d2ead120264b749e342700824f3c                          0.0s
 => => extracting sha256:9dd21ad5a4a6a856d82bb6bb6147c30ad90a9768c3651c55775354e7649bc74d                          0.0s
 => => extracting sha256:943ea0f0c2e42ccacc72ac65701347eadb2b0cb22828fac30f1400bba3d37088                          0.0s
 => => extracting sha256:103f50cb3e9f200431b555078cce5e8df3db6ddc2e54d714a10b994e430e98a3                          0.0s
 => [internal] load build context                                                                                  0.0s
 => => transferring context: 813B                                                                                  0.0s
 => [2/4] RUN chown -R nginx:nginx /etc/nginx/                                                                     0.4s
 => [3/4] COPY --chown=nginx files/conf/nginx.conf /etc/nginx/nginx.conf                                           0.0s
 => [4/4] COPY --chown=nginx files/www/index.html /usr/share/nginx/html                                            0.0s
 => exporting to image                                                                                             0.0s
 => => exporting layers                                                                                            0.0s
 => => writing image sha256:4a33822c9c20731a94641dfb73bbf3331e24245af23c2f49b60f32252feb3be4                       0.0s
 => => naming to docker.io/library/hello-risf:1.0.0                                                                0.0s
```

Let's launch it:

```bash
╰─ docker run -it hello-risf:1.0.0 whoami
nginx
╰─ docker run -d -p 8080:8080 hello-risf:1.0.0
5091c2f3401ba55509a33726e5de62c11443ed1088039243b4f35f569029880c
╰─ curl localhost:8080
<h1>HELLO RISF</h1>%
╰─ docker stop 509
509
```

### RISF Deployment

Let's [deploy](apps/risf/deployment.yml) and [expose](apps/risf/service.yml) the image that we built earlier:

```bash
╰─ k apply -f apps/risf/deployment.yml -f apps/risf/service.yml
deployment.apps/nginx-risf created
service/nginx-risf created
╰─ k get pods
NAME                          READY   STATUS    RESTARTS   AGE
nginx-risf-5c5667d674-rf24g   1/1     Running   0          4s
nginx-risf-5c5667d674-zlj49   1/1     Running   0          4s
╰─ k describe pod nginx-risf-98d74cc47-fjd6z | tail -n 6
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  30s   default-scheduler  Successfully assigned default/nginx-risf-5c5667d674-rf24g to desktop-2shl1em
  Normal  Pulled     30s   kubelet            Container image "hello-risf:1.0.0" already present on machine
  Normal  Created    30s   kubelet            Created container: nginx
  Normal  Started    30s   kubelet            Started container nginx
╰─ k port-forward service/nginx-risf 8080 # sudo apt install socat if you don't have the package
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
Handling connection for 8080 # From the curl command
╰─ curl localhost:8080 # In another terminal
<h1>HELLO RISF</h1>%
```

It works !

### ITSF Deployment

We are going to use the official rootless image for Nginx.  
Let's [deploy](apps/itsf/deployment.yml) and [expose](apps/itsf/service.yml) it as we did for the RISF deployment.  
We will place the Nginx configuration in a [configmap](apps/itsf/configmap-conf-nginx.yml), as the rootfs is readonly, and create a [PVC](apps/itsf/pvc.yml) in order to spawn the volume that will contain the index.html:

```bash
╰─ k apply -f apps/itsf/configmap-conf-nginx.yml -f apps/itsf/pvc.yml -f apps/itsf/deployment.yml -f apps/itsf/service.yml
configmap/nginx-config created
persistentvolumeclaim/pvc-nginx-itsf created
deployment.apps/nginx-itsf created
service/nginx-itsf created
╰─ k get pods
NAME                          READY   STATUS    RESTARTS   AGE
nginx-itsf-768f44b77b-b7gxs   1/1     Running   0          26s
nginx-itsf-768f44b77b-j4bzk   1/1     Running   0          26s
nginx-risf-5c5667d674-rf24g   1/1     Running   0          3m3s
nginx-risf-5c5667d674-zlj49   1/1     Running   0          3m3s
╰─ k describe pod nginx-itsf-768f44b77b-b7gx | tail -n 7
  ----    ------     ----  ----               -------
  Normal  Scheduled  49s   default-scheduler  Successfully assigned default/nginx-itsf-768f44b77b-b7gxs to desktop-2shl1em
  Normal  Pulling    48s   kubelet            Pulling image "nginxinc/nginx-unprivileged:1.27.4"
  Normal  Pulled     35s   kubelet            Successfully pulled image "nginxinc/nginx-unprivileged:1.27.4" in 12.722s (12.722s including waiting). Image size: 192008218 bytes.
  Normal  Created    35s   kubelet            Created container: nginx
  Normal  Started    35s   kubelet            Started container nginx
╰─ k describe pvc pvc-nginx-itsf | tail -n 6
  Type    Reason                 Age                From                                                                                                Message
  ----    ------                 ----               ----                                                                                                -------
  Normal  WaitForFirstConsumer   2m                persistentvolume-controller                                                                         waiting for first consumer to be created before binding
  Normal  Provisioning           2m                rancher.io/local-path_local-path-provisioner-5b5f758bcf-6b8kg_477508dd-7646-4df4-bc7f-e87dad41bec1  External provisioner is provisioning volume for claim "default/pvc-nginx-itsf"
  Normal  ExternalProvisioning   2m                persistentvolume-controller                                                                         Waiting for a volume to be created either by the external provisioner 'rancher.io/local-path' or manually by the system administrator. If volume creation is delayed, please verify that the provisioner is running and correctly registered.
  Normal  ProvisioningSucceeded  2m                rancher.io/local-path_local-path-provisioner-5b5f758bcf-6b8kg_477508dd-7646-4df4-bc7f-e87dad41bec1  Successfully provisioned volume pvc-89a17dea-cd0d-49d0-ad85-0953e562c697
╰─ k cp apps/itsf/www-files/index.html nginx-itsf-768f44b77b-b7gxs:/usr/share/nginx/html/index.html  
╰─ k port-forward service/nginx-itsf 8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
Handling connection for 8080 # From the curl command
╰─ curl localhost:8080 # In another terminal
<h1>HELLO ITSF</h1>%
```

For this test I went for a dead simple `kubectl cp` in order to copy the index.html in the PersitentVolume but for real workload a `Job` or a `CronJob` is more suited.

## PKI Setup

At home I use [mkcert](https://github.com/FiloSottile/mkcert) but it doesn't show how generate and sign certificates.  
For this part, we will follow the [pki-tutorial](https://pki-tutorial.readthedocs.io/en/latest/simple/).

Note: for this test, the passphrase for all keys is `risf`. All confs can be found under [pki/etc](pki/etc).

### Root CA & Signing CA

Let's create the folder structure and init the DB:

```bash
╰─ mkdir pki && cd pki
╰─ mkdir -p ca/root-ca/private ca/root-ca/db crl certs
╰─ chmod 700 ca/root-ca/private
╰─ cp /dev/null ca/root-ca/db/root-ca.db
╰─ echo 01 > ca/root-ca/db/root-ca.crt.srl
╰─ echo 01 > ca/root-ca/db/root-ca.crl.srl
```

We will edit the default [root-ca.conf](https://pki-tutorial.readthedocs.io/en/latest/simple/root-ca.conf.html) but will change a few things in `ca_dn` and `match_pol`:

```text
[ ca_dn ]
countryName             = "FR"
stateOrProvinceName     = "Alpes-Maritimes"
localityName            = "Nice"
organizationName        = "RISF"
organizationalUnitName  = "RISF Root CA"
...
[ match_pol ]
countryName             = match
stateOrProvinceName     = match
localityName            = match
organizationName        = match                 # Must match 'RISF'
organizationalUnitName  = optional              # Included if present
```

Let's generate a CSR/key pair and the root CA cert:

```bash
╰─ openssl req -new -config etc/root-ca.conf -out ca/root-ca.csr -keyout ca/root-ca/private/root-ca.key
.......+.+..+.............+...+......+...............+...........+.........+.+...+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..+....+.....+......+.......+..+......+.......+......+......+.....+...+...+.......+...+..+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*...+...+.........+.....+.+..+......+.+.........+.....+......+.........+....+..+.............+..+....+...........+.........+..........+......+.....+...............+.+..+...+....+...........+...+.......+............+...+.................+.......+...+...+..+......+.+...+......+.....+.+........+.+..+.........+...+.............+...+..............+.+...........+..........+...........+...+....+..+...+............+......+............+.............+...+.........+...+.........+.....................+..+.+......+......+...+.....+.+..................+...+........+...............+....+...+.....+.......+..+......+...............+...+.+...+..+.+..............+....+......+...+...............+..+.............+..+.+..+.+..+............+...+....+........+.+.....+....+......+...........+.+.....+....+...............+.........+.....+.........+...+.......+..+....+.....+....+...+........+....+...+..+.+...........+.+..+.............+.....+....+...+..+.+...+.....+....+.....+...+....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..........+...+......+...+............+....+......+......+..+...+....+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*............+...+.......+...........+....+..+.+.....+..........+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
╰─ openssl ca -selfsign -config etc/root-ca.conf -in ca/root-ca.csr -out ca/root-ca.crt -extensions root_ca_ext
Using configuration from etc/root-ca.conf
Enter pass phrase for ./ca/root-ca/private/root-ca.key:
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number:
            01:9a:c6:0a:52:ed:43:0f:32:5e:b2:c1:2b:8c:bd:ed:41:f2:84:bb
        Validity
            Not Before: Mar 10 16:30:56 2025 GMT
            Not After : Mar 10 16:30:56 2035 GMT
        Subject:
            countryName               = FR
            stateOrProvinceName       = Alpes-Maritimes
            localityName              = Nice
            organizationName          = RISF
            organizationalUnitName    = RISF Root CA
        X509v3 extensions:
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Subject Key Identifier:
                39:6B:12:ED:A2:3B:D8:39:55:A2:E0:65:26:AE:D9:9F:02:30:A9:60
            X509v3 Authority Key Identifier:
                39:6B:12:ED:A2:3B:D8:39:55:A2:E0:65:26:AE:D9:9F:02:30:A9:60
Certificate is to be certified until Mar 10 16:30:56 2035 GMT (3652 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

Before generating the Signing CA cert, let's modify the default [signing-ca.conf](https://pki-tutorial.readthedocs.io/en/latest/simple/signing-ca.conf.html):

```text
[ ca_dn ]
countryName             = "FR"
stateOrProvinceName     = "Alpes-Maritimes"
localityName            = "Nice"
organizationName        = "RISF"
organizationalUnitName  = "RISF Signing CA"
...
[ match_pol ]
countryName             = match
stateOrProvinceName     = match
localityName            = match
organizationName        = match                 # Must match 'RISF'
organizationalUnitName  = optional              # Included if present
emailAddress            = optional              # Included if present
```

We can now create the folder structure for the signing CA, init the DB, create a CSR/key pair and finally generate our Signing CA certificate:

```bash
╰─ mkdir -p ca/signing-ca/private ca/signing-ca/db crl certs
╰─ chmod 700 ca/signing-ca/private
╰─ cp /dev/null ca/signing-ca/db/signing-ca.db
╰─ echo 01 > ca/signing-ca/db/signing-ca.crt.srl
╰─ echo 01 > ca/signing-ca/db/signing-ca.crl.srl
╰─ openssl req -new -config etc/signing-ca.conf -out ca/signing-ca.csr -keyout ca/signing-ca/private/signing-ca.key
.....+............+...+...+....+.....+.........+.+...+..+......+...+......+..................+.+..+...+....+......+......+...+......+...........+...+......+.......+........+.+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.....+....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+...........+.......+............+...........+.........+.+.....+......+...............+.+.....+.......+.....+............+....+..+...+.........................+.....+.+...+...+.....+...+.+...+......+...........+.+..+...+...+....+...+....................+...............+....+...........+...............+....+.........+......+..................+...........+.+...........+...+....+..+....+..............+.+..+...+....+...+........+....+......+.....+.+.....+...+..................+....+...........+............+...............+.......+...+...+..+...+.......+..+.+..+......+......+....+..+...+.+......+...........................+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
.+.....+..........+.................+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.........+....+...+......+..+..........+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..+......+..+...+.+.....+................+..+.......+.....+.+.........+......+.....+.......+..+...+..........+.....+.+.....+.+......+..+...+...+......+...+.......+.....+...+.+.........+........+....+.........+.....+................+...+......+......+......+..+...+...+....+.................+...+............+..........+.....+....+.................+.........+...+.+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
╰─ openssl ca -config etc/root-ca.conf -in ca/signing-ca.csr -out ca/signing-ca.crt -extensions signing_ca_ext
Using configuration from etc/root-ca.conf
Enter pass phrase for ./ca/root-ca/private/root-ca.key:
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number:
            46:35:39:75:5e:36:40:4c:a9:45:fb:6c:8b:4d:fb:69:04:7d:aa:b8
        Validity
            Not Before: Mar 10 16:32:37 2025 GMT
            Not After : Mar 10 16:32:37 2035 GMT
        Subject:
            countryName               = FR
            stateOrProvinceName       = Alpes-Maritimes
            localityName              = Nice
            organizationName          = RISF
            organizationalUnitName    = RISF Signing CA
        X509v3 extensions:
            X509v3 Key Usage: critical
                Certificate Sign, CRL Sign
            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:0
            X509v3 Subject Key Identifier:
                AC:BC:69:49:05:7F:58:2F:04:35:1B:14:59:A7:EF:9C:16:36:DA:53
            X509v3 Authority Key Identifier:
                39:6B:12:ED:A2:3B:D8:39:55:A2:E0:65:26:AE:D9:9F:02:30:A9:60
Certificate is to be certified until Mar 10 16:32:37 2035 GMT (3652 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

### RISF & ITSF Certificates

As for the Root CA & the Signing CA, let's modify the default [server.conf](https://pki-tutorial.readthedocs.io/en/latest/simple/server.conf.html):

```text
[ req ]
default_bits            = 2048
default_md              = sha256
req_extensions          = server_reqext
distinguished_name      = server_dn
prompt                  = no

[ server_reqext ]
keyUsage                = critical,digitalSignature,keyEncipherment
extendedKeyUsage        = serverAuth,clientAuth
subjectKeyIdentifier    = hash
subjectAltName          = $ENV::SAN

[ server_dn ]
countryName             = "FR"
stateOrProvinceName     = "Alpes-Maritimes"
localityName            = "Nice"
organizationName        = "RISF"
```

We will hardcode most of the fields and the subjectAltName will come from an environment variable. We set `prompt` to no in order to speed things up as the fields are the same for both deployments.

Let's create the CSR/key pair for `hello-risf.local.domain`:

```bash
╰─ SAN="DNS:hello-risf.local.domain" openssl req -new -keyout certs/hello-risf.key -out certs/hello-risf.csr -config etc/server.conf
...+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*...+............+.+...+..+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.....+.+......+...+...+.....+...+.+.....+...+......+.........+..........+..............................+..+.+.........+...+..+..........+...+.....+.............+.................+......+......+...+.......+..................+........+.......+...........+.+..............................+......+...+..+.........+...+......+.+......+............+...............+......+..+...+.+........+.......+.....+....+.....+.+.....+......+.........+.........+.....................+.+.....+....+.....+.........+.+...+......+..+...+....+......+.....+.......+........+....+...........+............+.......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..........+..+.............+..+...+...+....+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*...........+....+....................+...................+..+.+...+.................+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
```

Let's retrieve in the CSR the values that we have either hardcoded in the server.conf or passed as a variable:

```bash
╰─ openssl req -text -in certs/hello-risf.csr | grep "Subject Alternative Name" -A 1
                X509v3 Subject Alternative Name:
                    DNS:hello-risf.local.domain
╰─ openssl req -text -in certs/hello-risf.csr | grep "Subject:"
        Subject: C = FR, ST = Alpes-Maritimes, L = Nice, O = RISF                  
```

All infos are stored in the CSR, let's generate the certificate:

```bash
╰─ openssl ca -config etc/signing-ca.conf -in certs/hello-risf.csr -out certs/hello-risf.crt -extensions server_ext
Using configuration from etc/signing-ca.conf
Enter pass phrase for ./ca/signing-ca/private/signing-ca.key:
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number:
            0d:09:2a:3b:f1:bd:88:b5:6b:6b:96:af:b4:82:04:ae:50:fc:8f:f9
        Validity
            Not Before: Mar 10 16:33:46 2025 GMT
            Not After : Mar 10 16:33:46 2027 GMT
        Subject:
            countryName               = FR
            stateOrProvinceName       = Alpes-Maritimes
            localityName              = Nice
            organizationName          = RISF
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Basic Constraints:
                CA:FALSE
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Subject Key Identifier:
                06:87:D8:1B:BB:D7:89:1F:E7:DB:52:77:56:41:6E:07:5B:50:A7:65
            X509v3 Authority Key Identifier:
                AC:BC:69:49:05:7F:58:2F:04:35:1B:14:59:A7:EF:9C:16:36:DA:53
            X509v3 Subject Alternative Name:
                DNS:hello-risf.local.domain
Certificate is to be certified until Mar 10 16:33:46 2027 GMT (730 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

Let's repeat the previous steps for `hello-itsf.local.domain`:

```bash
╰─ SAN="DNS:hello-itsf.local.domain" openssl req -new -keyout certs/hello-itsf.key -out certs/hello-itsf.csr -config etc/server.conf
....+.+..+...............+.......+......+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.....+....+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.............+...+............+.....+............+...+...+....+...+..+......+......+.+..............+....+.........+..+.+...............+...+..+.........+.+...+..+.........+....+.........+...........+.+.........+.....+...................+.....+.......+...+...+....................+....+...+........+...+.......+............+.....+.......+...........+..........+.........+..............+......+...+...............+..........+......+...........+.+..+.+...........+.+........+.+.....+......+.........+.+...+.....+.........+.+..............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
.....+...+..+...............+......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*......+..+...+.+...........+...+...............+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+..+......+.......+...+.....................+.....+...+......+....+...+..+...+.........+...+......+.+........+.......+.....+...............+....+...........+.+...+..+.......+..+...+.......+.........+......+...............+.....+.........+....+.....+...+...+....+..+.+.....+...........................+...+...+.+......+..+.+............+...+...............+.........+............+....................+.+..+.+..+.............+.....+.........+.......+.....+.+...+.....+.+.....+......+...............+....+...+..+.............+...+.....+....+.....+...+......+....+.....+...............+.......+......+.....+....+.....+......+.............+..+..........+.....+....+.....+...+.............+...+.....+.......+..+............................+...............+........+....+..+....+...............+..+....+...........+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
╰─ openssl ca -config etc/signing-ca.conf -in certs/hello-itsf.csr -out certs/hello-itsf.crt -extensions server_ext
Using configuration from etc/signing-ca.conf
Enter pass phrase for ./ca/signing-ca/private/signing-ca.key:
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number:
            2e:0d:45:1a:e5:8f:d2:bb:92:2e:b8:b9:25:fd:f5:02:50:84:5a:46
        Validity
            Not Before: Mar 10 16:34:08 2025 GMT
            Not After : Mar 10 16:34:08 2027 GMT
        Subject:
            countryName               = FR
            stateOrProvinceName       = Alpes-Maritimes
            localityName              = Nice
            organizationName          = RISF
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Basic Constraints:
                CA:FALSE
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Subject Key Identifier:
                BD:49:EB:63:04:69:E8:FA:52:5C:7C:A7:41:CA:1C:EA:A4:30:02:71
            X509v3 Authority Key Identifier:
                AC:BC:69:49:05:7F:58:2F:04:35:1B:14:59:A7:EF:9C:16:36:DA:53
            X509v3 Subject Alternative Name:
                DNS:hello-itsf.local.domain
Certificate is to be certified until Mar 10 16:34:08 2027 GMT (730 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

## Expose our services with Traefik

K3S comes with Traefik so we don't need to install something else:

```bash
╰─ k get pods -n kube-system | grep traefik
helm-install-traefik-crd-z5p2k            0/1     Completed   0          31m
helm-install-traefik-mphvl                0/1     Completed   1          31m
svclb-traefik-b792c4dd-l6ksq              2/2     Running     0          31m
traefik-5cbdcf97f4-pklbb                  1/1     Running     0          31m
```

Now we need to create a secret with the key and cert that we have produced earlier, but first we need to decrypt the keys. Then let's create the cert secrets for both deployments and save those (for this test) under apps/{risf,itsf}/secret-cert.yml. Real certs/keys are stored in Vault or any other secrets management tool:

```bash
╰─ openssl rsa -in certs/hello-risf.key -out certs/hello-risf-decrypt.key
Enter pass phrase for certs/hello-risf.key:
writing RSA key
╰─ openssl rsa -in certs/hello-itsf.key -out certs/hello-itsf-decrypt.key
Enter pass phrase for certs/hello-itsf.key:
writing RSA key
╰─ cd ..
╰─ k create secret tls risf-cert --cert=pki/certs/hello-risf.crt --key=pki/certs/hello-risf-decrypt.key
secret/risf-cert created
╰─ k get secrets risf-cert -o yaml > apps/risf/secret-cert.yml
╰─ k create secret tls itsf-cert --cert=pki/certs/hello-itsf.crt --key=pki/certs/hello-itsf-decrypt.key
secret/itsf-cert created
╰─ k get secrets itsf-cert -o yaml > apps/itsf/secret-cert.yml
╰─ k get secrets risf-cert -o yaml > apps/risf/secret-cert.yml
```

Now let's create both Ingress:

```bash
╰─ k apply -f apps/risf/ingress.yml
ingress.networking.k8s.io/itsf-ingress created
╰─ k apply -f apps/itsf/ingress.yml
ingress.networking.k8s.io/ingress-itsf created
╰─ k get ing
NAME           CLASS    HOSTS                     ADDRESS       PORTS     AGE
itsf-ingress   <none>   hello-itsf.local.domain   172.22.86.1   80, 443   17m
risf-ingress   <none>   hello-risf.local.domain   172.22.86.1   80, 443   17m
```

Let's add a record for hello-*.local.domain in `/etc/hosts`:

```bash
╰─ echo "172.22.86.1     hello-itsf.local.domain hello-risf.local.domain" | sudo tee -a /etc/hosts
172.22.86.1     hello-itsf.local.domain hello-risf.local.domain
╰─ cat /etc/hosts | grep hello
172.22.86.1     hello-itsf.local.domain hello-risf.local.domain
```

If we try to reach one of our ingress we encounter an error as the issuer is not known:

```bash
╰─ curl "https://hello-risf.local.domain"
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

If we use the insecure flag -k along with a -v flag to add more verbosity, we can see that we are using the certificate issued by our Signing CA and that we are reaching the RISF deployment:

```bash
╰─ curl -kv "https://hello-risf.local.domain"
*   Trying 172.22.86.1:443...
* Connected to hello-risf.local.domain (172.22.86.1) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* TLSv1.0 (OUT), TLS header, Certificate Status (22):
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS header, Certificate Status (22):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS header, Finished (20):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.2 (OUT), TLS header, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256
* ALPN, server accepted to use h2
* Server certificate:
*  subject: C=FR; ST=Alpes-Maritimes; L=Nice; O=RISF  # DN from risf-cert secret
*  start date: Mar 10 16:33:46 2025 GMT
*  expire date: Mar 10 16:33:46 2027 GMT
*  issuer: C=FR; ST=Alpes-Maritimes; L=Nice; O=RISF; OU=RISF Signing CA # DN of the Signing CA
*  SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
* Using HTTP2, server supports multiplexing
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* Using Stream ID: 1 (easy handle 0x559490a59eb0)
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
> GET / HTTP/2
> Host: hello-risf.local.domain
> user-agent: curl/7.81.0
> accept: */*
>
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* Connection state changed (MAX_CONCURRENT_STREAMS == 250)!
* TLSv1.2 (OUT), TLS header, Supplemental data (23):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* TLSv1.2 (IN), TLS header, Supplemental data (23):
< HTTP/2 200
< accept-ranges: bytes
< content-type: text/html
< date: Mon, 10 Mar 2025 16:36:13 GMT
< etag: "67ce94ff-13"
< last-modified: Mon, 10 Mar 2025 07:30:07 GMT
< server: nginx/1.27.4
< content-length: 19
<
* TLSv1.2 (IN), TLS header, Supplemental data (23):
* Connection #0 to host hello-risf.local.domain left intact
<h1>HELLO RISF</h1>
```

Now we add our signing CA to the truststore:

```bash
╰─ sudo cp pki/ca/signing-ca.crt /usr/local/share/ca-certificates
╰─ sudo update-ca-certificates
Updating certificates in /etc/ssl/certs...
rehash: warning: skipping ca-certificates.crt,it does not contain exactly one certificate or CRL
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...
done.
```

Yay it works:

```bash
╰─ curl https://hello-risf.local.domain
<h1>HELLO RISF</h1>%
╰─ curl https://hello-itsf.local.domain
<h1>HELLO ITSF</h1>%
```
