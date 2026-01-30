
# Creating a Local CA + Wildcard TLS Certificate for `*.casa.local` (k3s Homelab)

This guide provides **step‑by‑step commands** to create:
- A **private Root Certificate Authority (CA)**
- A **wildcard TLS certificate** for `*.casa.local`
- A **certificate chain** suitable for Traefik
- A **Kubernetes TLS secret** for your k3s cluster

All commands are fully copy‑paste ready.

---

## 🟦 1. Create a Root CA

The Root CA will be trusted on your devices (Android, Linux, Windows) and will sign all service certificates inside your homelab.

### **1.1 Generate the CA private key**
```bash
openssl genrsa -out casa_local_ca.key 4096
```

### **1.2 Create the CA certificate**
```bash
openssl req -x509 -new -nodes   -key casa_local_ca.key   -sha256 -days 3650   -out casa_local_ca.crt   -subj "/C=ES/ST=Extremadura/L=Santibanez/CN=Casa Local Homelab CA"
```

> The file `casa_local_ca.crt` is what you will install into: Android, Linux, Windows, macOS.

---

## 🟦 2. Create a Wildcard Certificate for `*.casa.local`

### **2.1 Create an OpenSSL configuration file**
Create the file `casa-local-cert.cnf`:

```ini
[req]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn

[dn]
C = ES
ST = Extremadura
L = Santibanez
CN = *.casa.local

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = *.casa.local
DNS.2 = casa.local
```

---

### **2.2 Generate the wildcard private key**
```bash
openssl genrsa -out casa_local_wildcard.key 2048
```

### **2.3 Generate the CSR**
```bash
openssl req -new   -key casa_local_wildcard.key   -out casa_local_wildcard.csr   -config casa-local-cert.cnf
```

### **2.4 Sign the wildcard certificate with your CA**
```bash
openssl x509 -req   -in casa_local_wildcard.csr   -CA casa_local_ca.crt   -CAkey casa_local_ca.key   -CAcreateserial   -out casa_local_wildcard.crt   -days 825   -sha256   -extensions req_ext   -extfile casa-local-cert.cnf
```

---

## 🟦 3. Create a Traefik‑friendly Certificate Chain

Combine certificate + CA:

```bash
cat casa_local_wildcard.crt casa_local_ca.crt > casa_local_wildcard_chain.crt
```

You now have:
- `casa_local_wildcard_chain.crt` → Certificate + CA
- `casa_local_wildcard.key` → Private key

---

## 🟦 4. Create the Kubernetes TLS Secret

For k3s built‑in Traefik (namespace: `kube-system`):

```bash
kubectl create secret tls casa-local-tls   --cert=casa_local_wildcard_chain.crt   --key=casa_local_wildcard.key   -n kube-system
```

If using a custom Helm-based Traefik, replace namespace with `traefik`.

---

## 🟦 5. Install CA Certificate on Devices

### **Android**
1. Copy `casa_local_ca.crt` to device.
2. Go to Settings → Security → Encryption & credentials.
3. Select **Install certificate** → **CA certificate**.
4. Choose the file.

### **Linux**
```bash
sudo cp casa_local_ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

### **Windows**
1. Right-click `casa_local_ca.crt`
2. *Install Certificate*
3. Choose **Local Machine**
4. Store: **Trusted Root Certification Authorities**

### **Firefox** (separate trust store)
Preferences → Privacy & Security → Certificates → Authorities → **Import**

---

## 🟦 6. Summary of All Commands

```bash
# Create CA
openssl genrsa -out casa_local_ca.key 4096
openssl req -x509 -new -nodes -key casa_local_ca.key -sha256 -days 3650 -out casa_local_ca.crt -subj "/C=ES/ST=Extremadura/L=Santibanez/CN=Casa Local Homelab CA"

# Create wildcard cert
openssl genrsa -out casa_local_wildcard.key 2048
openssl req -new -key casa_local_wildcard.key -out casa_local_wildcard.csr -config casa-local-cert.cnf
openssl x509 -req -in casa_local_wildcard.csr -CA casa_local_ca.crt -CAkey casa_local_ca.key -CAcreateserial -out casa_local_wildcard.crt -days 825 -sha256 -extensions req_ext -extfile casa-local-cert.cnf

# Create chain
cat casa_local_wildcard.crt casa_local_ca.crt > casa_local_wildcard_chain.crt

# Kubernetes TLS secret
kubectl create secret tls casa-local-tls --cert=casa_local_wildcard_chain.crt --key=casa_local_wildcard.key -n kube-system
```

---

If you'd like, I can also generate a **Traefik TLSStore + TLSOption config** that uses `casa-local-tls` as the *default certificate* for your entire cluster.
