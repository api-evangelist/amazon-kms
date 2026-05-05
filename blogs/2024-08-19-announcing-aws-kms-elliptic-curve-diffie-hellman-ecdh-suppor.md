---
title: "Announcing AWS KMS Elliptic Curve Diffie-Hellman (ECDH) support"
url: "https://aws.amazon.com/blogs/security/announcing-aws-kms-elliptic-curve-diffie-hellman-ecdh-support/"
date: "Mon, 19 Aug 2024 14:16:02 +0000"
author: "Patrick Palmer"
feed_url: "https://aws.amazon.com/blogs/security/category/security-identity-compliance/aws-key-management-service/feed/"
---
<p>When using cryptography to protect data, protocol designers often prefer symmetric keys and algorithms for their speed and efficiency. However, when data is exchanged across an untrusted network such as the internet, it becomes difficult to ensure that <em>only</em> the exchanging parties can know the same key. Asymmetric key pairs and algorithms help to solve this problem by allowing a public key to be shared over an untrusted network. And by using a key agreement scheme, two parties can use each other’s public key in combination with their own private key to each derive the same shared secret.</p> 
<p>We’re excited to announce that <a href="https://aws.amazon.com/kms/" rel="noopener" target="_blank">AWS Key Management Service (AWS KMS)</a> now supports <a href="https://aws.amazon.com/about-aws/whats-new/2024/06/aws-kms-elliptic-curve-diffie-hellman-ecdh-key-agreement/" rel="noopener" target="_blank">Elliptic Curve Diffie-Hellman (ECDH) key agreement on elliptic curve (ECC) KMS keys</a>. You can use the new <code style="color: #000000;">DeriveSharedSecret</code> API action to enable two parties to establish a secure communication channel by using a derived shared secret.</p> 
<p>In this blog post we provide an overview of the new API action and explain how it can help you establish secure communications by exchanging only public keys to obtain a derived shared secret. We then show example commands to demonstrate how AWS KMS and OpenSSL can be used by two parties to derive a shared secret.</p> 
<p>With this new <code style="color: #000000;">DeriveSharedSecret</code> API action, customers can take an external party’s public key and, in combination with a private key that resides within AWS KMS, derive a shared secret which can be used to derive a symmetric encryption key with a key derivation function (KDF). Customers can then use this symmetric encryption key to encrypt data locally within their application.</p> 
<p>The same external party can combine their own related private key with the customer’s corresponding public key from AWS KMS to derive the same shared secret.</p> 
<p>Now that both parties have the same shared secret, they can generate a symmetric encryption key that can be used to encrypt and decrypt the data they exchange.</p> 
<p><code style="color: #000000;">DeriveSharedSecret</code> offers a simple and secure way for customers to use their private key from within their application, enabling new asymmetric cryptography use cases for keys protected by AWS KMS, such as elliptic curve integrated encryption scheme (ECIES) or end-to-end encryption (E2EE) schemes.</p> 
<h2>AWS KMS DeriveSharedSecret overview</h2> 
<p>The <a href="https://docs.aws.amazon.com/kms/latest/APIReference/API_DeriveSharedSecret.html" rel="noopener" target="_blank">AWS KMS API Reference documentation</a> covers the <code style="color: #000000;">DeriveSharedSecret</code> API action in more detail than we include in this post. We broadly describe how to interact with the API action, using the following steps:</p> 
<ol> 
 <li>Create an elliptic curve (ECC) KMS key, selecting that the key be used for <code style="color: #000000;">KEY_AGREEMENT</code> and choosing one of the supported key specs. You will not be able to modify existing ECC keys to be used for key agreement.</li> 
 <li>Have another party create an elliptic curve key that matches the key spec you defined for your KMS key.</li> 
 <li>Retrieve the public key associated with your KMS key by using the existing <code style="color: #000000;">GetPublicKey</code> API action.</li> 
 <li>Exchange public keys through a trusted means of exchange with the other party. Note that <code style="color: #000000;">DeriveSharedSecret</code> expects a base64-encoded DER-formatted public key.</li> 
 <li>Use the other party’s public key as an input, along with your specified <code style="color: #000000;">KEY_AGREEMENT</code> key. The only key agreement algorithm supported by AWS KMS at launch is ECDH.</li> 
 <li>The other party should use the public key retrieved from AWS KMS and the private key associated with their generated ECC key pair to derive a shared secret.</li> 
