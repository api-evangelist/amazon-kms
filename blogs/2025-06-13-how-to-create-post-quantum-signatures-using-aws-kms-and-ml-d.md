---
title: "How to create post-quantum signatures using AWS KMS and ML-DSA"
url: "https://aws.amazon.com/blogs/security/how-to-create-post-quantum-signatures-using-aws-kms-and-ml-dsa/"
date: "Fri, 13 Jun 2025 18:11:30 +0000"
author: "Jake Massimo"
feed_url: "https://aws.amazon.com/blogs/security/category/security-identity-compliance/aws-key-management-service/feed/"
---
<p>As the capabilities of quantum computing evolve, AWS is committed to helping our customers stay ahead of emerging threats to public-key cryptography. Today, we’re announcing the integration of <a href="https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.204.pdf" rel="noopener noreferrer" target="_blank">FIPS 204: Module-Lattice-Based Digital Signature Standard (ML-DSA)</a> into <a href="https://aws.amazon.com/kms" rel="noopener noreferrer" target="_blank">AWS Key Management Service (AWS KMS)</a>. Customers can now create and use ML-DSA keys through the same familiar AWS KMS APIs they use today for digital signatures, including <code style="color: #000000;">CreateKey</code>, <code style="color: #000000;">Sign</code>, and <code style="color: #000000;">Verify</code> operations. This new feature is generally available and you can use ML-DSA in the following AWS Regions: US West (N. California), and Europe (Milan) with the remaining commercial Regions to follow in the coming days. This launch is part of our broader AWS post-quantum cryptography migration plan, which we covered in our recent <a href="https://aws.amazon.com/blogs/security/aws-post-quantum-cryptography-migration-plan/" rel="noopener noreferrer" target="_blank">blog post</a>. In this post, we guide you through creating ML-DSA keys and post-quantum signatures with AWS KMS.</p> 
<p>Many organizations use AWS KMS to cryptographically sign firmware, operating systems, applications, or other artifacts. With ML-DSA support in AWS KMS, you can now generate and use post-quantum keys for signing operations within FIPS-140-3 Level 3 certified HSMs. By implementing ML-DSA signatures now, you can help make sure that your systems remain secure throughout their operational lifetime, even if cryptographically relevant quantum computers become available. This is especially important for manufacturers who install long-lived roots of trust during production—whether embedded directly in hardware or in devices that might remain offline for extended periods. In both cases, cryptographic signatures cannot be easily updated after deployment, making post-quantum readiness critical for the entire operational lifetime of these systems.</p> 
<h2>What’s new</h2> 
<p>AWS KMS offers three new <a href="https://docs.aws.amazon.com/kms/latest/developerguide/symm-asymm-choose-key-spec.html" rel="noopener noreferrer" target="_blank">AWS KMS key specs</a>: <code style="color: #000000;">ML_DSA_44</code>, <code style="color: #000000;">ML_DSA_65</code>, and <code style="color: #000000;">ML_DSA_87</code>, which you can use with the new post-quantum <a href="https://docs.aws.amazon.com/kms/latest/APIReference/API_Sign.html#KMS-Sign-request-SigningAlgorithm" rel="noopener noreferrer" target="_blank">SigningAlgorithm</a> <code style="color: #000000;">ML_DSA_SHAKE_256</code>. Like our other signing algorithms, this name includes the hash function that’s used within the signature scheme to digest messages before signing or verification. In this case, the hash function used is SHAKE256—part of the SHA-3 family of hash functions standardized by NIST in <a href="https://nvlpubs.nist.gov/nistpubs/fips/nist.fips.202.pdf" rel="noopener noreferrer" target="_blank">FIPS 202</a>.</p> 
<p>Table 1 shows the details for each key spec, including their NIST security categories and corresponding key sizes in bytes. Each ML-DSA key spec represents a balance between security strength and resource requirements. ML-DSA-44 is suitable for applications requiring security comparable to classical 128-bit encryption, while ML-DSA-65 and ML-DSA-87 provide progressively stronger security levels equivalent to classical 192-bit and 256-bit encryption, respectively. As you move up in security levels, you’ll notice corresponding increases in key and signature sizes, enabling you to choose the key spec that best matches your security needs and engineering constraints.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Key spec</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>NIST security Level</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Public key (B)</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Private key (B)</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Signature (B)</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ML_DSA_44</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">1 (equivalent to 128-bit security)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">1312</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">2560</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">2420</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ML_DSA_65</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">3 (equivalent to 192-bit security)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">1952</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">4032</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">3309</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ML_DSA_87</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">5 (equivalent to 256-bit security)</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">2592</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">4896</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">4627</td> 
  </tr> 
 </tbody> 
