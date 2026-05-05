---
title: "Build a mobile driver’s license solution based on ISO/IEC 18013-5 using AWS Private CA and AWS KMS"
url: "https://aws.amazon.com/blogs/security/build-a-mobile-drivers-license-solution-based-on-iso-iec-18013-5-using-aws-private-ca-and-aws-kms/"
date: "Wed, 04 Sep 2024 17:33:17 +0000"
author: "Ram Ramani"
feed_url: "https://aws.amazon.com/blogs/security/category/security-identity-compliance/aws-key-management-service/feed/"
---
<p>A mobile driver’s license (mDL) is a digital representation of a physical driver’s license that’s stored on a mobile device.&nbsp;An mDL is a significant improvement over physical credentials, which can be lost, stolen, counterfeited, damaged, or contain outdated information, and can expose unconsented personally identifiable information (<a href="https://csrc.nist.gov/glossary/term/PII" rel="noopener" target="_blank">PII</a>). Organizations are working together to use mDLs across various situations, ranging from validating identity during airplane boarding to sharing information for age-restricted activities.</p> 
<p>The trust in the mDL system is based on public-private key cryptography where mDLs are signed by issuing authorities using their private key and verified using the issuing authority’s public key. In this blog post, we show you how to build an mDL issuing authority in <a href="https://aws.amazon.com" rel="noopener" target="_blank">Amazon Web Services (AWS)</a> using <a href="https://aws.amazon.com/private-ca/" rel="noopener" target="_blank">AWS Private Certificate Authority</a> and <a href="https://aws.amazon.com/kms/" rel="noopener" target="_blank">AWS Key Management Service (AWS KMS)</a> according to mDL specification <a href="https://www.iso.org/standard/69084.html" rel="noopener" target="_blank">ISO/IEC 18013-5:2021</a>. These AWS services align with the cryptographic requirements placed on the issuing authorities by ISO/IEC 18013-5. While we have tailored this post to an mDL use case, the sign and verify mechanism using AWS Private CA and AWS KMS can be used for multiple kinds of digital identity verification.</p> 
<h2>Solution overview</h2> 
<p>AWS Private CA provides you with a highly available private certificate authority (CA) service without the initial investment and ongoing maintenance costs of operating your own private CA. CA administrators can use AWS Private CA to create a complete CA hierarchy, including online root and subordinate CAs, without needing external CAs. You can issue, rotate, and revoke certificates that are trusted within your organization using AWS Private CA.</p> 
<p>AWS Private CA can issue <a href="https://aws.amazon.com/about-aws/whats-new/2024/01/aws-private-ca-mobile-drivers-license-certificates/" rel="noopener" target="_blank">certificates formatted as required by ISO/IEC 18013-5</a>. You can build a certificate authority (CA) in AWS Private CA—referred to as the issuing authority certificate authority (IACA) in ISO/IEC 18013-5. We create an IACA self-signed root certificate and an mDL document signing certificate in AWS Private CA.</p> 
<p>AWS KMS is a managed service that you can use to create and control the cryptographic keys that are used to protect your data. AWS KMS uses FIPS 140-2 Level 3 validated hardware security modules (HSMs) to protect AWS KMS keys, which is a requirement for building an issuing authority as described in ISO/IEC 18013-5. We create an asymmetric key pair in AWS KMS for signing and verification of the mDL document. We programmatically create a certificate signing request (CSR) that’s signed by the asymmetric key pair stored in AWS KMS. The CSR is sent to the AWS Private CA service for issuing the mDL document signing certificate that matches the certificate profile requirement specified for the document signing certificate in ISO/IEC 18013-5.</p> 
<p>We sign an mDL document using the private key of the asymmetric key pair created in AWS KMS with a <a href="https://docs.aws.amazon.com/kms/latest/developerguide/concepts.html#key-usage" rel="noopener" target="_blank">KeyUsage </a>value of <code style="color: #000000;">SIGN_VERIFY</code>. The signed mDL document is delivered to a mobile device where it’s stored in a digital wallet and produced for verification by mDL readers. The mDL readers are configured with IACA certificates from various issuing authorities that allow them to verify the mDL documents signed by respective issuing authorities. An example of an issuing authority could be a state government agency that issues driver’s licenses.</p> 
<h2>Least privilege</h2> 
<p>The solution in this post uses AWS KMS and AWS Private CA services. Before you implement the process described in this post, ensure that the <a href="https://aws.amazon.com/iam" rel="noopener" target="_blank">AWS Identity and Access Management (IAM)</a> principal you choose follows the principle of least privilege and that permissions are scoped to the minimum required permissions required. See <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html" rel="noopener" target="_blank">Security best practices in IAM</a> to learn more.</p> 
<h2>Solution architecture</h2> 
<p>A sample solution architecture for building an mDL issuing authority in AWS is shown in Figure 1. The figure shows the step-by-step process starting from setting up a private CA and issuing an mDL document signing certificate to mDL issuance and verification. The infrastructure that’s built using this architecture includes a root certificate authority, which issues a document signer certificate. You can find the certificate requirements in section B.1 Certificate Profile of ISO/IEC 18013-5.</p> 
<div class="wp-caption aligncenter" id="attachment_35603" style="width: 790px;">
 <a href="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/08/30/img1-7.png" rel="noopener" target="_blank"><img alt="Figure 1: mDL issuing authority architecture and process flow in AWS" class="size-full wp-image-35603" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/08/30/img1-7.png" style="border: 1px solid #bebebe;" width="780" /></a>
 <p class="wp-caption-text" id="caption-attachment-35603">Figure 1: mDL issuing authority architecture and process flow in AWS</p>
