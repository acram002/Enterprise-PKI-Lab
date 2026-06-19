# Enterprise PKI Lab

UPDATE: I have moved the web server from the AWS EC2 instance it was on to a self-hosted Ubuntu VM w/ routing through a Cloudflared tunnel (to not have to pay for it). However, the live demo is not functional now although you can visit the site at https://pki.pki-forge.com to see the updates to the project and the architecture before/after. Feel free to perform the lab as it is still a great way to learn PKI, AD CS, and web hosting!

Purpose: Hands-on lab for learning PKI fundamentals. Building a certificate authority hierarchy, publishing revocation data, and issuing a trusted TLS certificate end-to-end.

Overview: A two-tier Certificate Authority lab built on AWS EC2, using Windows Server 2022 and Active Directory Certificate Services (AD CS). Includes a public IIS web server hosting CRL and AIA distribution points, with a CA-signed TLS certificate.

---

## Live Demo

A live instance of this lab is currently running at **[pki.pki-forge.com](http://pki.pki-forge.com)**.

Follow the steps below to walk through the full certificate trust flow yourself:

1. **Observe the untrusted certificate**: navigate to [https://pki.pki-forge.com](https://pki.pki-forge.com). Your browser will show a certificate warning because the Root CA is not yet in your trust store.
2. **Download the CA certificate**: navigate to [http://pki.pki-forge.com](http://pki.pki-forge.com) and download the `.p7b` certificate file.
3. **Install and trust the certificate**: open the `.p7b` file and install it to the **Trusted Root Certification Authorities** store.
4. **Verify trust**: navigate back to [https://pki.pki-forge.com](https://pki.pki-forge.com). The site should now load with a valid, trusted certificate issued by the lab's own CA.

---

## Architecture

| VM | Role | Notes |
|----|------|-------|
| VM0 *(optional)* | Domain Controller (AD DS) | Enables domain join + GPO cert management. Elastic IP. |
| VM1 | Root CA | No public IP. Offline except during cert signing. |
| VM2 | Issuing CA | Outbound traffic + RDP. Domain-join optional. |
| VM3 | Web Server (IIS) | Elastic IP. Ports 80, 443, 3389. Hosts PKI site + CRL/AIA. |
| S3 Bucket | File Transfer | Staging for cert/CRL transfers between instances. |

All instances: Windows Server 2022, t3.medium (4 GB RAM), 50 GB gp3.

> **Note:** This lab can also run on-prem with Proxmox on a 16–32 GB RAM machine. Windows evaluation ISOs can be used for licensing.

---

## Prerequisites

- AWS account with EC2 and S3 access
- Registered domain name (required for public HTTPS demo)
- AWS CLI installed on each instance (`aws configure` with appropriate IAM permissions)
- Basic familiarity with Windows Server and IIS

---

## Setup Overview

### 1. Provision Infrastructure

1. Launch 4 x Windows Server 2022 EC2 instances per the architecture table above.
2. Assign Elastic IPs to VM0 (if used) and VM3.
3. Configure Security Groups:
   - VM1: RDP inbound only, no outbound
   - VM3: TCP 80, 443, 3389 inbound
4. Create an S3 bucket (e.g., `pki-lab-transfer`) for file staging. Block public access.
5. Create a DNS **A record** pointing `pki.yourdomain.com` to VM3's Elastic IP.

---

### 2. Configure the Web Server (VM3)

Create the IIS directory structure:

```
C:\inetpub\pki\
    ├── aia\
    ├── crl\
    ├── indexHTTP\
    └── indexHTTPS\
```

In IIS Manager:
- Create a new site mapped to `C:\inetpub\pki\`
- Enable **Directory Browsing**
- Add virtual directories for `/aia`, `/crl`, `/indexHTTP`, `/indexHTTPS`
- Add an HTTP binding on port 80 for your domain

Verify by navigating to `http://pki.yourdomain.com` — you should see the directory listing.

---

### 3. Build the Root CA (VM1)

1. Install **AD CS** role → Standalone CA → Root CA.
2. Key settings: RSA 4096-bit, SHA-256, 10-year validity.
3. Set AIA and CDP extensions:
   ```
   AIA: http://pki.yourdomain.com/aia/<CaName>.crt
   CDP: http://pki.yourdomain.com/crl/<CaName>.crl
   ```
4. Export the Root CA certificate (`.cer`).
5. Publish the CRL and export the `.crl` file from `C:\Windows\System32\CertSrv\CertEnroll\`.
6. Transfer both files to VM3 via S3 → copy to `aia\` and `crl\` directories.
7. **Shut down VM1.** Only power it on to sign subordinate CA certificates.

---

### 4. Build the Issuing CA (VM2)

1. Install **AD CS** role → Standalone (or Enterprise if domain-joined) → Subordinate CA.
2. Save the **Certificate Signing Request (CSR)** to disk.
3. Copy the CSR to VM1 via S3. Start VM1.
4. On VM1: submit and issue the CSR, export the signed cert as `.p7b`.
5. Shut down VM1.
6. On VM2: install the signed certificate and import the Root CA `.cer` into **Trusted Root Certification Authorities**.
7. Configure AIA and CDP extensions (same URLs as Root CA).
8. Publish the Issuing CA CRL and transfer it to VM3.

---

### 5. Issue a TLS Certificate for the Web Server (VM3)

1. In `certmgr.msc` → **Advanced Operations → Create Custom Request** (no template, PKCS#10).
2. Set Subject: `CN=pki.yourdomain.com`, SAN: `DNS=pki.yourdomain.com`, RSA 2048.
3. Save the CSR and copy it to VM2 via S3.
4. On VM2, submit and issue the certificate, export as `.p7b`.
5. Copy `.p7b` back to VM3. Import into the Personal certificate store.
6. In IIS, add an **HTTPS binding** on port 443 using the new certificate.
7. Place the Root CA `.cer` in `C:\inetpub\pki\indexHTTP\` for manual client trust installation.

> **Enterprise CA only:** You'll need to create a certificate template that includes the SAN extension before submitting the CSR. Duplicate the Web Server template in `certtmpl.msc` and publish it to the CA.

---

## Verification

1. **Before trusting the Root CA**: browse to `https://pki.yourdomain.com`. Expect a certificate error.
2. **Download the Root CA cert** from `http://pki.yourdomain.com/indexHTTP/`.
3. **Verify the thumbprint** before installing:
   ```
   certutil -hashfile <RootCA.cer> SHA256
   ```
   Compare against the thumbprint on VM1.
4. Install the Root CA cert to **Trusted Root Certification Authorities**.
5. Browse again to `https://pki.yourdomain.com`: the padlock should now be green with a valid chain.

---

## File Reference

| File | Location on VM3 |
|------|----------------|
| Root CA cert | `C:\inetpub\pki\aia\` |
| Issuing CA cert | `C:\inetpub\pki\aia\` |
| Root CA CRL | `C:\inetpub\pki\crl\` |
| Issuing CA CRL | `C:\inetpub\pki\crl\` |
| Root CA cert (download) | `C:\inetpub\pki\indexHTTP\` |

**Useful commands:**

```
certutil -CRL                          # Publish CRL on Issuing CA
certreq -submit <file.req>             # Submit CSR to CA
certutil -hashfile <file.cer> SHA256   # Verify certificate thumbprint
```

---

## Notes & Caveats

- **S3 for file transfer is not a production pattern.** Root CA keys should be stored offline with HSM backing.
- **Server hardening is not covered** but is recommended for any internet-facing instance.
- CRLs expire on a schedule — automate republishing with a Scheduled Task (`certutil -CRL`) running on an interval shorter than the CRL validity period.

---

## Potential Extensions

- Automate CRL sync from VM2 to S3/VM3 via AWS Lambda or Scheduled Task
- Deploy Root CA on Linux with OpenSSL for a cross-platform setup
- Add OCSP responder as an alternative to CRL
- Use GPO to auto-distribute the Root CA cert to domain-joined machines
- Explore autoenrollment with Enterprise CA and certificate templates

## Further Reading

I created this lab/project by pulling from numerous articles, documentation, and videos. Here's a list of the sources I used.


Solid PKI, Two-tier CA overview. Includes design tradeoffs (i.e. offline root on dedicated server h/w or portable h/w) 

https://www.entrust.com/sites/default/files/documentation/whitepapers/offline-ca-best-practices-wp.pdf 

 

Process for setting up offline root and AD CS issuing CA w/ AIA/CRL points. Includes HSM integration 

https://docs.securosys.com/ms-pki-adcs/category/tutorials/ 

 

Additional reading on setting up two-tier PKI w/ AD CS 

https://techcommunity.microsoft.com/blog/microsoft-security-blog/step-by-step-2-tier-pki-lab/4413982 


Further reading on two-tier PKI w/ AD CS. Helpful info on setting up HTTP web server (IIS) 
https://www.windows-noob.com/forums/topic/16252-how-can-i-configure-pki-in-a-lab-on-windows-server-2016-part-1/


Youtube video for cert template error 
https://www.youtube.com/watch?v=fmpozm3j2bM

 
Same as above, manual csr request for web server 

https://www.gradenegger.eu/en/manual-application-for-a-web-server-certificate/ 

 

Same as above, bypassing cert template error to issue certs to non-domain servers 

https://www.reddit.com/r/sysadmin/comments/wws636/certificate_for_nondomain_server_using_microsoft/ 

 

Further reading on manual custom request for web server to avoid web browser trust issues 

https://techcommunity.microsoft.com/discussions/edgeinsiderdiscussions/how-do-i-get-edge-to-trust-our-internal-certificate-authority/785333/replies/827884 