</table> 
<p>When using the AWS KMS Sign API with a RAW <a href="https://docs.aws.amazon.com/kms/latest/APIReference/API_Sign.html#KMS-Sign-request-MessageType" rel="noopener noreferrer" target="_blank">MessageType</a>, the message to be signed is limited to 4096 bytes. For messages larger than 4096 bytes, pre-processing the message outside of AWS KMS to create what’s known as <a href="https://csrc.nist.gov/csrc/media/Projects/post-quantum-cryptography/documents/faq/fips204-sec6-03192025.pdf" rel="noopener noreferrer" target="_blank">µ (mu)</a> is required to generate a smaller-sized message input to the KMS Sign API. This <em>external mu</em> process pre-digests the message using the public key of the ML-DSA signing key pair to create a message size of 64 bytes. To support this launch, we’ve added a new message type in the KMS Sign API—<code style="color: #000000;">EXTERNAL_MU</code>—that can be used with ML-DSA signing or verification calls to indicate when a message has been pre-processed using <a href="https://csrc.nist.gov/csrc/media/Projects/post-quantum-cryptography/documents/faq/fips204-sec6-03192025.pdf" rel="noopener noreferrer" target="_blank">µ (mu)</a> before submitted to AWS KMS.</p> 
<p>In the following sections, we include more information about constructing external mu and demonstrate basic AWS KMS operations with ML-DSA. We cover key creation, signature generation and verification, and both <code style="color: #000000;">RAW</code> and <code style="color: #000000;">EXTERNAL_MU</code> signing modes. Note that the produced <code style="color: #000000;">RAW</code> or <code style="color: #000000;">EXTERNAL_MU</code> ML-DSA signatures are identical when the same message and signing key are used.</p> 
<h2>Creating ML-DSA keys</h2> 
<p>To start, create an asymmetric AWS KMS key using the <a href="https://aws.amazon.com/cli" rel="noopener noreferrer" target="_blank">AWS Command Line Interface (AWS CLI)</a> example command:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">aws kms create-key --key-spec ML_DSA_65 --key-usage SIGN_VERIFY
</code></pre> 
</div> 
<p>This command will return a response similar to the following:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">{
    "KeyMetadata": {
        "Origin": "AWS_KMS",
        "KeyId": "<span style="color: #ff0000;"><em>1234abcd-12ab-34cd-56ef-1234567890ab</em></span>",
        "MultiRegion": false,
        "Description": "",
        "KeyManager": "CUSTOMER",
        "Enabled": true,
        "SigningAlgorithms": [
            "ML_DSA_SHAKE_256"
        ],
        "CustomerMasterKeySpec": "ML_DSA_65",
        "KeyUsage": "SIGN_VERIFY",
        "KeySpec": "ML_DSA_65",
        "KeyState": "Enabled",
        "CreationDate": 1748371316.734,
        "Arn": "<span style="color: #ff0000;"><em>arn:aws:kms:us-west-2:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab</em></span>",
        "AWSAccountId": "<span style="color: #ff0000;"><em>111122223333</em></span>"
    }
}
</code></pre> 
</div> 
<p>Make note of the <code style="color: #000000;">KeyId</code> or <code style="color: #000000;">Arn</code> value from the response; you’ll need this to reference your key in subsequent signing operations. The response confirms that the creation of an <code style="color: #000000;">ML_DSA_65</code> key configured for <code style="color: #000000;">SIGN_VERIFY</code> operations, which will use the <code style="color: #000000;">ML_DSA_SHAKE_256</code> signing algorithm for signature operations.</p> 
<h2>Signing</h2> 
<p>In this section, we include some examples of ML-DSA signing and verifying a JSON Web Token (JWT) commonly used to transfer claims between parties for web authorization. In 2021, we described how to sign and verify JWTs with Elliptic Curve Digital Signature Algorithm (ECDSA), a classic asymmetric cryptographic algorithm (see <a href="https://aws.amazon.com/blogs/security/how-to-verify-aws-kms-signatures-in-decoupled-architectures-at-scale/" rel="noopener noreferrer" target="_blank">How to verify AWS KMS signatures in decoupled architectures at scale</a>). In the following examples, the token is instead signed with an ML-DSA private key managed by AWS KMS and verified either within AWS KMS or externally using OpenSSL.</p> 
<p>The JWT content to be signed is from <a href="https://www.rfc-editor.org/rfc/rfc7519#section-3.1" rel="noopener noreferrer" target="_blank">section 3.1 of RFC7519</a>. More specifically, the JWT header is:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">{"typ":"JWT",
 "alg":"ML-DSA-65"}