</ol> 
<p>The result of the preceding steps is that both parties have the same output without exchanging secret information. Only public keys were exchanged between the two parties. The output of <code style="color: #000000;">DeriveSharedSecret</code> is the raw shared secret. This shared secret is the multiplication of points on the elliptic curves and can result in many more bytes than are needed for an encryption key. We recommend that customers use a KDF, following the <a href="https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-56Ar3.pdf" rel="noopener" target="_blank">National Institute of Standards and Technology (NIST) SP800-56A Rev. 3 section 5.8</a> guidance, to derive encryption keys from this shared secret.</p> 
<p>For the purposes of this post, we will demonstrate the steps by using the <a href="https://aws.amazon.com/cli/" rel="noopener" target="_blank">AWS CLI</a> and OpenSSL command line. AWS has incorporated best practices for customers within the AWS Encryption SDK. You can find more details at <a href="https://docs.aws.amazon.com/encryption-sdk/latest/developer-guide/use-kms-ecdh-keyring.html" rel="noopener" target="_blank">AWS KMS ECDH keyrings</a>.</p> 
<h2>Example use case</h2> 
<p>An example use case where you might wish to use ECDH key agreement is for end-to-end encryption. Although protocols exist that provide a secure framework for secure communications (for example, within <a href="https://aws.amazon.com/wickr/" rel="noopener" target="_blank">AWS Wickr</a>), we will highlight the simplified high-level steps behind some of these protocols. In our example use case, Alice and Bob are both part of a messaging network. This network is managed by a centralized service, and this service must not be able to access Alice or Bob’s unencrypted messages.</p> 
<div class="wp-caption aligncenter" id="attachment_35492" style="width: 790px;">
 <img alt="Figure 1: High-level architecture for the service described in the example use case" class="size-full wp-image-35492" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/08/14/img1_v2.png" style="border: 1px solid #bebebe;" width="780" />
 <p class="wp-caption-text" id="caption-attachment-35492">Figure 1: High-level architecture for the service described in the example use case</p>
</div> 
<p>As shown in Figure 1, Alice and Bob each have an ECC key pair and participate in the secret derivation by using ECDH, through the following steps:</p> 
<ol> 
 <li>Alice registers her public key in the centralized key storage service. A detailed discussion of the key storage service is beyond the scope of this post.</li> 
 <li>Bob, an AWS KMS user, calls the AWS KMS <code style="color: #000000;">GetPublicKey</code> action to obtain the public key for the ECC KMS key pair.</li> 
 <li>Bob registers his public key in the same centralized key storage service.</li> 
 <li>Alice, who wants to exchange encrypted messages with Bob, retrieves Bob’s public key from the centralized key storage service.</li> 
 <li>Bob gets a notification that Alice wants to communicate with him, and he retrieves Alice’s public key from the centralized key storage service.</li> 
 <li>Using Bob’s public key and her private key, Alice derives a shared secret by using her cryptography provider.</li> 
 <li>Using Alice’s public key and his private key, Bob derives a shared secret by using <code style="color: #000000;">DeriveSharedSecret</code>.</li> 
 <li>Alice and Bob now have an identical shared secret. From this shared secret, she can create a symmetric encryption key by using a suitable KDF. The symmetric encryption key can be used to create ciphertext that can be sent to Bob.</li> 
</ol> 
<h2>Example use case walkthrough</h2> 
<p>You can use the following steps to create a KMS key for ECDH use and derive a shared secret by using AWS KMS. For our demonstration purposes, the user Alice (from our example use case) is using OpenSSL as the cryptography tool. We will show how the AWS KMS user Bob and OpenSSL user Alice can derive a shared secret by using each other’s public key.</p> 
<h3>General prerequisites</h3> 
<p>You must have the following prerequisites in place in order to implement the solution:</p> 
<ul> 
 <li><a href="https://aws.amazon.com/cli/" rel="noopener" target="_blank">AWS CLI</a> — The latest version is recommended. The example here uses aws-cli/2.15.40 and aws-cli/1.32.110.</li> 
 <li><a href="https://www.openssl.org/" rel="noopener" target="_blank">OpenSSL</a> — The example here uses OpenSSL 3.3.0.</li> 
 <li>Both parties (Alice and Bob, from our example use case) have an ECC key on the same curve. The steps in the next section, <strong>Key creation prerequisite</strong>, explain how these keys can be created.</li> 
