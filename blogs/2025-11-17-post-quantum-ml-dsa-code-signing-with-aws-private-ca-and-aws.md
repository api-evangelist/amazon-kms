---
title: "Post-quantum (ML-DSA) code signing with AWS Private CA and AWS KMS"
url: "https://aws.amazon.com/blogs/security/post-quantum-ml-dsa-code-signing-with-aws-private-ca-and-aws-kms/"
date: "Mon, 17 Nov 2025 19:40:02 +0000"
author: "Panos Kampanakis"
feed_url: "https://aws.amazon.com/blogs/security/category/security-identity-compliance/aws-key-management-service/feed/"
---
<p>Following our <a href="https://aws.amazon.com/about-aws/whats-new/2025/06/aws-kms-post-quantum-ml-dsa-digital-signatures/" rel="noopener" target="_blank">recent announcement of ML-DSA support in AWS Key Management Service</a> (AWS KMS), we <a href="https://aws.amazon.com/about-aws/whats-new/2025/11/aws-private-ca-post-quantum-digital-certificates/" rel="noopener" target="_blank">just introduced post-quantum ML-DSA signature support in AWS Private Certificate Authority</a> (AWS Private CA). Customers can use <a href="https://aws.amazon.com/private-ca/" rel="noopener" target="_blank">AWS Private CA</a> to create and manage their own private public key infrastructure (PKI) hierarchies. Through this integration, you can establish and use customer-managed quantum-resistant roots of trust for code signing, device authentication, outside (of AWS) workload authentication with <a href="https://aws.amazon.com/iam/roles-anywhere/" rel="noopener" target="_blank">AWS IAM Roles Anywhere</a>, or communication tunnels such as IKEv2/IPsec or Mutual TLS (mTLS) using private PKI.</p> 
<p>As outlined in the AWS post-quantum cryptography <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/blogs/security/aws-post-quantum-cryptography-migration-plan/" rel="noopener" target="_blank">migration plan</a></span>, establishing quantum-resistant roots of trust is critical for systems that need to maintain security for extended periods of time. ML-DSA, a signature scheme standardized in <span class="LinkEnhancement"><a class="Link" href="https://csrc.nist.gov/pubs/fips/204/final" rel="noopener" target="_blank">FIPS 204</a></span>, provides quantum resistance while maintaining the performance characteristics needed for deployments at scale.</p> 
<p>Previously, <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/blogs/security/code-signing-aws-certificate-manager-private-ca-aws-key-management-service-asymmetric-keys" rel="noopener" target="_blank">we shared</a></span> how to use AWS Private CA and <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/kms" rel="noopener" target="_blank">AWS KMS</a></span> for code signing. In this post, we show you how to combine the post-quantum signing capability provided by AWS KMS with post-quantum code-signing PKI from AWS Private CA. Consumers of signed code that have been pre-provisioned with the post-quantum PKI roots can rest assured that the software could not have been forged by an adversary with a cryptographically relevant quantum computer (CRQC). For demonstration purposes, we use the <span class="LinkEnhancement"><a class="Link" href="https://github.com/aws-samples/diy-code-signing-kms-private-ca" rel="noopener" target="_blank">diy-code-signing-kms-private-ca</a></span> sample program, which uses the <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/sdk-for-java/" rel="noopener" target="_blank">AWS SDK for Java</a></span>. This code creates a PKI infrastructure, generates a code-signing certificate, signs binary code, and verifies the signature. Although we break down the steps to demonstrate the functionality in this post, you can run the <span class="LinkEnhancement"><a class="Link" href="https://github.com/aws-samples/diy-code-signing-kms-private-ca/blob/master/src/main/java/com/amazonaws/acmpcakms/examples/Runner.java" rel="noopener" target="_blank">Runner</a></span> as-is to see it in action with commands found in the <span class="LinkEnhancement"><a class="Link" href="https://github.com/aws-samples/diy-code-signing-kms-private-ca/blob/master/README.md" rel="noopener" target="_blank">README</a></span> file.</p> 
<p>This post uses the Cryptographic Message Syntax (CMS) standard for encapsulating the signatures generated for input binary data. It stores the signature, X.509 certificate, and chain used to establish trust. The signature, known as a <i>detached signature</i>, doesn’t contain the original data. The detached signature can be used together with the original file, which was signed with standard tools such as OpenSSL natively to validate the authenticity of the file.</p> 
<h2>Create a post-quantum PKI hierarchy</h2> 
<p>For this post, we will use AWS Private CA to introduce a <i>code-signing PKI</i>. It will consist of a root CA to sign a subordinate CA, and a code-signing certificate signed by the subordinate CA. The whole chain will consist of quantum-resistant ML-DSA certificates.</p> 
<h3>CA hierarchy creation</h3> 
<p>First, the post-quantum CA hierarchy must be created with ML-DSA. In this example, we use the ML-DSA-65 variant of the post-quantum signature algorithm. You can do this with the AWS Java SDK as shown in the <span class="LinkEnhancement"><a class="Link" href="https://github.com/aws-samples/diy-code-signing-kms-private-ca/blob/master/src/main/java/com/amazonaws/acmpcakms/examples/Runner.java" rel="noopener" target="_blank">Runner.java</a></span> file:</p> 
<div class="Enhancement"> 
 <div class="Enhancement-item"> 
  <div class="CodeBlockWP hide-language"> 
   <div class="code-toolbar"> 
    <pre class="unlimited-height-code language-text"><code class="language-text">PrivateCA rootPrivateCA = PrivateCA.builder()
	.withCommonName(ROOT_COMMON_NAME)
	.withType(CertificateAuthorityType.ROOT)
	.withAlgorithmFamily(ML_DSA_65_ALGORITHM_FAMILY)
	.getOrCreate();