</div> 
<p>In this post, we use <a href="https://aws.amazon.com/cli/" rel="noopener" target="_blank">AWS Command Line Interface (AWS CLI)</a> commands, but these can be replaced by AWS SDK API calls if needed. Along with the AWS CLI steps, a <a href="https://github.com/aws-samples/csr-signing-using-kms" rel="noopener" target="_blank">GitHub sample</a> is provided that’s used to programmatically create and sign an mDL document signing CSR using AWS KMS.</p> 
<p>See the AWS CLI commands documentation for <a href="https://docs.aws.amazon.com/cli/latest/reference/acm-pca/" rel="noopener" target="_blank">AWS Private CA</a> and <a href="https://docs.aws.amazon.com/cli/latest/reference/kms/" rel="noopener" target="_blank">AWS KMS</a> for detailed information on the commands used in this solution.</p> 
<h2>Solution walkthrough</h2> 
<p>Use the following steps to create the infrastructure needed for mDL signing and verification.</p> 
<h3>Step 1: Create IACA CA in AWS Private CA</h3> 
<p>In this step, the root of trust IACA (issuing authority CA) will be created. The IACA root CA is the root of trust that will be used for verification of the mDL.</p> 
<ol> 
 <li>Create a local <code style="color: #000000;">ca_config.txt</code> file with the following content. The contents of this file are derived from the Certificate profiles section (Annex B) within ISO/IEC 18013-5. You can change the <code style="color: #000000;">Country</code> and <code style="color: #000000;">CommonName</code> values in the file as needed for your requirements. 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">{
  "KeyAlgorithm": "EC_prime256v1",
  "SigningAlgorithm": "SHA256WITHECDSA",
  "Subject": {
    "Country": "US",
    "CommonName": "mDL IACA Root"
  }
}</code></pre> 
  </div> </li> 
 <li>The IACA root certificate will be paired with a certificate revocation list (CRL). See <a href="https://docs.aws.amazon.com/privateca/latest/userguide/crl-planning.html" rel="noopener" target="_blank">Planning a certificate revocation list (CRL)</a> for information about configuring CRLs. Create a local file called <code style="color: #000000;">revocation_config.txt</code> with the following information to configure a CRL. The values for <code style="color: #000000;">CustomCname</code> and <code style="color: #000000;">S3BucketName</code> are examples, update them with the values that you have created within your AWS account. Update <code style="color: #000000;">ExpirationInDays</code> to fit your requirements. We recommend configuring encryption on the <a href="https://aws.amazon.com/s3" rel="noopener" target="_blank">Amazon Simple Storage Service (Amazon S3)</a> bucket containing your CRLs. 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">{
  "CrlConfiguration": {
    "CustomCname": "example.com",
    "Enabled": true,
    "S3BucketName": "crlmdlbucket",
    "ExpirationInDays": 5000,  
  }
}</code></pre> 
  </div> </li> 
 <li>Invoke an AWS CLI command to <a href="https://awscli.amazonaws.com/v2/documentation/api/latest/reference/acm-pca/create-certificate-authority.html" rel="noopener" target="_blank">create a private certificate authority</a>. Replace the <code style="color: #000000;">region</code> parameter as needed. Update the <code style="color: #000000;">file://</code> paths in the following command to the locations where you’ve stored the <code style="color: #000000;">ca_config.txt</code> and <code style="color: #000000;">revocation_config.txt</code> files. 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">aws acm-pca create-certificate-authority \ 
    --region us-west-1 \
    --certificate-authority-configuration file://ca_config.txt \
    --revocation-configuration file://revocation_config.txt \
    —-certificate-authority-type "ROOT"</code></pre> 
  </div> </li> 
 <li>The command should produce the following output. The output contains the <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference-arns.html" rel="noopener" target="_blank">Amazon Resource Name</a> (ARN) of the created CA. You will need this ARN in subsequent steps. 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">{
    "CertificateAuthorityArn": "arn:aws:acm-pca:us-west-1:123412345678:certificate-authority/0116z123-dv7a-59b1-x7be-1231v7257113"
}</code></pre> 
  </div> </li> 