</code></pre> 
</div> 
<p>And the JWT claim set is:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">{"iss":"joe",
 "exp":1748952000,
 "http://example.com/is_root":true}
</code></pre> 
</div> 
<p>You can produce the JWT message to be signed by using the Base64URL encoding of the header and payload as:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">echo -n -e '{"typ":"JWT",\015\012 "alg":"ML-DSA-65"}' | \
	basenc --base64url -w 0 | \
	sed 's/=//g' ; echo -n "." ; echo -n -e '{"iss":"joe",\015\012 "exp":1748952000,\015\012 "http://example.com/is_root":true}' | \
	basenc --base64url -w 0 | sed 's/=//g' ; echo ""
</code></pre> 
</div> 
<p>This command will output the following Base64 to be signed with ML-DSA:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">eyJ0eXAiOiJKV1QiLA0KICJhbGciOiJNTC1EU0EtNjUifQ.eyJpc3MiOiJqb2UiLA0KICJleHAiOjE3NDg5NTIwMDAsDQogImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ
</code></pre> 
</div> 
<p>Note that the following examples output the ML-DSA signature produced on the message by using the ML-DSA private key managed by AWS KMS in a binary format. You need to convert them to Base64URL to use them in JWT, but various data encryption and signing formats can use these signatures. These include Cryptographic Message Syntax (CMS), CBOR Object Signing and Encryption (COSE), or image signing encodings for UEFI and Open Titan. While converting between binary and these formats is straightforward, support for the new algorithms might not be available in common cryptographic implementations of these signing formats at the time of this writing.</p> 
<h3>RAW ML-DSA signing (no external mu)</h3> 
<p>To sign a message of less than 4096 bytes in AWS KMS with ML-DSA, you can use the AWS CLI:aws kms sign \</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">aws kms sign \
    --key-id <span style="color: #ff0000;"><em>&lt;1234abcd-12ab-34cd-56ef-1234567890ab&gt;</em></span> \
    --message ' eyJ0eXAiOiJKV1QiLA0KICJhbGciOiJNTC1EU0EtNjUifQ.eyJpc3MiOiJqb2UiLA0KICJleHAiOjE3NDg5NTIwMDAsDQogImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ' \
    --message-type RAW \
    --signing-algorithm ML_DSA_SHAKE_256 \
    --output text \
    --query Signature | base64 --decode &gt; ExampleSignature.bin