PrivateCA subordinatePrivateCA = PrivateCA.builder()
    .withIssuer(rootPrivateCA).withCommonName(SUBORDINATE_COMMON_NAME)
    .withType(CertificateAuthorityType.SUBORDINATE)
	.withAlgorithmFamily(ML_DSA_65_ALGORITHM_FAMILY)
    .getOrCreate();</code></pre> 
    <p></p>
   </div> 
   <p></p>
  </div> 
  <p></p>
 </div> 
 <p></p>
</div> 
<p></p> 
<div class="RichTextHeading"> 
 <h3>Code-signer creation</h3> 
 <p></p>
</div> 
<p>For code signing, you need an asymmetric key pair and a code-signing certificate. The asymmetric ML-DSA key pair is generated in AWS KMS and the code-signing certificate is issued by AWS Private CA.</p> 
<p><b>Create an ML-DSA key pair in AWS KMS</b></p> 
<p>First, you must create an asymmetric key pair for code signing operations. Similar to the creation of the hierarchy, the AWS Java SDK can be used to create that AWS KMS key (key pair). Signing will be taking place with the key pair’s private key in AWS KMS. The corresponding public key will be in the code-signing leaf certificate signed by the subordinate CA. These calls are performed as part of the main method within the <span class="LinkEnhancement"><a class="Link" href="https://github.com/aws-samples/diy-code-signing-kms-private-ca/blob/master/src/main/java/com/amazonaws/acmpcakms/examples/Runner.java" rel="noopener" target="_blank">Runner.java</a></span> file:</p> 
<div class="Enhancement"> 
 <div class="Enhancement-item"> 
  <div class="CodeBlockWP hide-language"> 
   <div class="code-toolbar"> 
    <pre class="unlimited-height-code language-text"><code class="language-text">AsymmetricCMK codeSigningCMK = AsymmetricCMK
    .builder().withAlias(CMK_ALIAS)
	.withAlgorithmFamily(ML_DSA_65_ALGORITHM_FAMILY)
    .getOrCreate();</code></pre> 
    <p></p>
   </div> 
   <p></p>
  </div> 
  <p></p>
 </div> 
 <p></p>
</div> 
<p>Alternatively, you can generate the key pair in AWS KMS with the <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/console/" rel="noopener" target="_blank">AWS Management Console</a></span> or the <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/cli" rel="noopener" target="_blank">AWS Command Line Interface (AWS CLI)</a></span> as shown in the <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/blogs/security/how-to-create-post-quantum-signatures-using-aws-kms-and-ml-dsa/" rel="noopener" target="_blank">ML-DSA KMS security blog</a></span>.</p> 
<p><b>Issue a code-signing certificate</b></p> 
<p>Creating a certificate signing request (CSR) using AWS Private CA is a two-step process. First, you must create a CSR that contains both an identity (Subject) and the previously created AWS KMS public key. The following code snippet in <span class="LinkEnhancement"><a class="Link" href="https://github.com/aws-samples/diy-code-signing-kms-private-ca/blob/master/src/main/java/com/amazonaws/acmpcakms/examples/Runner.java" rel="noopener" target="_blank">Runner.java</a></span> accomplishes this:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">String codeSigningCSR = codeSigningCMK
	.generateCSR(END_ENTITY_COMMON_NAME);