</ol> 
<h3>Step 2: Retrieve the CSR for IACA root certificate</h3> 
<p>You’ll create an IACA root certificate, which starts with retrieving a CSR. This step retrieves the CSR for the IACA root certificate. The <code style="color: #000000;">certificate-authority-arn</code> parameter carries the CA ARN that was generated in Step 1.</p> 
<ol> 
 <li>The following command will output a <a href="https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail" rel="noopener" target="_blank">Privacy-Enhanced Mail</a> (PEM) formatted CSR. 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">aws acm-pca get-certificate-authority-csr \
    --region us-west-1 \
    --output text \
    --certificate-authority-arn arn:aws:acm-pca:us-west-1:123412345678:certificate-authority/0116z123-dv7a-59b1-x7be-1231v7257113</code></pre> 
  </div> </li> 
 <li>The following is the format of the output CSR: 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">-----BEGIN CERTIFICATE REQUEST-----
..
-----END CERTIFICATE REQUEST-----</code></pre> 
  </div> </li> 
 <li>Store the output text in a file called <code style="color: #000000;">IACA.csr</code>.</li> 
</ol> 
<h3>Step 3: Generate root certificate</h3> 
<ol> 
 <li>This step issues the IACA root certificate. Create a file named <code style="color: #000000;">extensions.txt</code> using the following contents, which are derived from the Certificate profiles section of ISO/IEC 18013-5. <p>The <code style="color: #000000;">KeyUsage</code> extension with <code style="color: #000000;">KeyCertSign</code> and <code style="color: #000000;">CRLSign</code> should be set to <code style="color: #000000;">true</code>. A custom extension for the CRL distribution point is set and the validity of the certificate should be set to 9 years or 3285 days (set in the next step). Because the IACA root certificate is only used to issued mDLs, a maximum validity period of 9 years is sufficient, as indicated in Table B.1 of ISO/IEC 18013-5. Additionally, a CRL distribution point extension must be present. In the following example, the CRL URL encoded in the CDP extension is <code style="color: #000000;">http://example.com/crl/0116z123-dv7a-59b1-x7be-1231v72571136.crl</code>, aligning with both the CA CRL configuration applied to the CA at creation and to the CA ID. For base-64 encoding of the CDP extension, you can refer to this <a href="https://github.com/aws-samples/csr-signing-using-kms/blob/main/src/main/java/com/amazonaws/kmscsr/examples/GenerateCDPSample.java" rel="noopener" target="_blank">java sample</a>.</p> 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">{
  "Extensions": {
    "KeyUsage": {
      "KeyCertSign": true,
      "CRLSign": true
    },
    "CustomExtensions": [
      {
        "ObjectIdentifier": "2.5.29.31",
        "Value": "MEgwRqBEoEKGQGh0dHA6Ly9leGFtcGxlLmNvbS9jcmwvMDExNnoxMjMtZHY3YS01OWIxLXg3YmUtMTIzMXY3MjU3MTEzNi5jcmw="
       }
    ]
  }
}</code></pre> 
  </div> </li> 
 <li>Issue the following command to AWS Private CA to create the certificate. 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">aws acm-pca issue-certificate \
    --region us-west-1 \
    --certificate-authority-arn arn:aws:acm-pca:us-west-1:123412345678:certificate-authority/0116z123-dv7a-59b1-x7be-1231v7257113 \
    --template-arn "arn:aws:acm-pca:::template/BlankRootCACertificate_PathLen0_APIPassthrough/V1" \
    --signing-algorithm "SHA256WITHECDSA" \
    --csr fileb://IACA.csr \
    --validity Value=3285,Type="DAYS" \
    --api-passthrough file://extensions.txt</code></pre> 
  </div> </li> 
 <li>The preceding command will produce the following output: 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">{
  "CertificateArn": "arn:aws:acm-pca:us-west-1:123412345678:certificate-authority/0116z123-dv7a-59b1-x7be-1231v7257113/certificate/34a1dab03117f0e89c54b1234fe13318"
}</code></pre> 
  </div> </li> 
