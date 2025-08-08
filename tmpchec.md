## 1. Name of the decision

Technology/Tools to be used for implementing container signing, which needs to be implemented by CFW-Infra team.

## 1.1 Affected Products

- CFW-onboard
- CFW-offboard
- CFW-frame
- CFW-infra

## 2. Problem Description

Container Image signing for security and authenticity, so that the end user knows it is using the actual image **what it says.**

 - Solution must ensure container images used are signed by the creator for integrity and trustworthiness of container images.
 - Solution must ensure the images should be signed and verified using self-signed or trusted CA certificates.
 - Solution must provide verification of the signatures of images while deployment in AKS.

**Goal:**

- Implement container image signing for all the acr images and verification at the deployment stage.

## 3. Considerations

### 3.1 Assumptions

- Secure Key Management
- Sign Images Based on Immutable Identifiers
- Automate Signing in CI/CD Pipelines
- Implement Signature Verification Policies in AKS

### 3.2 Technical Constraints

- Secure Private key storage
- Immutable Identifiers for Signing

## 4. Evaluated Approaches

### 4.1 Option 1 :  *Notation CLI + Self-signed/Trusted CA Cert*

Image signing via Notation cli with Certs ([Reference link](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-tutorial-sign-build-push))

Notation CLI is a command-line tool from the Notary Project designed specifically for signing and verifying container images and other OCI (Open Container Initiative) artifacts.

Key Features:

- **Cryptographic Signing and Verification:** Allows users to apply digital signature to container images using industry standard algorithms.
- **Support for Multiple Key Management Services (KMS):** Integrates with cloud KMS solutions (such as Azure Key Vault, AWS Signer, and others) using plugins.
- **Flexible Credential and Plugin Support:** Auhtenticates to remote KV using various methods, including environment credentials, managed identities, and azurecli creds.
  
#### 4.1.1 Installation and Setup

**DigiCert for Signing:**

- Recommendation by Security Team

  1. Let’s Encrypt (DV only))
  2. Amazon Web Services (AWS) Certificate Manager (ACM) (not ACM Private CA) (DV only))
  3. Microsoft Azure integrated App Service Certificates provided by Starfield Technologies, LLC on top of GoDaddy (DV only))
  4. Google Trust Services (DV only)
  5. DigiCert (DV, OV, EV)

Domain Validated (DV): This is the lowest level of validation. The CA only verifies that the requestor has control over the domain name. This is usually done through email verification, DNS records, or file upload. DV certificates are quick and cheap to obtain, but they don't provide any organizational identity.

- Organization Validation(OV) and Extended Validation(EV) Offered by DigiCert is a good option for us as it verifies organization identity.

- DigiCert-OV (Code Signing cert):  
    Code Signing - HSM  
    Use your own hardware security module (HSM)  
    $600.00/year subscription

- DigiCert-EV (Code Signing cert)    
    Code Signing - HSM  
    Use your own hardware security module (HSM)  
    $840.00/year subscription

**Self-Signed Cert Creation:**

Creating self-signed cert for signing images

- Generating Private key RSA type:
  - openssl genrsa -out cert-key.pem 2048
- Generating Certificate singing request(CSR) for Certificate creation:
  - openssl req -new -key cert-key.pem  -out image-sign.csr -subj "/C=IND/ST=BLR/O=Bosch/CN=*.mobility-commonframeworks.bosch.tech"
- Generating Certificate:
    ```yaml
    cat >> openssl_config.cnf <<EOF
    [ v3_req ]
    basicConstraints = critical,CA:FALSE
    keyUsage = critical,digitalSignature
    extendedKeyUsage = codeSigning
    authorityKeyIdentifier = keyid:always
    EOF
    ```
  - openssl x509 -req -days 365 -in image-sign.csr -signkey cert-key.pem -extensions v3_req -extfile openssl_config.cnf -out cert.pem
  - Add the key and certificate to Azure Key-vault for security and ease of using (Add priavte key and cert in a single pem file and upload it to AKV).

**Notation CLI:**

- Download, extract and install Notation CLI
  ```yaml
  curl -Lo notation.tar.gz https://github.com/notaryproject/notation/releases/download/v1.3.2/notation_1.3.2_linux_amd64.tar.gz
  tar xvzf notation.tar.gz
  # Copy the Notation binary to the desired bin directory in your $PATH, for example
  cp ./notation /usr/local/bin
  ```