</code></pre> 
</div> 
<p>OpenSSL 3.5 or later can parse this CSR to view its content with the following command if the CSR contents have been written to disk at <code class="CodeInline" style="color: #000;">csr.pem:</code></p> 
<div class="Enhancement"> 
 <div class="Enhancement-item"> 
  <div class="CodeBlockWP hide-language"> 
   <div class="code-toolbar"> 
    <pre class="unlimited-height-code language-text"><code class="language-text">openssl req -in csr.pem -inform pem -text -noout
Certificate Request:
	Data:
		Version: 1 (0x0)
		Subject: CN=CodeSigningCertificate
		Subject Public Key Info:
			Public Key Algorithm: ML-DSA-65
				ML-DSA-65 Public-Key:
				pub:
					&lt;Public Key Data&gt;   
		Attributes:
			Requested Extensions:
				X509v3 Basic Constraints:
					CA:FALSE
	Signature Algorithm: ML-DSA-65
	Signature Value:
		&lt;Signature Data&gt;</code></pre> 
    <p></p>
   </div> 
   <p></p>
  </div> 
  <p></p>
 </div> 
 <p></p>
</div> 
<p>You can see that the CSR contains an ML-DSA-65 public key. Its corresponding private key will be used to sign code.</p> 
<p>The CSR is used by the subordinate CA to issue the code-signing certificate. Note that the code-signing template is used in the <code class="CodeInline" style="color: #000;">templateArn</code> of the <code class="CodeInline" style="color: #000;">IssueCertificate</code> request in the relevant <span class="LinkEnhancement"><a class="Link" href="https://github.com/aws-samples/diy-code-signing-kms-private-ca/blob/5f78213d6ea8b0260e8e3da9f529d6a6021b28fb/src/main/java/com/amazonaws/acmpcakms/examples/PrivateCA.java#L189" rel="noopener" target="_blank">PrivateCA.java file</a></span>. The inclusion of this template helps ensure that AWS Private CA will issue a certificate with the correct Key Usage (KU) and Extended Key Usage (EKU) extension values, regardless of the values presented in the CSR.</p> 
<div class="Enhancement"> 
 <div class="Enhancement-item"> 
  <div class="CodeBlockWP hide-language"> 
   <div class="code-toolbar"> 
    <pre class="unlimited-height-code language-text"><code class="language-text">IssueCertificateRequest issueCertificateRequest = IssueCertificateRequest.builder()
	.idempotencyToken(UUID.randomUUID().toString())
	.certificateAuthorityArn(subordinatePrivateCA.arn())
	.csr(SdkBytes.fromUtf8String(csr))
	.signingAlgorithm(algorithmFamily.getPcaSigningAlgorithm())
	.templateArn("arn:aws:acm-pca:::template/CodeSigningCertificate/V1")
	.validity(validity)
	.build();

IssueCertificateResponse issueCertificateResponse = client
	.issueCertificate(issueCertificateRequest);

String certificateArn = issueCertificateResponse.certificateArn();

GetCertificateRequest getCertificateRequest = GetCertificateRequest.builder()
	.certificateAuthorityArn(ca.arn())
	.certificateArn(certificateArn)
	.build();</code></pre> 
    <p></p>
   </div> 
   <p></p>
  </div> 
  <p></p>
 </div> 
 <p></p>
</div> 
<p>The response includes the ML-DSA-65 code-signing certificate. You can use OpenSSL 3.5 or later to inspect the contents of the certificate after you save it to a file named <code class="CodeInline" style="color: #000;">code-signing-cert.pem</code>:</p> 
<div class="Enhancement"> 
 <div class="Enhancement-item"> 
  <div class="CodeBlockWP hide-language"> 
   <div class="code-toolbar"> 
    <pre class="unlimited-height-code language-text"><code class="language-text">openssl x509 -in code-signing-cert.pem -inform pem -text -noout