</ol> 
<p>Note that the IACA root CA created with AWS Private CA currently doesn’t have a CRL distribution point (CDP) extension by default. However, that is a mandatory extension according to the IACA root certificate profile in ISO/IEC 18013-5. To implement this, we use a custom extension passed in using <a href="https://docs.aws.amazon.com/privateca/latest/APIReference/API_ApiPassthrough.html" rel="noopener" target="_blank">API passthrough,</a> which embeds the CDP extension. The distribution point specified in that extension must be based on the CA ID, which is <code style="color: #000000;">0116z123-dv7a-59b1-x7be-1231v7257113</code> derived from the CA ARN that was created in Step 1.</p> 
<h3>Step 4: Retrieve root certificate </h3> 
<p>This step retrieves the IACA root certificate in PEM format.</p> 
<ol> 
 <li>Use the following code to retrieve the IACA root certificate: 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">aws acm-pca get-certificate \
    --region us-west-1 \
    --certificate-authority-arn arn:aws:acm-pca:us-west-1:123412345678:certificate-authority/0116z123-dv7a-59b1-x7be-1231v7257113 \
    --certificate-arn arn:aws:acm-pca:us-west-1:123412345678:certificate-authority/0116z123-dv7a-59b1-x7be-1231v7257113/certificate/34a1dab03117f0e89c54b1234fe13318 \
    --output text</code></pre> 
  </div> </li> 
 <li>The command output will be a PEM formatted certificate similar to the following: 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">-----BEGIN CERTIFICATE-----
..
-----END CERTIFICATE-----</code></pre> 
  </div> </li> 
 <li>Store the output text in a file named <code style="color: #000000;">IACA-Root-CA-Cert.pem</code>.</li> 
</ol> 
<h3>Step 5: Import root certificate</h3> 
<p>Use the following code to import the root certificate into AWS Private CA and make the certificate authority active and ready to issue certificates.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">aws acm-pca import-certificate-authority-certificate \
    --region us-west-1 \
    --certificate-authority-arn arn:aws:acm-pca:us-west-1:123412345678:certificate-authority/0116z123-dv7a-59b1-x7be-1231v7257113 \
    --certificate fileb://IACA-Root-CA-Cert.pem</code></pre> 