- Install & verify the Notation Azure Key Vault plugin on linux
  ```yaml
  notation plugin install --url https://github.com/Azure/notation-azure-kv/releases/download/v1.2.1/notation-azure-kv_1.2.1_linux_amd64.tar.gz --sha256sum 67c5ccaaf28dd44d2b6572684d84e344a02c2258af1d65ead3910b3156d3eaf5
  notation plugin ls
  ```
 - Secure access permissions to ACR and AKV
   - ACR user should have AcrPull and AcrPush roles enabled.
   - For AKV, following permissions are required for an identity:
  
      `Create` permissions for creating a certificate

      `Get` permissions for reading existing certificates

      `Sign` permissions for signing operations

#### 4.1.2 Signing Image using Notation CLI

Sign using Self-Signed/Trusted CA Cert

1. Authenticate to ACR
    ``` 
    az acr login --name <ACR_NAME> 
    ```
2. Image to sign (It is better to use the digest value to identify the image for signing since tags are mutable and can be overwritten).
   ```
   IMAGE=<REGISTRY-NAME>/<REPO-NAME>:<TAG>
   IMAGE=<REGISTRY-NAME>/<REPO-NAME>@<DIGEST>
   ```
3. Get the Key ID of the signing key (we added the cert in AKV with pvt key).
    ```
    KEY_ID=$(az keyvault certificate show -n <CERT_NAME> --vault-name <AKV_NAME> --query 'kid' -o tsv)
    ```
4. For Self-Signed
   
   Sign the container image with the COSE signature format using the signing key ID. To sign with a self-signed certificate, you need to set the plugin configuration value
   self_signed=true.
   ```
   notation sign --signature-format cose --id $KEY_ID --plugin azure-kv --plugin-config self_signed=true $IMAGE
   ```
   For Trusted CA:

   When creating certificates for signing and verification, the certificates must meet the ([Notary Project certificate requirement](https://github.com/notaryproject/specifications/blob/v1.0.0/specs/signature-specification.md#certificate-requirements))
   
   If the certificate contains the entire certificate chain, run the following command:
   ```
   notation sign --signature-format cose $IMAGE --id $KEY_ID --plugin azure-kv
   ```
   If the certificate does not contain the chain, use the --plugin-config ca_certs=<ca_bundle_file> parameter to pass the CA certificates in a PEM file to AKV plugin, run the following command:  
   ```
   notation sign --signature-format cose $IMAGE --id $KEY_ID --plugin azure-kv --plugin-config ca_certs=<ca_bundle_file>
   ```

#### 4.1.3 Verify a container image with Notation CLI

1. Download public certificate.
   ```
   az keyvault certificate download --name <CERT_NAME> --vault-name <AKV_NAME> --file sign-cert.pem/<cert-path>
   ```
2. Add the downloaded public certificate to named trust store for signature verification.
   ```
   STORE_TYPE="ca"
   STORE_NAME="xyz"
   notation cert add --type $STORE_TYPE --store $STORE_NAME $CERT_PATH
   ``` 
3. Configure trust policy before verification:

    Trust policies allow users to specify fine-tuned verification policies. 
    ```json
    cat <<EOF > ./trustpolicy.json
    {
        "version": "1.0",
        "trustPolicies": [
            {
                "name": "<policy-name>",
                "registryScopes": [ "<REGISTRY-NAME>/<REPO-NAME>" ],
                "signatureVerification": {
                    "level" : "strict" 
                },
                "trustStores": [ "$STORE_TYPE:$STORE_NAME" ],
                "trustedIdentities": [
                    "x509.subject: <subject-of-the-cert>/"
                ]
            }
        ]
    }
    EOF
    ```
4. Use notation policy to import the trust policy configuration from a JSON file that we created previously.
   ```
   notation policy import ./trustpolicy.json
   notation policy show
   ```
5. Use notation verify to verify the container image hasn't been altered since build time.
   '''
   notation verify $IMAGE
   '''

#### 4.1.4 Pros and Cons

**Pros**

- Standardized & secure cryptographic technique
- Easy integration for container registries(docker hub, ACR, AWS ECR)
- Flexibility in certificate management(AWS KMS, Azure Kv)
-  Azure supported Plugins for verification in AKS while deployment

**Cons**

- Learning curve, requires understanding of cryptographic keys, certs and OCi artifacts
- Feature maturity, as it is new tool and still evolving so no all features of integrations are fully stable
  


**Risks**
- No warning on docker pull if root certs are not present