Certificate:
	Data:
		Version: 3 (0x2)
		Serial Number:
			1a:15:af:1e:64:8d:cd:29:b4:dc:66:2a:8b:1e:ee:b0
		Signature Algorithm: ML-DSA-65
		Issuer: CN=CodeSigningSubordinate-MLDSA65
		Validity
			Not Before: Sep 24 13:10:38 2025 GMT
			Not After : Sep 24 14:10:38 2026 GMT
		Subject: CN=CodeSigningCertificate
		Subject Public Key Info:
			Public Key Algorithm: ML-DSA-65
				ML-DSA-65 Public-Key:
				pub:
					&lt;Public Key Data&gt;
		X509v3 extensions:
			X509v3 Basic Constraints:
				CA:FALSE
			X509v3 Authority Key Identifier:
B7:EF:2E:C9:7A:A8:7E:B5:D6:2D:9A:3F:C7:A7:F8:9D:74:01:6A:EF
			X509v3 Subject Key Identifier:

7F:63:35:0C:56:F8:ED:F1:2A:DF:B5:2E:7C:F1:2C:D9:A0:0E:63:B6
			X509v3 Key Usage: critical
				Digital Signature
			X509v3 Extended Key Usage: critical
				Code Signing
	Signature Algorithm: ML-DSA-65
	Signature Value:
		&lt;Signature Data&gt;</code></pre> 
    <p></p>
   </div> 
   <p></p>
  </div> 
  <p></p>
 </div> 
 <p></p>
</div> 
<p>You can see that the certificate includes the ML-DSA-65 public key of the code-signing key pair and the ML-DSA-65 signature from the subordinate CA. You also see the KU and the EKU values, which represent a code-signing certificate from the AWS Private CA template.</p> 
<h2>Sign code</h2> 
<p>At this point, you have set up the code-signing PKI, have a code-signing certificate issued by AWS Private CA and a corresponding ML-DSA key pair residing in KMS.</p> 
<p>The Java SDK can be used to generate a CMS signature for a code binary file. In the background, this is accomplished by calling the <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/kms/latest/APIReference/API_Sign.html" rel="noopener" target="_blank">AWS KMS Sign API</a></span> with the ML-DSA key pair as shown in <span class="LinkEnhancement"><a class="Link" href="https://github.com/aws-samples/diy-code-signing-kms-private-ca/blob/master/src/main/java/com/amazonaws/acmpcakms/examples/Runner.java" rel="noopener" target="_blank">Runner.java</a></span>. The following is part of the Java code. This first snippet involves building a certificate chain and then using it along with the code-signing AWS KMS key, the signer’s certificate, and <code class="CodeInline" style="color: #000;">&lt;DATA_TO_SIGN&gt;</code>, the byte array representation of the code file, to generate the detached signature in a CMS structure.</p> 
<div class="Enhancement"> 
 <div class="Enhancement-item"> 
  <div class="CodeBlockWP hide-language"> 
   <div class="code-toolbar"> 
    <pre class="unlimited-height-code language-text"><code class="language-text">	// Parse code-signing certificate from PEM
	X509CertificateHolder signerCert = CertificateUtils
		.fromPEM(codeSigningCertificate.certificate());

	Collection&lt;X509CertificateHolder&gt; chainCerts = CertificateUtils
		.toCertificateHolders(codeSigningCertificate.certificateChain());

	// Build certificate chain including code-signing cert and intermediate certs
	Collection&lt;X509CertificateHolder&gt; certChain = new ArrayList&lt;&gt; ();
	certChain.add(signerCert);

	// Parse certificate chain
	for (X509CertificateHolder chainCert : chainCerts) {
		if (!chainCert.equals(signerCert)) {
			certChain.add(chainCert);
		}
	}

	// Create detached CMS signature
	CMSCodeSigningObject cmsCodeSigningObject = CMSCodeSigningObject
		.createDetachedSignature(
			codeSigningCMK,
			ML_DSA_65_ALGORITHM_FAMILY,
			&lt;DATA_TO_SIGN&gt;,
			signerCert,
			certChain);</code></pre> 
    <p></p>
   </div> 
   <p></p>
  </div> 
  <p></p>
 </div> 
 <p></p>
</div> 
<p>The code-signing object is written to disk in <code class="CodeInline" style="color: #000;">signature-MLDSA65.p7s</code>. You can inspect it with OpenSSL 3.5 or later:</p> 
<div class="Enhancement"> 
 <div class="Enhancement-item"> 
  <div class="CodeBlockWP hide-language"> 
   <div class="code-toolbar"> 
    <pre class="unlimited-height-code language-text"><code class="language-text">openssl cms -cmsout -in signature-MLDSA65.p7s -inform DER -print
