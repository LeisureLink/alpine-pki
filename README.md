# leisurelink/alpine-pki

An [Alpine Linux](https://alpinelinux.org/) based docker image derived from [leisurelink/alpine-base](https://github.com/LeisureLink/alpine-base), adding fundamental PKI certificate request creation on startup.

## Basics

This image is intended as a base image. It adds `/etc/cont-init.d/00-init-pki` startup script which is executed by [the s6 overlay's cont-init process](https://github.com/just-containers/s6-overlay).

Our objective is to have a simple, automated mechanism anabling a newly created container to participate in the environment's public key infrastructure. We feel the following objectives are worthwhile:

1. the container's private key never leaves the container
2. the container generates and signs its own certificate request
3. the container trusts it's host to get a certificate issued on its behalf by the appropriate CA
4. when a container finishes starting up, it has valid signed certificates sufficient to participate in the environment's PKI


`00-init-pki` works with two directories:
* **PKI_ROOT_DIR**  - defaults to `/etc/pki` - directory where `key.pem`, `cert.pem`, and `ca.pem` are stored.
* **PKI_MOUNT_DIR** - defaults to `/run/pki` - directory where the certificate request is written, and where signed certificates are picked up.

On startup, the following occurs:

1. if the private key `/etc/pki/key.pem` is not present it is created
2. if the environment variable `PKI_SUBJECT` is not present the script exits
3. if the certificate `/etc/pki/cert.pem` is not present then a certificate request is created in `/var/pki/$(hostname -f).csr`
4. if a certificate request was created, `00-init-pki` waits for the certificate file `/var/run/$(hostname -f).crt`, timing out after 30 seconds,
5. if a certificate request was created, `00-init-pki` waits for the CA certificate file `/var/run/$(hostname -f).ca`, timing out after 30 seconds
6. as files arrive, they are moved to their permanent home:
  * `/var/run/$(hostname -f).crt` becomes `/etc/pki/cert.pem`
  * `/var/run/$(hostname -f).ca` becomes `/etc/pki/ca.pem`

`00-init-pki` can only be effective if:
* the environment variable `PKI_SUBJECT` is specified,
* **and** if a volume is mounted to `/var/pki`
* **and** another process, probably on the docker host, is waiting for `.csr` files to arrive in the mounted directory. See `leisurelink/alpine-pki-helper` for a docker image that satisfies this last requirement.


### Example

```bash
docker run -it --rm \
       --volume /monitored/csr/path:/var/pki \
       --env PKI_SUBJECT="/C=US/ST=UTAH/localityName=Salt Lake City/O=LeisureLink Inc./OU=Tech Team/CN=example" \
       --env PKI_ALTERNATE_NAMES=my.examples.leisurelink.local,your.examples.leisurelink.local \
       --env PKI_ALTERNATE_IP=192.168.99.100,10.0.0.12 \
       --hostname=c74bdf0d3ad4.examples.leisurelink.local \
       leisurelink/alpine-pki /bin/sh
```

NOTE: we set the environment variable `PKI_ALTERNATE_IP` to the addresses where we've got the container ports mapped. This can be useful depending on the container networking strategy.

NOTE: we set the environment variable `PKI_ALTERNATE_NAMES` to reflect the _host names_ various other parties may refer to when connecting to the container.

Sample output:

```bash
...
cont-init.d] 00-init-pki: executing...
Generating RSA private key, 2048 bit long modulus
............+++
.........................................+++
e is 65537 (0x10001)
Created private key /etc/pki/key.pem and made it owner readonly.
Certificate not found; generating a certificate request.

/C=US/ST=UTAH/localityName=Salt Lake City/O=LeisureLink Inc./OU=Tech Team/CN=example

[alt_names]
IP.1 = 172.17.0.2
IP.2 = 192.168.99.100
IP.3 = 10.0.0.12
DNS.1 = c74bdf0d3ad4.examples.leisurelink.local
DNS.2 = my.examples.leisurelink.local
DNS.3 = your.examples.leisurelink.local
Placed certificate request in /var/pki; waiting for a signed cert....
Moved signed cert to /etc/pki/cert.pem and made it owner readonly.
Removed certificate request from /var/pki.
Waiting for the CA cert....
Moved CA cert to /etc/pki/ca.pem and made it owner readonly.
[cont-init.d] 00-init-pki: exited 0.
...
```

And the certificate request:

```bash
openssl req -noout -text -in c74bdf0d3ad4.examples.leisurelink.local.csr
Certificate Request:
    Data:
        Version: 0 (0x0)
        Subject: C=US, ST=UTAH, L=Salt Lake City, O=LeisureLink Inc., OU=Tech Team, CN=example
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c6:11:5f:ed:08:8e:51:a0:20:ee:4c:8f:f1:9f:
                    f1:ef:1b:16:a7:6b:d2:6a:89:df:4c:d6:f6:2c:cb:
                    af:85:79:b4:7e:2a:3b:b9:e0:da:3d:1a:7f:59:3c:
                    ca:d9:2b:5b:1b:42:7b:c3:83:37:28:94:85:3d:e1:
                    50:5f:fc:f0:44:c4:29:87:cd:fb:79:94:6f:98:3d:
                    2d:c1:29:35:38:0f:a4:54:6d:d9:ca:83:c9:46:03:
                    37:3d:9f:cd:ef:21:21:1a:bc:ef:59:51:37:1c:7f:
                    db:e6:1f:5d:ea:54:9a:a5:1d:12:72:92:6a:4b:15:
                    2a:d1:4f:63:28:89:2c:5d:c3:9a:14:30:26:ae:9c:
                    57:a3:ad:23:7d:dc:15:91:67:4d:5f:34:3e:fb:0b:
                    b8:aa:30:2f:ef:2b:6e:e1:2a:c7:fb:f4:75:0b:60:
                    21:fc:dc:1c:3e:7c:8a:f3:9f:d5:dc:61:a6:83:67:
                    d2:03:4d:3a:83:b4:90:65:5d:53:6f:c0:e2:b8:cf:
                    6a:41:db:d1:21:c2:b1:8b:90:98:47:71:18:6c:80:
                    8a:d7:57:b3:62:82:a4:c7:36:3f:f7:81:e4:72:fe:
                    33:85:93:6a:cc:fb:cd:21:f1:e7:15:65:b6:11:8b:
                    7a:c1:51:e7:eb:ef:23:02:c3:8e:9f:dd:82:06:9c:
                    0b:f5
                Exponent: 65537 (0x10001)
        Attributes:
        Requested Extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            X509v3 Key Usage:
                Digital Signature, Non Repudiation, Key Encipherment
            X509v3 Subject Alternative Name:
                IP Address:172.17.0.2, IP Address:192.168.99.100, IP Address:10.0.0.12, DNS:c74bdf0d3ad4.examples.leisurelink.local, DNS:my.examples.leisurelink.local, DNS:your.examples.leisurelink.local
    Signature Algorithm: sha256WithRSAEncryption
         38:9f:e7:72:a5:1c:d2:a6:15:36:b2:f5:30:05:c2:46:57:51:
         c8:d3:ff:28:1c:60:b2:79:d1:87:5a:79:ec:83:97:da:05:f8:
         e7:17:55:07:8d:e9:3d:9c:2e:e0:e2:b1:3f:95:05:13:97:5e:
         a9:17:c5:d6:56:e3:5e:01:e5:b1:d2:31:ba:ac:b7:fa:4d:91:
         db:6f:4d:c8:30:34:ec:99:0c:53:ba:1e:3d:63:b1:98:c4:10:
         23:56:a1:31:3c:4f:eb:ba:1b:d8:b8:5e:21:f4:8e:f3:fc:a8:
         28:db:d5:b0:46:6f:91:b4:d1:86:59:e4:3b:2f:73:70:cd:51:
         8c:95:a0:ba:66:90:57:3d:5d:e4:e3:90:ed:6b:d6:18:cf:3a:
         78:86:51:7d:b2:20:ca:2e:1e:8e:0c:b8:74:de:e7:f8:ef:7e:
         b3:27:2d:bb:8a:bb:95:85:92:0b:8e:6f:e2:77:3a:7e:45:2d:
         e2:0f:ef:cf:f0:6a:2d:12:b1:4d:dd:eb:35:aa:97:82:d1:dd:
         d8:d0:25:45:a4:2a:e2:aa:8e:c7:98:5c:44:00:88:0b:c8:c2:
         70:e9:6c:d3:71:b7:1b:27:d7:1a:f8:34:53:4c:9a:c3:28:68:
         cc:49:29:f3:7a:1d:50:1d:65:9d:9c:7e:43:24:41:dd:93:a2:
         a6:43:1b:0e
```

## License

[MIT](https://github.com/LeisureLink/alpine-pki/blob/master/LICENSE)