</div> 
<p>You should see <code style="color: #000000;">success</code> after running the command.</p> 
<h3>Step 6: Create an asymmetric key in AWS KMS</h3> 
<p>In this step, create an asymmetric signing key in AWS KMS which will be used to sign the mDL document signing CSR.</p> 
<ol> 
 <li>Use the following command to create an asymmetric key: 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">aws kms create-key \
    --region us-west-1 \
    --key-spec ECC_NIST_P256 \
    --key-usage SIGN_VERIFY</code></pre> 
  </div> </li> 
 <li>The command should produce the following output: 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">{
  "KeyMetadata": {
    "AWSAccountId": "123412345678",
    "KeyId": "3ab87971-1fe2-45d9-955a-5dc7f65558zf",
    "Arn": "arn:aws:kms:us-west-1:123412345678:key/3ab87971-1fe2-45d8-955c-5dc7f65558ef",
    "CreationDate": "2024-05-18T19:53:27.318000+00:00",
    "Enabled": true,
    "Description": "",
    "KeyUsage": "SIGN_VERIFY",
    "KeyState": "Enabled",
    "Origin": "AWS_KMS",
    "KeyManager": "CUSTOMER",
    "CustomerMasterKeySpec": "ECC_NIST_P256",
    "KeySpec": "ECC_NIST_P256",
    "SigningAlgorithms": [
      "ECDSA_SHA_256"
    ],
    "MultiRegion": false
  }
}</code></pre> 
  </div> </li> 
 <li>Note the Arn value from the output. You will use it in Step 7 to configure the CSR creation utility for the mDL document signing certificate.</li> 
</ol> 
<h3>Step 7: Use the CSR creation utility to generate the document signing CSR</h3> 
<p>We published a sample utility in GitHub that creates a CSR signed by an AWS asymmetric key.</p> 
<ol> 
 <li>Clone the <a href="https://github.com/aws-samples/csr-signing-using-kms" rel="noopener" target="_blank">GitHub repository</a> and then follow the instructions in the README file from the repository to configure and run it.</li> 
 <li>This program will output a PEM formatted CSR similar to the following: 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">-----BEGIN CERTIFICATE REQUEST-----
..
-----END CERTIFICATE REQUEST-----</code></pre> 
  </div> </li> 
 <li>Copy the output and store it in a file named <code style="color: #000000;">document-signing-kms.csr</code>. You will use the file in Step 8 to create the mDL document signing certificate based on this CSR.</li> 
</ol> 
<h3>Step 8: Generate an mDL document signing certificate</h3> 
<p>This step <a href="https://awscli.amazonaws.com/v2/documentation/api/2.0.34/reference/acm-pca/issue-certificate.html" rel="noopener" target="_blank">creates the document signing certificate</a> from the CSR that’s signed using the AWS KMS asymmetric key.</p> 
<ol> 
 <li>Create a file named <code style="color: #000000;">extensionSigner.txt</code> with the following contents. The contents of this file are derived from the Certificate profiles section of ISO/IEC 18013-5. The JSON snippet that follows shows the extension structure containing the <code style="color: #000000;">KeyUsage</code> extension with <code style="color: #000000;">DigitalSignature</code> field set to <code style="color: #000000;">true</code>. 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">{
     "Extensions": {
         "KeyUsage": {
             "DigitalSignature": true
         },
         "ExtendedKeyUsage": [
             {
                 "ExtendedKeyUsageObjectIdentifier": "1.0.18013.5.1.2"
             }
         ]
     }
}</code></pre> 
  </div> </li> 
 <li>Use the following AWS CLI command to create the certificate. 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">aws acm-pca issue-certificate \
    --region us-west-1 \
    --certificate-authority-arn arn:aws:acm-pca:us-west-1:123412345678:certificate-authority/0116z123-dv7a-59b1-x7be-1231v7257113 \
    --template-arn "arn:aws:acm-pca:::template/BlankEndEntityCertificate_APIPassthrough/V1" \
    --signing-algorithm "SHA256WITHECDSA" \
    --csr fileb://document-signing-kms.csr \
    --validity Value=1825,Type="DAYS" \
    --api-passthrough file://extensionSigner.txt</code></pre> 
  </div> </li> 
 <li>Output: 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">{
    "CertificateArn": "arn:aws:acm-pca:us-west-1:123412345678:certificate-authority/0116z123-dv7a-59b1-x7be-1231v7257113/certificate/d462fcd3b9h3beb45c7c312241d42fba"
}</code></pre> 
  </div> </li> 
 <li>You will use the <code style="color: #000000;">CertificateArn</code> from the output in Step 9 to retrieve the mDL document signing certificate.</li> 