CMS_ContentInfo:
	contentType: pkcs7-signedData (1.2.840.113549.1.7.2)
	d.signedData:
		version: 1
		digestAlgorithms:
			algorithm: shake256 (2.16.840.1.101.3.4.2.12)
			parameter: &lt;ABSENT&gt;
		encapContentInfo:
			eContentType: pkcs7-data (1.2.840.113549.1.7.1)
			eContent: &lt;ABSENT&gt;
		certificates:
			d.certificate:
				cert_info:
					version: 2
					serialNumber: 0xD0B2937F5BABC80AD55C0A90E1DE7057
					signature:
						algorithm: ML-DSA-65 (2.16.840.1.101.3.4.3.18)
						parameter: &lt;ABSENT&gt;
					issuer:			CN=CodeSigningSubordinate-MLDSA65
					validity:
						notBefore: Oct 28 15:05:27 2025 GMT
						notAfter: Oct 28 16:05:26 2026 GMT
					subject:		CN=CodeSigningCertificate
					key:		X509_PUBKEY:
						algor:
							algorithm: ML-DSA-65 (2.16.840.1.101.3.4.3.18)
							parameter: &lt;ABSENT&gt;
						public_key:(0 unused bits)
							...
						issuerUID: &lt;ABSENT&gt;
						subjectUID: &lt;ABSENT&gt;
						extensions:
							object: X509v3 Basic Constraints (2.5.29.19)
							critical: FALSE
							value:
								0000 - 30 00 0.
                                    
							object: X509v3 Authority Key Identifier (2.5.29.35)
							critical: FALSE
							value:
								0000 - 30 16 80 14 b7 ef 2e c9-7a a8 7e b5 d60.......z.~..
								000d - 2d 9a 3f c7 a7 f8 9d 74-01 6a ef-.?....t.j.

                        	object: X509v3 Subject Key Identifier (2.5.29.14)
							critical: FALSE
							value:
								0000 - 04 14 7f 63 35 0c 56 f8-ed f1 2a df b5...c5.V...*..
								000d - 2e 7c f1 2c d9 a0 0e 63-b6.|.,...c.

                         	object: X509v3 Key Usage (2.5.29.15)
							critical: TRUE
							value:
								0000 - 03 02 07 80....
                                    
							object: X509v3 Extended Key Usage (2.5.29.37)
							critical: TRUE
							value:
								0000 - 30 0a 06 08 2b 06 01 05-05 07 03 030...+.......
					sig_alg:
						algorithm: ML-DSA-65 (2.16.840.1.101.3.4.3.18)
						parameter: &lt;ABSENT&gt;
					signature:(0 unused bits)
						...
		d.certificate:
			cert_info:
			version: 2
			serialNumber: 29577999257397559174219641462943780786
			signature:
				algorithm: ML-DSA-65 (2.16.840.1.101.3.4.3.18)
				parameter: &lt;ABSENT&gt;
				issuer:			CN=CodeSigningRoot-MLDSA65
				[...]
                
		d.certificate:
			cert_info:
			version: 2
			serialNumber: 0xB9419A2C5D2422B3A58A5B449546D74B
			signature:
				algorithm: ML-DSA-65 (2.16.840.1.101.3.4.3.18)
				parameter: &lt;ABSENT&gt;
				issuer:			CN=CodeSigningRoot-MLDSA65
				[...]
	crls:
		&lt;ABSENT&gt;
	signerInfos:
		version: 1
		d.issuerAndSerialNumber:
			issuer:				CN=CodeSigningSubordinate-MLDSA65
			serialNumber: 0xD0B2937F5BABC80AD55C0A90E1DE7057
		digestAlgorithm:
			algorithm: shake256 (2.16.840.1.101.3.4.2.12)
			parameter: &lt;ABSENT&gt;
		signedAttrs:
			object: contentType (1.2.840.113549.1.9.3)
			set:
				OBJECT:pkcs7-data (1.2.840.113549.1.7.1)

			object: signingTime (1.2.840.113549.1.9.5)
			set:
				UTCTIME:Oct 28 16:05:27 2025 GMT

			object: id-aa-CMSAlgorithmProtection (1.2.840.113549.1.9.52)
			set:
				SEQUENCE:
	0:d=0hl=2 l=26 cons: SEQUENCE
	2:d=1hl=2 l=11 cons:SEQUENCE
	4:d=2hl=2 l=9 prim:OBJECT:shake256
	15:d=1hl=2 l=11 cons:cont [ 1 ]
	17:d=2hl=2 l=9 prim:OBJECT:ML-DSA-65

        	object: messageDigest (1.2.840.113549.1.9.4)
			set:
				OCTET STRING:
					...
		signatureAlgorithm:
			algorithm: ML-DSA-65 (2.16.840.1.101.3.4.3.18)
			parameter: &lt;ABSENT&gt;
		signature:
			[...]</code></pre> 
    <p></p>
   </div> 
   <p></p>
  </div> 
  <p></p>
 </div> 
 <p></p>