</ul> 
<h4>Key creation prerequisite</h4> 
<p>Alice and Bob must use the same ECC curve during key creation. The <code style="color: #000000;">DeriveSharedSecret</code> API action supports curves ECC_NIST_P256, ECC_NIST_P384, and ECC_NIST_P521, which map to P-256, P-384, and P-521 respectively in OpenSSL. The curves that AWS KMS supports are the curves approved by the U.S. National Institute of Standards and Technology (NIST). Additionally, AWS KMS supports the SM2 key spec only in Amazon Web Services China Regions.</p> 
<p style="text-decoration: underline;">Bob creates an asymmetric KMS key for key agreement purposes</p> 
<p>Bob creates a key pair in AWS KMS by using the <a href="https://docs.aws.amazon.com/kms/latest/APIReference/API_CreateKey.html" rel="noopener" target="_blank">CreateKey</a> API action. In the following example, Bob creates an ECC key pair with <code style="color: #000000;">ECC_NIST_P256</code> for the <code style="color: #000000;">KeySpec</code> parameter and <code style="color: #000000;">KEY_AGREEMENT</code> for the <code style="color: #000000;">KeyUsage</code> parameter.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">aws kms create-key \
--key-spec ECC_NIST_P256 \
--key-usage KEY_AGREEMENT \
--description "Example ECDH key pair"</code></pre> 
</div> 
<p>The response looks something like this:</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">{
    "KeyMetadata": {
        "AWSAccountId": "111122223333",
        "KeyId": "a1b2c3d4-5678-90ab-cdef-EXAMPLE11111",
        "Arn": "arn:aws:kms:us-east-1:111122223333:key/a1b2c3d4-5678-90ab-cdef-EXAMPLE11111",
        "CreationDate": "2024-06-25T13:06:24.888000-07:00",
        "Enabled": true,
        "Description": "Example ECDH key pair",
        "KeyUsage": "KEY_AGREEMENT",
        "KeyState": "Enabled",
        "Origin": "AWS_KMS",
        "KeyManager": "CUSTOMER",
        "CustomerMasterKeySpec": "ECC_NIST_P256",
        "KeySpec": "ECC_NIST_P256",
        "KeyAgreementAlgorithms": [
            "ECDH"
        ],
        "MultiRegion": false
    }
}</code></pre> 
</div> 
<p>You can follow the <a href="https://docs.aws.amazon.com/kms/latest/developerguide/asymm-create-key.html" rel="noopener" target="_blank">Creating asymmetric KMS keys</a> documentation to see how to use the <a href="https://aws.amazon.com/console/" rel="noopener" target="_blank">AWS Management Console</a> to create a KMS key pair with the same properties as shown here. This example creates a KMS key with a default <a href="https://docs.aws.amazon.com/kms/latest/developerguide/key-policies.html" rel="noopener" target="_blank">KMS key policy</a>. You should review and configure your key policy according to the principle of least privilege, as appropriate for your environment.</p> 
<blockquote>
 <p><strong>Note:</strong> When a KMS key is created, it will be logged by <a href="https://aws.amazon.com/cloudtrail/" rel="noopener" target="_blank">AWS CloudTrail</a>, a service that monitors and records activity within your account. API calls to the AWS KMS service are <a href="https://docs.aws.amazon.com/kms/latest/developerguide/logging-using-cloudtrail.html" rel="noopener" target="_blank">logged in CloudTrail</a>, which you can use to audit access to KMS keys.</p>
</blockquote> 
<p>To allow your KMS key to be identified by a human-readable string rather than by the <code style="color: #000000;">KeyId</code> value, you can <a href="https://docs.aws.amazon.com/kms/latest/developerguide/kms-alias.html" rel="noopener" target="_blank">create an alias for the KMS key</a> (replace the <code style="color: #000000;">target-key-id</code> value of <code style="color: #000000;">a1b2c3d4-5678-90ab-cdef-EXAMPLE11111</code> with your <code style="color: #000000;">KeyId</code> value). This makes it easier to use and manage your KMS keys.</p> 
<p>Bob creates an alias for his KMS key by using the CLI with the following command:</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">aws kms create-alias \
    --alias-name alias/example-ecdh-key \
    --target-key-id a1b2c3d4-5678-90ab-cdef-EXAMPLE11111 </code></pre> 
