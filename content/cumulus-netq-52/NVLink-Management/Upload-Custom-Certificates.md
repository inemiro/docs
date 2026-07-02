---
title: Upload Custom Certificates
author: NVIDIA
weight: 260
toc: 4
---

NetQ NVLink supports two types of certificates: self-signed or custom. Self-signed certificates are auto-generated during the NetQ NVLink installation process and require no further action. If you specified a custom certificate in the JSON template during the initial installation, follow the steps on this page to upload the certificates. 

## Prerequisites

- The JSON configuration file used to {{<link title="Install NetQ NVLink" text="install NetQ NVLink">}} must have the `cert-mode` attribute set to `user-cert`.
- You must have the following certificates. Make sure the certificates are valid and not expired.
    - A CA certificate (PEM-encoded) from your certificate authority.
    - A server TLS certificate and its corresponding private key (both PEM-encoded), signed by the same CA.
    - A PKCS#12 (.p12) certificate bundle for your switches, signed by the same CA. The .p12 file must not be password-protected.


## Upload the Certificates using the API

1. Upload your CA public certificate as a PEM-encoded file by making a POST request to the `/v1/certificates/ca` endpoint:

```
curl -X 'POST' \
  'https://<ip-address>/nmx/v1/certificates/ca' \
  -H 'accept: application/json' \
  -H 'Content-Type: multipart/form-data' \
  -F 'certificate=@<path-to-ca-cert.pem>'
```

A successful upload returns HTTP 200 OK with the CA certificate metadata:

```
{
  "type": "ca",
  "subject": "CN=My Organization CA",
  "issuer": "CN=My Organization CA",
  "serialNumber": "1a2b3c",
  "notBefore": "2025-01-01T00:00:00Z",
  "notAfter": "2030-01-01T00:00:00Z",
  "fingerprint": "sha256-hex-string",
  "keyAlgorithm": "RSA-4096"
}
```

2. Upload the server (southbound) TLS certificate and its private key (both as PEM-encoded files) by making a POST request to the `/v1/certificates/server` endpoint. The certificate must be signed by the same CA as the certificate you uploaded in the previous step.

```
curl -X 'POST' \
  'https://<ip-address>/nmx/v1/certificates/server' \
  -H 'accept: application/json' \
  -H 'Content-Type: multipart/form-data' \
  -F 'certificate=@<path-to-server-cert.pem>' \
  -F 'privateKey=@<path-to-server-key.pem>'
```

If the operation was successful, the API returns an operation ID which you can use to track the status of the upload. If the initial upload fails at any point, you can retry with the `force` query parameter. Setting `force=true` cleans up the metadata from the previous attempt and proceeds with a fresh upload:

```
curl -X 'POST' \
  'https://<ip-address>/nmx/v1/certificates/server?force=true' \
  -H 'accept: application/json' \
  -H 'Content-Type: multipart/form-data' \
  -F 'certificate=@<path-to-server-cert.pem>' \
  -F 'privateKey=@<path-to-server-key.pem>
```

Note that you cannot use the `force` parameter to replace a server certificate that has already been successfully uploaded. You must perform a fresh NetQ NVLink installation to replace a certificate.

3. After the certificates are uploaded successfully, {{<link title="NVLink Bringup/#bringup-examples-using-custom-certificates" text="perform a bringup">}}.

# Rotate Customer Certificates

You can rotate certificates (replace) them without reinstalling NetQ NVLink. Rotation allows replacing the CA, the server certificate, or the switch P12 certificates in place. Rotate certificates before they expire, or when a certificate or key is compromised.

## Prerequisites

- The CA and server certificates must already be uploaded. You cannot rotate a certificate that was never set. See {{<link title="#Upload the Certificates using the API" text="Upload the Certificates using the API">}} section.
- You need valid, unexpired replacement certificates:
  - A PEM-encoded CA certificate from your certificate authority.
  - A PEM-encoded server TLS certificate and its private key, signed by the same CA.
  - A PKCS#12 (`.p12`) bundle for your switches, signed by the same CA. It must not be password-protected.
- Rotations requires {{<link title="Maintenance mode" text="maintenance mode">}} to be enabled at the time you submit the request.

There is no required order between the three certificates. If you replace the CA, sign the new server and switch certificates with the new CA.

## Rotate the CA certificate

Rotation applies synchronously. Send a PUT request to `/v1/certificates/ca` with the new CA certificate:

```
curl -X 'PUT' \
  'https://<ip-address>/nmx/v1/certificates/ca' \
  -H 'accept: application/json' \
  -H 'Content-Type: multipart/form-data' \
  -F 'certificate=@<path-to-ca-cert.pem>'
```

A successful rotation returns `HTTP 200 OK` with the new certificate metadata.

## Rotate the server certificate

Send a PUT request to `/v1/certificates/server` with the new certificate and private key. The certificate must be signed by the current CA:

```
curl -X 'PUT' \
  'https://<ip-address>/nmx/v1/certificates/server' \
  -H 'accept: application/json' \
  -H 'Content-Type: multipart/form-data' \
  -F 'certificate=@<path-to-server-cert.pem>' \
  -F 'privateKey=@<path-to-server-key.pem>'
```

The request returns `HTTP 202 Accepted` with an operation ID for tracking status.

By default, the new certificate is staged and services pick it up on their next reconnection. To activate it immediately and force managed services to reconnect, add `Reconnect=true`:

```
curl -X 'PUT' \
  'https://<ip-address>/nmx/v1/certificates/server?Reconnect=true' \
  -H 'accept: application/json' \
  -H 'Content-Type: multipart/form-data' \
  -F 'certificate=@<path-to-server-cert.pem>' \
  -F 'privateKey=@<path-to-server-key.pem>'
```

## Rotate switch certificates

Send a PUT request to `/v1/certificates/switches` with the new `.p12` bundle and one or more switches. The bundle must be signed by the current CA:

```
curl -X 'PUT' \
  'https://<ip-address>/nmx/v1/certificates/switches' \
  -H 'accept: application/json' \
  -H 'Content-Type: multipart/form-data' \
  -F 'CertP12=@<path-to-switch-bundle.p12>' \
  -F 'Switches={"Address":"10.1.1.1"}' \
  -F 'Switches={"Address":"10.1.1.2"}'
```

Each `Switches` entry takes an `Address` (the switch management IP or hostname, which must match a managed switch). To apply a switch profile, add a global `ProfileID` field, or set `ProfileID` per switch to override it:

```
  -F 'ProfileID=551137c2f9e1fac808a5f572' \
  -F 'Switches={"Address":"10.1.1.1"}' \
  -F 'Switches={"Address":"10.1.1.2","ProfileID":"661248d3a0f2gbd919b6g683"}'
```

The request returns `HTTP 202 Accepted` with an operation ID.

To apply a different `.p12` bundle to different switches, submit separate requests.

In case rotation failed you can try again for the same switch.

## Track progress

The server and switch rotations run asynchronously. Poll `GET /v1/operations/{id}` with the returned operation ID to check progress:

```
curl -X 'GET' \
  'https://<ip-address>/nmx/v1/operations/551137c2f9e1fac808a5f572' \
  -H 'accept: application/json'
```