</div> 
<p>The CMS signature object directly encapsulates both the code-signing certificate and the subordinate CA certificate. It’s expected that the root certificate will reside in a customer-managed trust store. In addition to these certificates, the CMS object also contains the digest of the input data within the <code class="CodeInline" style="color: #000;">signedAttrs</code> of the <code class="CodeInline" style="color: #000;">signerInfos</code> in the ASN.1 structure. The digest algorithm is SHAKE256 and the OCTET STRING represents the binary digest itself. The use of ML-DSA in CMS is specified in <span class="LinkEnhancement"><a class="Link" href="https://datatracker.ietf.org/doc/html/rfc9882" rel="noopener" target="_blank">RFC9882</a></span>.</p> 
<blockquote>
 <p><b>Note</b>: Although this example uses one ML-DSA signature, some use cases might include dual signatures, a traditional and a quantum-resistant one. Such signed artifacts can be backwards compatible with legacy verifiers that support and can only verify the traditional signature. Upgraded verifiers can verify both signatures.</p>
</blockquote> 
<h2>Verify signed code</h2> 
<p>Before loading a signed code artifact, its signature needs to be verified. That includes verifying the signature of the code and validating the certificate chain to the trusted root CA. The following code snippet from the main method within the file <span class="LinkEnhancement"><a class="Link" href="https://github.com/aws-samples/diy-code-signing-kms-private-ca/blob/master/src/main/java/com/amazonaws/acmpcakms/examples/Runner.java" rel="noopener" target="_blank">Runner.java</a></span> is used for the certificate chain validation and the signature in the code object:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">X509CertificateHolder rootCACertificate = CertificateUtils.fromPEM(rootCACertificatePEM); 
cmsCodeSigningObject.verifyDetachedSignature(&lt;DATA_TO_SIGN&gt;, rootCACertificate);
</code></pre> 
</div> 
<p>The preceding code retrieves the ML-DSA public key from the code-signing certificate; AWS access or credentials aren’t needed to validate the signature. Entities that have the root CA certificate loaded in their trust store can verify it without needing access to the <span class="LinkEnhancement"><a class="Link" href="https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/kms.html#KMS.Client.verify" rel="noopener" target="_blank">AWS KMS verify API</a></span>.</p> 
<blockquote>
 <p><b>Note:</b> The <span class="LinkEnhancement"><a class="Link" href="https://github.com/aws-samples/diy-code-signing-kms-private-ca/blob/master/src/main/java/com/amazonaws/acmpcakms/examples/Runner.java" rel="noopener" target="_blank">Runner.java</a></span> implementation doesn’t use a certificate trust store that’s either part of a browser or part of a file system within the resident operating system of a device or a server. The trust store is placed in an instance of a Java class object for the purpose of this post. If you’re planning to use this code-signing example in a production system, you <i>must</i> change the implementation to use a trust store on the host. To do so, you can build and distribute a secure trust store that includes the root CA certificate.</p>
</blockquote> 
<p>Alternatively, OpenSSL 3.5 or later can be used to validate the detached signature of the provided input data file with <code class="CodeInline" style="color: #000;">root-ca-MLDSA65.pem,</code> the provided root CA certificate from AWS Private CA.</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">openssl cms -verify -in signature-MLDSA65.p7s -content &lt;input-data-file&gt; \
            -CAfile root-ca-MLDSA65.pem -inform DER -purpose any \
            -binary -out /dev/null