</div> 
<p style="text-decoration: underline;">Alice creates an ECC key for key agreement purposes by using OpenSSL</p> 
<p>Using the ecparam and genkey option of OpenSSL, Alice creates a P-256 ECC key. The P-256 curve is represented by AWS KMS as ECC_NIST_P256.</p> 
<blockquote>
 <p><strong>Note:</strong> For ECDH to work, the curve of the OpenSSL ECC key must be same as the ECC KMS key created by the other party (Bob, in our example use case).</p>
</blockquote> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">openssl ecparam -name P-256 \
        -genkey -out openssl_ecc_private_key.pem</code></pre> 
</div> 
<h3>Key exchange and secret derivation process</h3> 
<p>The following sections outline the steps that Alice and Bob will follow to share their public keys, retrieve one another’s public key, and then derive the same shared secret using AWS KMS and OpenSSL. The shared secrets derived by Alice and Bob respectively are then compared to show that they both derived the same shared secret.</p> 
<h4>Step 1: Alice generates and registers her OpenSSL public key with a central service</h4> 
<p>AWS KMS expects the public key in DER format. Therefore, in this example Alice creates a DER-format public key by using her ECC private key. Alice runs the following command to produce a DER-format file that contains her public key:</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">openssl ec -in openssl_ecc_private_key.pem \
        -pubout -outform DER \
        &gt; openssl_ecc_public_key.bin.der</code></pre> 
</div> 
<p>The file <code style="color: #000000;">openssl_ecc_public_key.bin.der</code> will have the public key in DER format, which Alice can store in the centralized key storage service (or send to anyone she would like to communicate with). Details about the centralized key storage service are beyond the scope of this post.</p> 
<h4>Step 2: Bob obtains the public key for his ECC KMS Key</h4> 
<p>To retrieve a copy of the public key for his ECC KMS key, Bob uses the <a href="https://docs.aws.amazon.com/kms/latest/APIReference/API_GetPublicKey.html" rel="noopener" target="_blank">GetPublicKey</a> API action. Bob calls this API by using the AWS CLI command <a href="https://docs.aws.amazon.com/cli/latest/reference/kms/get-public-key.html" rel="noopener" target="_blank">get-public-key</a>, as follows:</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">aws kms get-public-key \
    --key-id alias/example-ecdh-key \
    --output text \
    --query PublicKey | base64 --decode &gt; kms_ecdh_public_key.der</code></pre> 
</div> 
<p>The returned <code style="color: #000000;">PublicKey</code> value is a DER-encoded X.509 public key. Because the AWS CLI is being used, the public key output is base64-encoded for readability purposes. This base64-encoded value is decoded by using the base64 command, and the decoded value is stored in the output file. The file <code style="color: #000000;">kms_ecdh_public_key.der</code> contains the DER-encoded public key.</p> 
<blockquote>
 <p><strong>Note:</strong> If you call this API by using one of the AWS SDKs, such as <a href="https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/kms.html#KMS.Client.get_public_key" rel="noopener" target="_blank">Boto3</a>, then the returned <code style="color: #000000;">PublicKey</code> value is not base64-encoded.</p>
</blockquote> 
<p>In our example use case, Alice is using OpenSSL, which expects the public key in PEM format. Bob converts his DER-format public key into PEM format by using the following command:</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">openssl ec -pubin -inform DER -outform PEM \
        -in kms_ecdh_public_key.der \
        -out kms_ecdh_public_key.pem</code></pre> 
