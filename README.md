# 🔐 **Post-Quantum TLS Lab — Step-by-Step Guide (Apache + OQS + NIST ML-KEM / ML-DSA)**

This guide builds a fully functional lab that serves a website over **HTTPS with Post-Quantum TLS**, using:

- **Classical HTTPS certificates** (RSA/ECDSA).
- **Post-Quantum certificates** (ML-DSA, ML-KEM).
- **Hybrid TLS negotiation** compatible with NIST-standardized PQC algorithms.

We will use **Apache (httpd)** compiled with **OpenQuantumSafe OSSL3**.

# 🛠️ **Phase 1 — Cleanup & Environment Preparation**

First ensure no previous containers or files interfere.

```bash
# 1. Remove old containers
docker rm -f pqc-server 2>/dev/null || true

# 2. Remove previous lab files (avoid conflicts)
sudo rm -rf /opt/pqc-lab

# 3. Recreate clean directory structure
sudo mkdir -p /opt/pqc-lab/certs
sudo mkdir -p /opt/pqc-lab/html
sudo mkdir -p /opt/pqc-lab/config

# 4. Relax permissions so the non-root container user can write
sudo chmod -R 777 /opt/pqc-lab
```

# 🔐 **Phase 2 — Generate PQC Certificates (NIST-named algorithms)**

> **🍎 Apple Silicon (M1 / M2 / M3) note**
> The OQS images are published only for `linux/amd64`. If you're on an M-series Mac, append `--platform linux/amd64` to every `docker run` and `docker build` command in this guide. Otherwise Docker will either fail with `exec format error` or fall back to slow QEMU emulation.

Enter the OQS OpenSSL container:

```bash
docker run -it --rm -v /opt/pqc-lab/certs:/certs openquantumsafe/oqs-ossl3 sh
```

Run:

```bash
cd /certs

# 1. Create PQC Root CA (Pure PQC: ML-DSA-44, standardized as FIPS 204)
openssl req -x509 -new -newkey mldsa44 -keyout ca_pqc.key -out ca_pqc.crt -nodes -subj "/CN=PQC Lab Root CA" -days 365

# 2. Create hybrid server key + CSR (P-256 + ML-DSA-44)
#    The SAN (subjectAltName) is required by modern TLS clients and silences
#    Apache's "server certificate does NOT include an ID which matches the server name" warning.
openssl req -new -newkey p256_mldsa44 -keyout server.key -out server.csr -nodes \
    -subj "/CN=localhost" \
    -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

# 3. Sign the server certificate using our PQC CA
#    `-copy_extensions copy` is required to propagate the SAN from the CSR
#    into the final certificate (openssl drops CSR extensions by default).
openssl x509 -req -in server.csr -out server.crt -CA ca_pqc.crt -CAkey ca_pqc.key \
    -CAcreateserial -days 365 -copy_extensions copy
```

Exit with `exit` or `CTRL+D`.

# 🌐 **Phase 3 — Apache Configuration**

## 1. HTML Test Page

Create `/opt/pqc-lab/html/index.html`:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>PQC Lab — Simple Page</title>
  </head>
  <body>
    <h1>Post-Quantum TLS Lab — HTTPS Server</h1>
    <p>Post-Quantum TLS running on port <strong>4433</strong>.</p>
    <p>This static page confirms TLS negotiation & certificate validation.</p>
  </body>
</html>
```

## 2. Apache Configuration (`pqc.conf`)

Create `/opt/pqc-lab/config/pqc.conf`:

```apache
Listen 4433

SSLProtocol -all +TLSv1.3
SSLSessionCache "shmcb:/opt/httpd/logs/ssl_scache(512000)"
SSLSessionCacheTimeout 300

<VirtualHost *:4433>
    ServerName localhost
    DocumentRoot "/opt/httpd/htdocs"

    SSLEngine on
    SSLCertificateFile "/opt/httpd/pki/server.crt"
    SSLCertificateKeyFile "/opt/httpd/pki/server.key"

    <Directory "/opt/httpd/htdocs">
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
```

> **ℹ️ Why these global directives?**
> `SSLSessionCache` is **required** by Apache to start when SSL is enabled — omitting it makes httpd refuse to launch. `SSLProtocol -all +TLSv1.3` makes the lab deterministic: only TLS 1.3 negotiates PQC algorithms, so we explicitly disable everything else.

# 🚀 **Phase 4 — Build & Run the PQC Apache Server**

Create `/opt/pqc-lab/Dockerfile`:

```dockerfile
FROM openquantumsafe/httpd:latest

USER root

# Override the image's default SSL config with ours.
# The OQS image starts httpd with `-f httpd-conf/httpd.conf`, which
# includes `httpd-conf/httpd-ssl.conf`. Replacing that file is the
# cleanest way to make our vhost the active one without leaving the
# default vhost behind on the same port.
COPY config/pqc.conf /opt/httpd/httpd-conf/httpd-ssl.conf

RUN mkdir -p /opt/httpd/logs /opt/httpd/htdocs && chown -R daemon:daemon /opt/httpd/htdocs /opt/httpd/logs

USER daemon
```

Build and run:

```bash
cd /opt/pqc-lab

sudo chmod -R 777 .

docker rm -f pqc-server 2>/dev/null || true

docker build -t my-pqc-server .

docker run -d \
  --name pqc-server \
  --rm \
  -p 4433:4433 \
  -v /opt/pqc-lab/certs/server.crt:/opt/httpd/pki/server.crt:ro \
  -v /opt/pqc-lab/certs/server.key:/opt/httpd/pki/server.key:ro \
  -v /opt/pqc-lab/html:/opt/httpd/htdocs:ro \
  my-pqc-server
```

Check logs:

```bash
docker logs pqc-server
```

# 🧪 **Phase 5 — Verify TLS (with OQS curl)**

Browsers do not support ML-KEM or ML-DSA yet, so we test with OQS-enabled curl.

Run the client container:

```bash
docker run -it --rm -v /opt/pqc-lab/certs/ca_pqc.crt:/ca.crt --network host openquantumsafe/curl sh
```

### 1. Test HTTPS + PQC Negotiation

```bash
curl https://localhost:4433 --cacert /ca.crt -v
```

Look for a line like:

```
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / mlkem768 / p256_mldsa44
```

This means:

| Layer                     | Algorithm      | PQC Status   | Meaning                                    |
| ------------------------- | -------------- | ------------ | ------------------------------------------ |
| **Key Exchange (KEM)**    | `mlkem768`     | PQC-pure     | Prevents harvest-now-decrypt-later         |
| **Certificate Signature** | `p256_mldsa44` | Hybrid       | Server authentication uses classical + PQC |
| **Record Encryption**     | `AES_256_GCM`  | Quantum-safe | AES-256 remains safe even vs Grover        |

#### 🔑 A. Key Exchange — `mlkem768`

- Fully PQC
- Standardized as **NIST FIPS 203**
- Protects the symmetric session key from future quantum attacks.

#### 🔏 B. Certificate Authentication — `p256_mldsa44`

- **Hybrid signature**:

  - Classical P-256
  - PQC ML-DSA-44 (NIST FIPS 204)

- Attacker must break **both** to impersonate your server.

#### 🔒 C. Symmetric Encryption — `AES_256_GCM`

- Not replaced by PQC algorithms
- AES-256 already resists quantum attacks (Grover → 128-bit effective security)

🎉 **Congratulations!** You have built a **fully functional Post-Quantum TLS lab**

---

<div align="center">
  <small>
    Made with ❤️ by <a target="_blank" href="https://www.linkedin.com/in/jalvarezz13/">jalvarezz13</a>
  </small>
</div>