CMS Verification successful
</code></pre> 
</div> 
<blockquote>
 <p><b>Note: </b>Although this post focused on code-signing, AWS Private CA can enable post-quantum ML-DSA authentication for other private PKI use cases. In one scenario, applications outside of AWS can access AWS resources by <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/blogs/security/iam-roles-anywhere-with-an-external-certificate-authority/" rel="noopener" target="_blank">temporarily using certificate-based authentication and swapping it with AWS credentials with AWS IAM Roles Anywhere</a></span>. AWS IAM Roles Anywhere now supports ML-DSA PKIs like the one created in this post. In another scenario, an mTLS client or IKEv2/IPsec peer could use an ML-DSA certificate issued by AWS Private CA to be authenticated by a server or peer respectively who has been pre-provisioned with the post-quantum PKI root certificate.</p>
</blockquote> 
<h2>Conclusion</h2> 
<p>This announcement marks an important milestone for post-quantum authentication. With the introduction of ML-DSA X.509 certificates in AWS Private CA, customers can bring quantum resistance to their private PKI use cases. These use cases include client authentication for mTLS or IKEv2/IPsec tunnels, IAM Roles Anywhere, or applications that use private PKI authentication. ML-DSA certificates with AWS Private CA and signing with AWS KMS also enable post-quantum code-singing and establishing post-quantum long-lived roots of trust for devices designed to operate for a long time even after CRQCs became available. Learn more about <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/security/post-quantum-cryptography/" rel="noopener" target="_blank">post-quantum cryptography</a></span> in general and the overall AWS plan to <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/blogs/security/aws-post-quantum-cryptography-migration-plan/" rel="noopener" target="_blank">migrate to post-quantum cryptography</a></span>.</p> 
<hr /> 
<p>If you have feedback about this post, submit comments in the Comments section below. If you have questions about this post, start a new thread on the <a href="https://repost.aws/topics/TAEEfW2o7QS4SOLeZqACq9jA/security-identity-compliance%3Fsc_ichannel=ha&amp;sc_ilang=en&amp;sc_isite=repost&amp;sc_iplace=hp&amp;sc_icontent=TAEEfW2o7QS4SOLeZqACq9jA&amp;sc_ipos=0" rel="noopener" target="_blank">AWS Security, Identity, &amp; Compliance re:Post</a> or <a href="https://console.aws.amazon.com/support/home" rel="noopener" target="_blank">contact AWS Support</a>. For more details regarding AWS PQC efforts, refer to our <a href="https://aws.amazon.com/security/post-quantum-cryptography/" rel="noopener" target="_blank">PQC page</a>.<br />&nbsp;</p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Panos Kampanakis" class="aligncenter size-full wp-image-26603" height="1600" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2022/07/21/Panos-Profile-pic.jpg" width="1142" /> 
  </div> 
  <h3 class="lb-h4">Panos Kampanakis</h3> 
  <p>Panos is a Principal Security Engineer at AWS. He has experience with cybersecurity, applied cryptography, security automation, and vulnerability management. He has coauthored publications on cybersecurity and participated in various security standards bodies to provide common interoperable protocols and languages for security information sharing, cryptography, and public-key infrastructure. Currently, he works with engineers and industry standards partners to provide cryptographically secure tools, protocols, and standards.</p> 
  <p></p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Jake Massimo" class="aligncenter size-full wp-image-36679" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/06/09/jakemas.jpg" width="120" /> 
  </div> 
  <h3 class="lb-h4">Jake Massimo</h3> 
  <p>Jake is a Senior Applied Scientist on the AWS Cryptography team. His work interfaces Amazon with the global cryptographic community through international conferences, academic literature, and standards organizations and influences the adoption of post-quantum cloud-scale cryptographic technology. Recently, his focus has been architecting AWS post-quantum cryptographic capabilities, including core libraries and infrastructure so that AWS and customers can seamlessly transition to quantum-safe cryptography.</p> 
  <p></p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Author" class="aligncenter size-full wp-image-14159" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/06/29/Kyle-Schultheiss-Author.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Kyle Schultheiss</h3> 
  <p>Kyle is a Senior Software Engineer on the AWS Cryptography team. He has been working on the AWS Private Certificate Authority service since its inception in 2018. In prior roles, he contributed to other AWS services such as Amazon Virtual Private Cloud, Amazon EC2, and Amazon Route 53.</p> 
 </div> 
</footer>