</ol> 
<h3>Step 9: Retrieve the mDL document signing certificate</h3> 
<p>This step retrieves the document signing certificate in PEM format from AWS Private CA.</p> 
<ol> 
 <li>Use the following command to retrieve the document signing certificate: 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">aws acm-pca get-certificate \
    --region us-west-1 \
    --certificate-authority-arn arn:aws:acm-pca:us-west-1:123412345678:certificate-authority/0116z123-dv7a-59b1-x7be-1231v7257113 \
    --certificate-arn arn:aws:acm-pca:us-west-1:123412345678:certificate-authority/0116z123-dv7a-59b1-x7be-1231v7257113/certificate/d462fcd3b9h3beb45c7c312241d42fba \
    --output text</code></pre> 
  </div> </li> 
 <li>Store the output text in <code style="color: #000000;">document_signing_cert.pem</code>.</li> 
</ol> 
<p>You now have the mDL document signing certificate for packaging later with the <a href="https://cbor.io/" rel="noopener" target="_blank">Concise Binary Object Representation</a> (CBOR) structure required by ISO/IEC 18013-5.</p> 
<h3>Step 10: mDL reader ingests issuing authority’s mDL signing certificate chain</h3> 
<p>An mDL reader can trust the mDL presented by a user after cryptographically verifying the mDL. This verification requires the reader to possess the mDL signing certificate chain of the issuing authority that issued the user the mDL. As required by the decentralized public key infrastructure (PKI) trust model specified in ISO/IEC 18013-5, the mDL reader will ingest the mDL signing certificate chain of the issuing authority.</p> 
<h3>Step 11: User makes an mDL signing request to the issuing authority</h3> 
<p>The user makes a request to the issuing authority to sign the mDL.</p> 
<h3>Step 12: Issuing authority issues signed mDL to the user</h3> 
<p>The issuing authority will authenticate the user’s identity and issue a signed mDL. The issuing authority provisions mDL data to the user’s device along with a CBOR encoded object known as a mobile security object (MSO). MSOs contain a digest algorithm, individual digests of mDL data elements, and a validity period. After this MSO has been generated and encoded as required by ISO/IEC 18013-5:2021 section 9.1.2.4, the MSO can be signed by the issuing authority. This signature can be generated in AWS KMS as shown in the following command. Generating the encoded MSO is out of scope for this post.</p> 
<ol> 
 <li>Use the following command to produce the SHA-256 digest of encoded MSO object using the <a href="https://linux.die.net/man/1/sha256sum" rel="noopener" target="_blank">sha256sum</a> utility. 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">sha256sum &lt; EncodedMSO &gt; EncodedMSODigest</code></pre> 
  </div> </li> 
 <li>Sign the digest using the AWS KMS asymmetric key created in Step 6. 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">aws kms sign \
 --region us-west-1 \
 --key-id 3ab87971-1fe2-45d8-955c-5dc7f65558ef \
 --message fileb://EncodedMSODigest \
 --message-type DIGEST \
 --signing-algorithm ECDSA_SHA_256 \
 --output text \
 --query Signature | base64 --decode</code></pre> 
  </div> </li> 
 <li>This signature will be combined with the issuing authority certificate and the MSO to form a <a href="https://datatracker.ietf.org/doc/html/rfc8152" rel="noopener" target="_blank">CBOR Object Signing and Encryption (COSE)</a> signed message and will be presented with the mDL data elements to readers. Readers can validate this signature to confirm the integrity of the MSO.</li> 