</code></pre> 
</div> 
<p>Make sure to replace the <code style="color: #000000;">target-key-id</code> value of <code style="color: #ff0000;"><em>&lt;1234abcd-12ab-34cd-56ef-1234567890ab&gt;</em></code> with your <code style="color: #000000;">KeyId</code>. This command will produce a signature and write it to disk as <code style="color: #000000;">ExampleSignature.bin</code>.</p> 
<p>After producing the signature, you can create the complete JWT (consisting of header, payload, and signature) with a single command:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">echo -n "eyJ0eXAiOiJKV1QiLA0KICJhbGciOiJNTC1EU0EtNjUifQ.eyJpc3MiOiJqb2UiLA0KICJleHAiOjE3NDg5NTIwMDAsDQogImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ." ; \
	basenc --base64url -w 0 ExampleSignature.bin | \
	sed 's/=//g' ; echo ""
</code></pre> 
</div> 
<p>This command will output a ready-to-use JWT in the format required by <a href="https://datatracker.ietf.org/doc/html/rfc7519" rel="noopener noreferrer" target="_blank">RFC 7519</a> and signed using AWS KMS:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">eyJ0eXAiOiJKV1QiLA0KICJhbGciOiJNTC1EU0EtNjUifQ.eyJpc3MiOiJqb2UiLA0KICJleHAiOjE3NDg5NTIwMDAsDQogImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ.<span style="color: #ff0000;"><em>&lt;base64url of the signature as per RFC7519&gt;</em></span>
</code></pre> 
</div> 
<p id="external-mu"></p> 
<h3>External mu ML-DSA signing</h3> 
<p>Note that AWS KMS imposes a 4096-byte limit on the size of the raw message when using the Sign API to minimize the latency of the response. In cases where the message to be signed is larger than 4096 bytes or if pre-digesting the external mu has performance advantages you need, you must use the <code style="color: #000000;">EXTERNAL_MU</code> message type instead of <code style="color: #000000;">RAW</code> in AWS KMS. </p> 
<p>Before using the <code style="color: #000000;">EXTERNAL_MU</code> message type with the AWS KMS Sign API, you must locally perform a pre-hash calculation on your message. So, first, retrieve the public key from AWS KMS, and convert it to DER format using the following command (replace the example key ID with a valid key ID from your AWS account):</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">aws kms get-public-key \
    --key-id <span style="color: #ff0000;"><em>&lt;1234abcd-12ab-34cd-56ef-1234567890ab&gt;</em></span> \
    --output text \
    --query PublicKey | base64 --decode &gt; public_key.der
</code></pre> 
</div> 
<p>To construct the external mu digest:</p> 
<ol> 
 <li>Construct a message prefix <code style="color: #000000;">(M`)</code>: <code style="color: #000000;">M` = (domain separator || context length || context || Message)</code>. <p>In this example, set the domain separator value and context length as zero; this sets the context used in the signature as the empty string, which is the default. </p></li> 
 <li>Hash the public key then prepend it to the message prefix:<br /> <code style="color: #000000;">(SHAKE256(pk) || M’)</code>.</li> 
 <li>Hash to produce a 64-byte mu:<br /> <code style="color: #000000;">Mu = SHAKE256(SHAKE256(pk) || M’)</code></li> 
</ol> 
<p>You can use a single OpenSSL 3.5 command to construct the digest:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">{
    openssl asn1parse -inform DER -in public_key.der -strparse 17 -noout -out - 2&gt;/dev/null |
    openssl dgst -provider default -shake256 -xoflen 64 -binary;
    printf '\x00\x00';
    echo -n "eyJ0eXAiOiJKV1QiLA0KICJhbGciOiJNTC1EU0EtNjUifQ.eyJpc3MiOiJqb2UiLA0KICJleHAiOjE3NDg5NTIwMDAsDQogImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ"
} | openssl dgst -provider default -shake256 -xoflen 64 -binary &gt; mu.bin
</code></pre> 
</div> 
<p>Now you can call AWS KMS to sign the 64-byte digest to produce the ML-DSA signature in file <code style="color: #000000;">ExampleSignature.bin</code>, making sure to set the <code style="color: #000000;">MessageType</code> to <code style="color: #000000;">EXTERNAL_MU</code>:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">aws kms sign \
    --key-id <span style="color: #ff0000;"><em>1234abcd-12ab-34cd-56ef-1234567890ab</em></span> \
    --message fileb://mu.bin \
    --message-type EXTERNAL_MU \
    --signing-algorithm ML_DSA_SHAKE_256 \
    --output text \
    --query Signature | base64 --decode &gt; ExampleSignature.bin