</div> 
<p>The file <code style="color: #000000;">kms_ecdh_public_key.pem</code> contains the public key in PEM format.</p> 
<h4>Step 3: Bob registers his public key with the centralized key storage service</h4> 
<p>Bob saves his public key in PEM format, obtained in Step 2, in the centralized key storage service.</p> 
<h4>Step 4: Alice retrieves Bob’s public key to derive a shared secret</h4> 
<p>To perform ECDH key agreement, the two parties involved (Alice and Bob, in our example use case) need to exchange their public key with each other. Alice, who wants to send encrypted messages to Bob, retrieves Bob’s public key from the centralized key storage service.</p> 
<p>Bob’s public key, <code style="color: #000000;">kms_ecdh_public_key.pem</code>, is already in PEM format as expected by OpenSSL.</p> 
<h4>Step 5: Bob retrieves Alice’s public key to derive a shared secret</h4> 
<p>To perform ECDH key agreement, the two parties involved, Alice and Bob, need to exchange their public key with each other. Bob gets a notification that Alice wants to communicate with him, and he retrieves Alice’s public key from the centralized key storage service.</p> 
<p>Alice’s public key, <code style="color: #000000;">openssl_ecc_public_key.bin.der</code>, is already in DER format as expected by AWS KMS.</p> 
<h4>Step 6: Alice uses OpenSSL to derive the shared secret</h4> 
<p>Alice, using her private key and Bob’s public key, can derive the shared secret by using OpenSSL. Alice derives the shared secret by using the OpenSSL pkeyutl command with the derive option, as follows:</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">openssl pkeyutl -derive \
-inkey openssl_ecc_private_key.pem \
-peerkey kms_ecdh_public_key.pem &gt; openssl.ss</code></pre> 
</div> 
<p>The file <code style="color: #000000;">openssl.ss</code> will have the shared secret in binary format.</p> 
<h4>Step 7: Bob uses AWS KMS to derive the shared secret</h4> 
<p>Bob, using his private key (which remains securely within AWS KMS) and Alice’s public key, can derive the shared secret by using AWS KMS. The following example shows how Bob uses the <a href="https://docs.aws.amazon.com/kms/latest/APIReference/API_DeriveSharedSecret.html" rel="noopener" target="_blank">DeriveSharedSecret</a> API action with the AWS CLI command <code style="color: #000000;">derive-shared-secret</code>. At launch, the only supported key agreement algorithm is ECDH. Bob passes Alice’s public key for the <code style="color: #000000;">PublicKey</code> parameter.</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">aws kms derive-shared-secret \
--key-id alias/example-ecdh-key \
--public-key fileb://path/to/openssl_ecc_public_key.bin.der \
--key-agreement-algorithm ECDH \
--output text --query SharedSecret |base64 --decode &gt; kms.ss</code></pre> 
</div> 
<p>Because the AWS CLI is being used, the returned <code style="color: #000000;">SharedSecret</code> value is base64-encoded for readability purposes. Using the <code style="color: #000000;">base64 --decode</code> command, the decoded binary format is stored to the file.</p> 
<blockquote>
 <p><strong>Note:</strong> If you call this API by using one of the AWS SDKs, such as <a href="https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/kms.html#KMS.Client.get_public_key" rel="noopener" target="_blank">Boto3</a>, then the returned <code style="color: #000000;">SharedSecret</code> value is not base64-encoded.</p>
</blockquote> 
<p>The file <code style="color: #000000;">kms.ss</code> will have the shared secret in binary format.</p> 
<h4>Step 8: Using the shared secret and a suitable KDF, Alice derives an encryption key to encrypt her communication to Bob</h4> 
<p>You can use the following command to compare the two files containing the derived shared secrets that were obtained in Steps 6 and 7 and verify that they are identical:</p> 
<div class="hide-language"> 
 <pre class="unlimited-height-code"><code class="lang-text">diff -qs openssl.ss kms.ss</code></pre> 