</ol> 
<h3>Step 13: User presents their mDL to an mDL reader</h3> 
<p>The user presents their mDL to the mDL reader for identity verification, such as at an airport. This process is called mDL Initialization in ISO/IEC 18013-5:2021 section 6.3.2.2. The mDL is activated during this initialization step.</p> 
<h3>Step 14: An mDL reader requests mDL data from a user’s mobile device</h3> 
<p>The mDL reader issues an mDL retrieval request to the user’s mobile device. A key feature of mDLs is that they allow mDL holders to present a subset of their PII. An mDL reader will request specific attributes such as name and date of birth, requiring the mDL holder to consent to the release of this information. The mDL reader’s request contains the list of PII data element identifiers that it is requesting the mDL holder to share.</p> 
<h3>Step 15: User consents to share their mDL data</h3> 
<p>The user receives a prompt notifying them of mDL sharing request. This prompt shows the user the list of PII data elements that are being requested. The user consents to the request and the mDL data that includes the MSO is shared with the reader.</p> 
<h3>Step 16: Reader validates mDL integrity</h3> 
<p>The reader receives the mDL data and validates it for integrity. The inclusion of the MSO with the mDL data elements provides mDL readers with a mechanism for validating the integrity of the data they’ve received. The mDL reader can then hash and verify individual mDL data elements presented by the device. If all data elements match their corresponding entries in the MSO, the mDL device reader can attest that the data hasn’t been tampered with.</p> 
<p>As an example, assume that the mDL contains the following data elements:</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">24(&lt;&lt;
  {
    "digestID": 0,
    "random": h'BBA394B98088CAE238D35979F7210E18DFAF70354524D86149CA20046E4321B1',
    "elementIdentifer": "given_name",
    "elementValue": "John"
  }
&gt;&gt;),
24(&lt;&lt;
  {
    "digestID": 1,
    "random": h'901F63FD880A15B30EDCEEFA857201C52FB9EAD1D39C15BB592829D16CB8A368',
    "elementIdentifer": "family_name",
    "elementValue": "Doe"
  }
&gt;&gt;)</code></pre> 
</div> 
<p>And a Mobile Security Object containing the following data element digests:</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">24(&lt;&lt;
  {
    "version": "1.0",
    "digestAlgorithm": "SHA-256",
    "valueDigests":
    {
      "org.iso.18013.5.1":
      {
        0: h’D6AA81E454036313A9A681809151DDDBDF702289094F18286DDC591C41C6434E',
        1: h'4C3D83940CA8C5DE8060A23EB649C175E79B745B6A7D9939B4D16B3E46BB14D5'
      }
    }
  }
&gt;&gt;)</code></pre> 
</div> 
<p>The MSO’s integrity would first confirm that the validity period of the MSO (not shown) has not expired. It can then verify the signature (not shown) with the issuing authority’s public key. After this has been established, both data elements need to be verified. The CBOR representation of each element (<code style="color: #000000;">digestID</code>, <code style="color: #000000;">random</code>, <code style="color: #000000;">elementIdentifier</code>, and <code style="color: #000000;">elementValue</code>) is encoded as bytes and then hashed using SHA-256. For example, the following should equal <code style="color: #000000;">D6AA81E454036313A9A681809151DDDBDF702289094F18286DDC591C41C6434E</code>.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">SHA256(CBOR byte representation of 24(&lt;&lt;
    {
      "digestID": 0,
      "random": h'BBA394B98088CAE238D35979F7210E18DFAF70354524D86149CA20046E4321B1',
      "elementIdentifer": "given_name",
      "elementValue": "John"
    }
  &gt;&gt;))
)</code></pre> 
</div> 
<p>Likewise, the following example should equal<br /> <code style="color: #000000;">4C3D83940CA8C5DE8060A23EB649C175E79B745B6A7D9939B4D16B3E46BB14D5</code>.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">SHA256(CBOR byte representation of 24(&lt;&lt;
    {
      "digestID": 1,
      "random": h'901F63FD880A15B30EDCEEFA857201C52FB9EAD1D39C15BB592829D16CB8A368',
      "elementIdentifer": "family_name",
      "elementValue": "Doe"
    }
  &gt;&gt;)))</code></pre> 