</code></pre> 
</div> 
<p>The final signed JWT token is identical to the one produced previously in RAW mode.</p> 
<h2>Signature verification using AWS KMS</h2> 
<p>In this section, we show you how to verify ML-DSA signatures using AWS KMS or locally in your own environment. We assume that you have an ML-DSA signature in <code style="color: #000000;">ExampleSignature.bin</code>, produced on the JWT content with the private key in AWS KMS and identified with <code style="color: #000000;">KEY_ARN</code>.</p> 
<p>Note that, although the following examples demonstrate signature verification using public keys directly from AWS KMS, these same principles extend to certificate-based systems, such as a private PKI, in which public keys are embedded in end-entity certificates (of the signer). In such scenarios, verifiers would first verify the identity of the signer by validating the certificate chain ties to a trusted root, then use the public key of the end-entity certificate to verify the ML-DSA signature of the content. The <a href="https://www.ietf.org/" rel="noopener noreferrer" target="_blank">IETF</a> is standardizing ML-DSA for use in X.509 certificates through RFC draft <a href="https://datatracker.ietf.org/doc/html/draft-ietf-lamps-dilithium-certificates" rel="noopener noreferrer" target="_blank">draft-ietf-lamps-dilithium-certificates</a>.</p> 
<h3>RAW ML-DSA verification</h3> 
<p>To verify the signature using AWS KMS, you can call the following command, replacing the example <code style="color: #000000;">key-id</code> with the same one you used to sign.</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">aws kms verify \
    --key-id <span style="color: #ff0000;"><em>&lt;1234abcd-12ab-34cd-56ef-1234567890ab&gt;</em></span> \
    --message "eyJ0eXAiOiJKV1QiLA0KICJhbGciOiJNTC1EU0EtNjUifQ.eyJpc3MiOiJqb2UiLA0KICJleHAiOjE3NDg5NTIwMDAsDQogImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ" \
    --message-type RAW \
    --signing-algorithm ML_DSA_SHAKE_256 \
    --signature fileb://ExampleSignature.bin
</code></pre> 
</div> 
<p>The response will return:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">{
    "KeyId": "<span style="color: #ff0000;"><em>arn:aws:kms:us-west-2:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab</em></span>",
    "SignatureValid": true,
    "SigningAlgorithm": "ML_DSA_SHAKE_256"
}
</code></pre> 
</div> 
<p>The verification result is stored in the <code style="color: #000000;">SignatureValid</code> field.</p> 
<h3>External mu ML-DSA verification</h3> 
<p>If you have the external mu digest of the JWT content in <code style="color: #000000;">mu.bin</code> along with the signature and the corresponding keypair in AWS KMS, you can use the digest without having access to the entire message or calculating the digest again.</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">aws kms verify \
    --key-id <span style="color: #ff0000;"><em>&lt;1234abcd-12ab-34cd-56ef-1234567890ab&gt;</em></span> \
    --message fileb://mu.bin \
    --message-type EXTERNAL_MU \
    --signing-algorithm ML_DSA_SHAKE_256 \
    --signature fileb://ExampleSignature.bin