</div> 
<p>Because these files are identical, we can see that the same secret was derived using both AWS KMS and OpenSSL.</p> 
<p>Using the shared secret, Alice should then derive a symmetric encryption key by using a suitable KDF. She can use this symmetric encryption key to encrypt data and send the ciphertext to Bob.</p> 
<p>This blog post does not cover the steps to derive that symmetric encryption key, because that can be a complex topic depending on your use case. However, we note that you should not use the raw shared secret as an encryption key because it is not uniform. In other words, the shared secret has a lot of entropy, but the byte string itself is not random.</p> 
<p>NIST recommends that you use a KDF function over the raw shared secret (value Z as described in section 5.8 of <a href="https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-56Ar3.pdf" rel="noopener" target="_blank">NIST SP800-56A Rev. 3</a>). The KDFs that are recommended are described in more detail in <a href="https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-56Cr2.pdf" rel="noopener" target="_blank">NIST SP800-56C Rev. 2</a>. One such example is OpenSSL Single Step KDF (SSKDF) <a href="https://www.openssl.org/docs/man3.3/man7/EVP_KDF-SS.html" rel="noopener" target="_blank">EVP_KDF-SS</a>, but using this KDF involves choosing the other values, such as <code style="color: #000000;">FixedInfo</code>, carefully.</p> 
<p>To help customers make the right choice for the resulting KDF to use on the shared secret, the AWS Encryption SDK now includes <a href="https://docs.aws.amazon.com/encryption-sdk/latest/developer-guide/use-kms-ecdh-keyring.html" rel="noopener" target="_blank">AWS KMS ECDH keyrings</a>. The keyring is a construct within the AWS Encryption SDK that you implement within your code. The keyring handles the management of encryption keys while applying best practices to protect your data. You can use the keyring to reference your KMS keys for key agreement, and then call a function to encrypt data. Data will be encrypted by using a derived shared wrapping key following NIST recommendations, and the Encryption SDK applies <a href="https://docs.aws.amazon.com/encryption-sdk/latest/developer-guide/concepts.html#key-commitment" rel="noopener" target="_blank">key commitment</a> to the ciphertext.</p> 
<h2>Summary</h2> 
<p>In this blog post, we highlighted how you can use the recently launched <code style="color: #000000;">DeriveSharedSecret</code> API action to securely derive a shared secret. You’ve seen how ECDH can be used between two parties without having to share secret information across untrusted networks. We explained how you can audit your AWS KMS key usage through AWS CloudTrail logs. We highlighted that you would need to use a KDF to generate a symmetric encryption key from the shared secret. We strongly recommend that you use the AWS Encryption SDK to encrypt your data, which helps make sure that the recommended NIST key derivation functions are used for generating symmetric encryption keys.</p> 
<p>&nbsp;<br />If you have feedback about this post, submit comments in the<strong> Comments</strong> section below. If you have questions about this post, <a href="https://console.aws.amazon.com/support/home" rel="noopener noreferrer" target="_blank">contact AWS Support</a>.</p> 
<footer> 
 <div class="blog-author-box">
  <img alt="Patrick Palmer" class="alignleft size-full wp-image-33350" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2019/01/21/patrick-palmer-author-bio.jpg" style="margin-left: 12px; margin-right: 18px; margin-top: 12px; margin-bottom: 6px; width: 93.750px; height: 125px;" />
  <p></p> 
  <p><span class="lb-h4" style="line-height: 0.5; padding-top: 0px;">Patrick Palmer</span><br />Patrick is a Principal Security Specialist Solutions Architect at AWS. He helps customers around the world use AWS services in a secure manner and specializes in cryptography. When not working, he enjoys spending time with his growing family and playing video games.</p> 
 </div> 
 <div class="blog-author-box">
  <img alt="Raj Puttaiah" class="alignleft size-full wp-image-33350" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/08/12/rajherur.jpg" style="margin-left: 12px; margin-right: 18px; margin-top: 12px; margin-bottom: 6px; width: 93.750px; height: 125px;" />
  <p></p> 
  <p><span class="lb-h4" style="line-height: 0.5; padding-top: 0px;">Raj Puttaiah</span><br />Raj is a Software Development Manager for AWS KMS. Raj leads the development of AWS KMS features, focusing on operational excellence. When not working, Raj spends time with his family hiking the beautiful Washington outdoors, and accompanying his two sons to their activities.</p> 
 </div> 
 <div class="blog-author-box">
  <img alt="Michael Miller" class="alignleft size-full wp-image-33350" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2024/08/14/millerdq.jpg" style="margin-left: 12px; margin-right: 18px; margin-top: 12px; margin-bottom: 6px; width: 93.750px; height: 125px;" />
  <p></p> 
  <p><span class="lb-h4" style="line-height: 0.5; padding-top: 0px;">Michael Miller</span><br />Michael is a Senior Solutions Architect at AWS, based in Ireland. He helps public sector customers across the UK and Ireland accelerate their cloud adoption journey and specializes in security and networking. In prior roles, Michael has been responsible for designing architectures and supporting implementations across various sectors including service providers, consultancies, and financial services organizations.</p> 
 </div> 
</footer>