</div> 
<p>If all data elements pass this hash verification check, then the presented mDL contents can be trusted by the mDL reader.</p> 
<h2>Summary</h2> 
<p>As you saw in this solution, mobile driver’s licenses (mDLs) provide increased security and flexible consent management to preserve privacy for individuals. The principles of cryptographic signing and verification aren’t new and both AWS KMS and AWS Private CA are well suited for supporting digital identity applications, whether it’s a driver’s license or some other kind of identification. To learn more about AWS KMS asymmetric keys and AWS Private CA, see <a href="https://aws.amazon.com/blogs/security/digital-signing-asymmetric-keys-aws-kms/" rel="noopener" target="_blank">Digital signing with the new asymmetric keys feature of AWS KMS</a> and <a href="https://aws.amazon.com/blogs/security/how-to-host-and-manage-an-entire-private-certificate-infrastructure-in-aws/" rel="noopener" target="_blank">How to host and manage an entire private certificate infrastructure in AWS</a>.</p> 
<p>If you have feedback about this post, submit comments in the Comments section below. If you have questions about this post, start a new thread on the <a href="https://repost.aws/tags/TAJ7zd4vjzSfC_8JNlsbq2tA/aws-certificate-manager" rel="noopener" target="_blank">AWS Certificate Manager re:Post</a> and <a href="https://repost.aws/tags/TAMC3vcPOPTF-rPAHZVRj1PQ/aws-key-management-service" rel="noopener" target="_blank">AWS AWS Key Management Service re:Post</a>, or <a href="https://console.aws.amazon.com/support/home" rel="noopener" target="_blank">contact AWS Support</a>.</p> 
<footer> 
 <div class="blog-author-box">
  <img alt="Ram Ramani" class="alignleft size-full wp-image-33350" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2019/05/16/ramani-author-photo.jpeg" style="margin-left: 12px; margin-right: 18px; margin-top: 12px; margin-bottom: 6px; width: 93.750px; height: 125px;" />
  <span class="lb-h4" style="line-height: 2.1em; padding-top: 12px; margin-top: 24px;">Ram Ramani</span>
  <br />Ram is a Principal Security architect in AWS, responsible for leading the data protection and privacy focus areas. Prior to this role, Ram held software developer positions at various organizations with a focus on applied math and machine learning.
 </div> 
 <div class="blog-author-box">
  <img alt="Raj Jain" class="alignleft size-full wp-image-33350" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/02/19/Raj-Jain-Author.jpeg" style="margin-left: 12px; margin-right: 18px; margin-top: 12px; margin-bottom: 6px; width: 93.750px; height: 125px;" />
  <span class="lb-h4" style="line-height: 2.1em; padding-top: 12px; margin-top: 24px;">Raj Jain</span>
  <br />Raj is a Senior Software Engineer in the Amazon FinTech organization, responsible for developing security and compliance services that underlie the AWS and broader Amazon infrastructure. Raj is a published author in the Bell Labs Technical Journal, has authored IETF standards, AWS security blogs, and holds twelve patents.
 </div> 
 <div class="blog-author-box">
  <img alt="Kyle Schultheiss" class="alignleft size-full wp-image-33350" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/06/29/Kyle-Schultheiss-Author.jpg" style="margin-left: 12px; margin-right: 18px; margin-top: 12px; margin-bottom: 6px; width: 93.750px; height: 125px;" />
  <span class="lb-h4" style="line-height: 2.1em; padding-top: 12px; margin-top: 24px;">Kyle Schultheiss</span>
  <br />Kyle is a Senior Software Engineer on the AWS Cryptography team. He has been working on the ACM Private Certificate Authority service since its inception in 2018. In prior roles, he contributed to other AWS services such as Amazon Virtual Private Cloud, Amazon EC2, and Amazon Route 53.
 </div> 
</footer>