</code></pre> 
</div> 
<p>To regenerate the external mu <code style="color: #000000;">mu.bin</code> from the message and the public key, see the <a href="#external-mu">External mu ML DSA signing section</a> above.</p> 
<h3>Local signature verification using OpenSSL 3.5</h3> 
<p>If you want to reduce AWS KMS API consumption costs and better control the use of API quotas while keeping the security of AWS KMS-generated and stored keys for ML-DSA signature generation, you can verify ML-DSA signatures locally, outside of AWS KMS. </p> 
<p>In this example, you use OpenSSL 3.5 to verify the signature in <code style="color: #000000;">ExampleSignature.bin</code>. You first must fetch the DER-encoded public key from AWS KMS in file <code style="color: #000000;">public_key.der</code> as shown in the <a href="#external-mu">External mu ML DSA signing</a> section. OpenSSL 3.5 can then verify the signature on the message by using the public key.</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">echo -n "eyJ0eXAiOiJKV1QiLA0KICJhbGciOiJNTC1EU0EtNjUifQ.eyJpc3MiOiJqb2UiLA0KICJleHAiOjE3NDg5NTIwMDAsDQogImh0dHA6Ly9leGFtcGxlLmNvbS9pc19yb290Ijp0cnVlfQ" | \
	openssl dgst -verify public_key.der -signature ExampleSignature.bin
</code></pre> 
</div> 
<p>Successful verification will output: <code style="color: #000000;">Verified OK</code></p> 
<h2>Conclusion</h2> 
<p>Today’s launch of ML-DSA support in AWS KMS marks an important milestone in our commitment to post-quantum cryptography. With three different security levels of ML-DSA in both raw and external digest modes, you have flexible options to meet your security requirements while preparing for the quantum computing era. The seamless integration with existing AWS KMS APIs makes it straightforward to incorporate quantum-resistant signatures into your applications today. This implementation is particularly valuable if you need to:</p> 
<ul> 
 <li>Meet FIPS 140-3 compliance requirements when using post-quantum cryptography.</li> 
 <li>Sign code, artifacts, documents or other data that need to remain trusted and verifiable for many years into the future, including the period after cryptographically relevant quantum computers exist.</li> 
 <li>Start post-quantum cryptography testing as part of your application development process using a cryptographic service such as AWS KMS that has previously been approved for use.</li> 
</ul> 
<p>Learn more about <a href="https://aws.amazon.com/security/post-quantum-cryptography/" rel="noopener noreferrer" target="_blank">post-quantum cryptography</a> in general and the overall AWS plan to <a href="https://aws.amazon.com/blogs/security/aws-post-quantum-cryptography-migration-plan/" rel="noopener noreferrer" target="_blank">migrate to post-quantum cryptography</a>.</p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Jake Massimo" class="aligncenter size-full wp-image-36679" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/06/09/jakemas.jpg" width="120" /> 
  </div> 
  <h3 class="lb-h4">Jake Massimo</h3> 
  <p>Jake is an Applied Scientist on the AWS Cryptography team, where he bridges Amazon with the global cryptographic community through active participation in international conferences, academic research, and standards organizations. His work focuses on advancing the adoption of post-quantum cryptographic technology at cloud scale. Currently, he leads the development of optimized and formally verified post-quantum algorithms within the AWS cryptographic library.</p> 
  <p></p>
 </div> 
</footer> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Panos Kampanakis" class="aligncenter size-full wp-image-26603" height="1600" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2022/07/21/Panos-Profile-pic.jpg" width="1142" /> 
  </div> 
  <h3 class="lb-h4">Panos Kampanakis</h3> 
  <p>Panos has extensive experience with cyber security, applied cryptography, security automation, and vulnerability management. In his professional career, he has trained and presented on various security topics at technical events for numerous years. He has co-authored cybersecurity publications and participated in various security standards bodies to provide common interoperable protocols and languages for security information sharing, cryptography, and PKI.</p> 
  <p></p>
 </div> 
</footer> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Mayank Ambaliya" class="aligncenter size-full wp-image-38695" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/06/09/Mayank-Ambaliya.jpg" width="120" /> 
  </div> 
  <h3 class="lb-h4">Mayank Ambaliya</h3> 
  <p>Mayank is a Software Development Manager at AWS Key Management Service (AWS KMS), where he leads development of AWS KMS cryptographic APIs and custom key stores. Mayank has experience developing customer facing cryptographic APIs and cryptographic SDKs for AWS CloudHSM. Recently, he has been working on post-quantum algorithm support in AWS KMS and adding new cryptographic APIs in AWS KMS.</p> 
  <p></p>
 </div> 
</footer>
