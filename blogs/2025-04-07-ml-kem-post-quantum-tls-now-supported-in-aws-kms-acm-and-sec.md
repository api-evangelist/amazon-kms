---
title: "ML-KEM post-quantum TLS now supported in AWS KMS, ACM, and Secrets Manager"
url: "https://aws.amazon.com/blogs/security/ml-kem-post-quantum-tls-now-supported-in-aws-kms-acm-and-secrets-manager/"
date: "Mon, 07 Apr 2025 18:55:02 +0000"
author: "Alex Weibel"
feed_url: "https://aws.amazon.com/blogs/security/category/security-identity-compliance/aws-key-management-service/feed/"
---
<p><a href="https://aws.amazon.com/" rel="noopener" target="_blank">Amazon Web Services (AWS)</a> is excited to announce that <a href="https://datatracker.ietf.org/doc/draft-kwiatkowski-tls-ecdhe-mlkem/" rel="noopener" target="_blank">the latest hybrid post-quantum key agreement standards</a> for TLS have been deployed to three AWS services. Today, <a href="http://aws.amazon.com/kms" rel="noopener" target="_blank">AWS Key Management Service (AWS KMS)</a>, <a href="https://aws.amazon.com/acm/" rel="noopener" target="_blank">AWS Certificate Manager (ACM)</a>, and <a href="https://aws.amazon.com/secrets-manager/" rel="noopener" target="_blank">AWS Secrets Manager</a> endpoints now support <a href="https://doi.org/10.6028/NIST.FIPS.203" rel="noopener" target="_blank">Module-Lattice-Based Key-Encapsulation Mechanism (ML-KEM)</a> for hybrid post-quantum key agreement in non-FIPS endpoints in all AWS Regions in the <code style="color: #000000;">aws</code> partition. The <a href="https://github.com/aws/aws-secretsmanager-agent" rel="noopener" target="_blank">AWS Secrets Manager Agent</a>, built on <a href="https://github.com/awslabs/aws-sdk-rust" rel="noopener" target="_blank">AWS SDK for Rust</a> now also has <a href="https://github.com/awslabs/aws-sdk-rust/discussions/1257" rel="noopener" target="_blank">opt-in support for hybrid post-quantum key agreement</a>. With this, customers can bring secrets into their applications with end-to-end post-quantum enabled TLS.</p> 
<p>These three services were chosen because they are security-critical AWS services with the most urgent need for post-quantum confidentiality. These three AWS services have <a href="https://aws.amazon.com/about-aws/whats-new/2022/03/aws-kms-acm-support-latest-hybrid-post-quantum-tls-ciphers/" rel="noopener" target="_blank">previously deployed support for CRYSTALS-Kyber</a>, the predecessor of ML-KEM. Support for CRYSTALS-Kyber will continue through 2025, but will be removed across all AWS service endpoints in 2026 in favor of ML-KEM.</p> 
<h2 id="our-migration-to-post-quantum-cryptography">Our migration to post-quantum cryptography</h2> 
<p>AWS is committed to following our <a href="https://aws.amazon.com/blogs/security/aws-post-quantum-cryptography-migration-plan/" rel="noopener" target="_blank">post-quantum cryptography migration plan</a>. As part of this commitment, and part of the <a href="https://aws.amazon.com/blogs/security/customer-compliance-and-security-during-the-post-quantum-cryptographic-migration/" rel="noopener" target="_blank">AWS post-quantum shared responsibility model</a>, AWS plans to deploy support for ML-KEM to all AWS services with HTTPS endpoints over the coming years. AWS customers must update their TLS clients and SDKs to offer ML-KEM when connecting to AWS service HTTPS endpoints. This will protect against future <em>harvest now, decrypt later</em> threats posed by quantum computing advancements. Meanwhile, AWS service HTTPS endpoints will be responsible for selecting ML-KEM when offered by clients.</p> 
<p>Our commitment to negotiate hybrid post-quantum key agreement algorithms is enabled by <a href="https://github.com/awslabs/aws-lc" rel="noopener" target="_blank">AWS Libcrypto</a> (AWS-LC), <a href="https://csrc.nist.gov/projects/cryptographic-algorithm-validation-program/validation-search?searchMode=implementation&amp;vendor=Amazon&amp;productType=-1&amp;algorithm=180&amp;ipp=25" rel="noopener" target="_blank">our open-source FIPS-140-3-validated cryptographic library</a> used throughout AWS, and <a href="https://github.com/aws/s2n-tls" rel="noopener" target="_blank">s2n-tls</a>, our open-source TLS implementation used across AWS service HTTPS endpoints. AWS-LC has been awarded multiple FIPS certificates from NIST (<a href="https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/4631" rel="noopener" target="_blank">#4631</a>, <a href="https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/4759" rel="noopener" target="_blank">#4759</a>, and <a href="https://csrc.nist.gov/projects/cryptographic-module-validation-program/certificate/4816" rel="noopener" target="_blank">#4816</a>), and was <a href="https://aws.amazon.com/blogs/security/aws-lc-fips-3-0-first-cryptographic-library-to-include-ml-kem-in-fips-140-3-validation/" rel="noopener" target="_blank">the first open-source cryptographic module to include ML-KEM in a FIPS 140-3 validation.</a></p> 
<h2 id="the-effect-of-hybrid-post-quantum-ml-kem-on-tls-performance">The effect of hybrid post-quantum ML-KEM on TLS performance</h2> 
<p>Migrating from an Elliptic Curve Diffie-Hellman (ECDH)-only key agreement to an ECDH+ML-KEM hybrid key agreement necessarily requires that the TLS handshake send more data and perform more cryptographic operations. Switching from a classical to a hybrid post-quantum key agreement will transfer approximately 1600 additional bytes during the TLS handshake and will require approximately 80–150 microseconds more compute time to perform ML-KEM cryptographic operations. This is a one-time TLS connection startup cost and is amortized over the lifetime of the TLS connection across the HTTP requests sent over that connection.</p> 
<p>AWS is working to provide a smooth migration to hybrid post-quantum key agreement for TLS. This work includes performing benchmarks on example workloads to help customers understand the impact of enabling hybrid post-quantum key agreement with ML-KEM.</p> 
<p>Using the <a href="https://github.com/aws/aws-sdk-java-v2" rel="noopener" target="_blank">AWS SDK for Java v2</a>, AWS has measured the number of AWS KMS <code style="color: #000000;">GenerateDataKey</code> requests per second that a single thread can issue serially between an <a href="http://aws.amazon.com/ec2" rel="noopener" target="_blank">Amazon Elastic Compute Cloud (Amazon EC2)</a> C6in.metal client and the public AWS KMS endpoint. Both the client and server were in the us-west-2 Region. Classical TLS connections to AWS KMS negotiated the P256 elliptic curve for key agreement, and hybrid post-quantum TLS connections negotiated the X25519 elliptic curve with ML-KEM-768 for their hybrid key agreement. Your own performance characteristics might differ and will depend on your environment, including your instance type, your workload profiles, the amount of parallelism and number of threads used, and your network location and capacity. The HTTP request transaction rates were measured with TLS connection reuse both enabled and disabled.</p> 
<p>Figure 1 shows the number of requests per second issued at different percentiles when TLS 1.3 connection reuse is disabled. It shows that in the worst-case scenario—when the cost of a TLS handshake is never amortized and every HTTP request must perform a full TLS handshake—enabling hybrid post-quantum TLS decreases the transactions per second (TPS) by about 2.3 percent on average, from 108.7 TPS to 106.2 TPS.</p> 
<div class="wp-caption aligncenter" id="attachment_37873" style="width: 1757px;">
 <img alt="Figure 1: AWS KMS GenerateDataKey requests per second &lt;em&gt;without&lt;/em&gt; TLS connection reuse" class="size-full wp-image-37873" height="1221" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/04/02/ML-KEM-post-quantum-TLS-1.png" style="border: 1px solid #bebebe;" width="1747" />
 <p class="wp-caption-text" id="caption-attachment-37873">Figure 1: AWS KMS GenerateDataKey requests per second <em>without</em> TLS connection reuse</p>
</div> 
<p>Figure 2 shows the number of requests per second issued at different percentiles when TLS connection reuse is enabled. Reusing TLS connections and amortizing the cost of a TLS handshake over many HTTP requests is the default setting in the AWS SDK for Java v2. We show that enabling hybrid post-quantum TLS when using default SDK settings leaves the TPS rate almost unchanged, with only a 0.05 percent decrease on average, from 216.1 TPS to 216.0 TPS.</p> 
<div class="wp-caption aligncenter" id="attachment_37874" style="width: 1757px;">
 <img alt="Figure 2: AWS KMS GenerateDataKey requests per second &lt;em&gt;with&lt;/em&gt; TLS connection reuse" class="size-full wp-image-37874" height="1226" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/04/02/ML-KEM-post-quantum-TLS-2.png" style="border: 1px solid #bebebe;" width="1747" />
 <p class="wp-caption-text" id="caption-attachment-37874">Figure 2: AWS KMS GenerateDataKey requests per second <em>with</em> TLS connection reuse<br /></p>
</div> 
<p>Our results show that the performance impact of enabling hybrid post-quantum TLS is negligible when using typical configuration settings in your SDK. Our measurements show that enabling hybrid post-quantum TLS for a default-case example workload only lowered maximum TPS rate by 0.05 percent. Our results also show that overriding SDK defaults to force the worst-case scenario of performing a new TLS handshake for every request only decreased maximum TPS rate by 2.3 percent.</p> 
<p>The following table shows the benchmark data that we measured. Each benchmark performed 500 one-second TPS measurements for varying TLS key agreement settings and TLS connection reuse settings. The measurements used <code style="color: #000000;">v2.30.22</code> of the AWS SDK for Java v2. The TLS key agreement was switched between classical and hybrid post-quantum by toggling the <a href="https://sdk.amazonaws.com/java/api/latest/software/amazon/awssdk/http/crt/AwsCrtHttpClient.Builder.html#postQuantumTlsEnabled(java.lang.Boolean)" rel="noopener" target="_blank">postQuantumTlsEnabled()</a> configuration. TLS connection reuse was toggled by injecting a <a href="https://datatracker.ietf.org/doc/html/rfc2616#section-14.10" rel="noopener" target="_blank">Connection: close</a> HTTP header into <a href="https://sdk.amazonaws.com/java/api/latest/software/amazon/awssdk/core/RequestOverrideConfiguration.Builder.html#putHeader(java.lang.String,java.util.List)" rel="noopener" target="_blank">each HTTP request</a>. This header forces the TLS connection to be shut down after each HTTP request and requires that a new TLS connection be created for each HTTP request.</p> 
<table border="1" width="0"> 
 <colgroup> 
  <col style="width: 22%;" /> 
  <col style="width: 8%;" /> 
  <col style="width: 10%;" /> 
  <col style="width: 8%;" /> 
  <col style="width: 5%;" /> 
  <col style="width: 5%;" /> 
  <col style="width: 5%;" /> 
  <col style="width: 5%;" /> 
  <col style="width: 5%;" /> 
  <col style="width: 5%;" /> 
  <col style="width: 5%;" /> 
 </colgroup> 
 <thead> 
  <tr> 
   <th><strong>TLS key agreement</strong></th> 
   <th><strong>TLS conn resuse</strong></th> 
   <th style="text-align: right;"><strong>Total HTTP requests</strong></th> 
   <th style="text-align: right;"><strong>Average (TPS)</strong></th> 
   <th style="text-align: right;"><strong>p01 (TPS)</strong></th> 
   <th style="text-align: right;"><strong>p10 (TPS)</strong></th> 
   <th style="text-align: right;"><strong>p25 (TPS)</strong></th> 
   <th style="text-align: right;"><strong>p50 (TPS)</strong></th> 
   <th style="text-align: right;"><strong>p75 (TPS)</strong></th> 
   <th style="text-align: right;"><strong>p90 (TPS)</strong></th> 
   <th style="text-align: right;"><strong>p99 (TPS)</strong></th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td>Classical (P256)</td> 
   <td>No</td> 
   <td style="text-align: right;">54,367</td> 
   <td style="text-align: right;">108.7</td> 
   <td style="text-align: right;">78</td> 
   <td style="text-align: right;">86</td> 
   <td style="text-align: right;">96</td> 
   <td style="text-align: right;">102</td> 
   <td style="text-align: right;">129</td> 
   <td style="text-align: right;">137</td> 
   <td style="text-align: right;">145</td> 
  </tr> 
  <tr> 
   <td>Hybrid post-quantum (X25519MLKEM768)</td> 
   <td>No</td> 
   <td style="text-align: right;">53,106</td> 
   <td style="text-align: right;">106.2</td> 
   <td style="text-align: right;">76</td> 
   <td style="text-align: right;">85</td> 
   <td style="text-align: right;">93</td> 
   <td style="text-align: right;">100</td> 
   <td style="text-align: right;">126</td> 
   <td style="text-align: right;">134</td> 
   <td style="text-align: right;">141</td> 
  </tr> 
  <tr> 
   <td>Classical (P256)</td> 
   <td>Yes</td> 
   <td style="text-align: right;">108,052</td> 
   <td style="text-align: right;">216.1</td> 
   <td style="text-align: right;">181</td> 
   <td style="text-align: right;">194</td> 
   <td style="text-align: right;">200</td> 
   <td style="text-align: right;">216</td> 
   <td style="text-align: right;">233</td> 
   <td style="text-align: right;">240</td> 
   <td style="text-align: right;">245</td> 
  </tr> 
  <tr> 
   <td>Hybrid post-quantum (X25519MLKEM768)</td> 
   <td>Yes</td> 
   <td style="text-align: right;">107,994</td> 
   <td style="text-align: right;">216</td> 
   <td style="text-align: right;">177</td> 
   <td style="text-align: right;">194</td> 
   <td style="text-align: right;">200</td> 
   <td style="text-align: right;">216</td> 
   <td style="text-align: right;">233</td> 
   <td style="text-align: right;">239</td> 
   <td style="text-align: right;">245</td> 
  </tr> 
 </tbody> 
</table> 
<h2 id="removing-support-for-draft-post-quantum-standards">Removing support for draft post-quantum standards</h2> 
<p>AWS service endpoints with support for CRYSTALS-Kyber, the predecessor of ML-KEM, will continue to support CRYSTALS-Kyber through 2025. We will slowly phase out support for the pre-standard CRYSTALS-Kyber implementations after customers have moved to the ML-KEM standard. Customers using previous versions of the AWS SDK for Java with CRYSTALS-Kyber support should upgrade to the latest SDK versions that have ML-KEM support. No code changes are necessary for customers using a generally available release of the AWS SDK for Java v2 to upgrade from CRYSTALS-Kyber to ML-KEM.</p> 
<p>Customers currently negotiating CRYSTALS-Kyber who do not upgrade their AWS Java SDK v2 clients by 2026 will see their clients gracefully fall back to a classical key agreement once CRYSTALS-Kyber is removed from AWS service HTTPS endpoints.</p> 
<h2 id="how-to-use-hybrid-post-quantum-key-agreement">How to use hybrid post-quantum key agreement</h2> 
<p>If using the <a href="https://github.com/awslabs/aws-sdk-rust" rel="noopener" target="_blank">AWS SDK for Rust</a>, you can enable the hybrid post-quantum key agreement by adding the rustls package to your crate and enabling the <code style="color: #000000;">prefer-post-quantum</code> feature flag. See the rustls <a href="https://docs.rs/rustls/0.23.23/rustls/manual/_05_defaults/index.html#about-the-post-quantum-secure-key-exchange-x25519mlkem768" rel="noopener" target="_blank">documentation</a> for more information.</p> 
<p>If using the <a href="https://github.com/aws/aws-sdk-java-v2" rel="noopener" target="_blank">AWS SDK for Java 2.x</a>, you can enable hybrid post-quantum key agreement by calling <code style="color: #000000;">.postQuantumTlsEnabled(true)</code> when building your AWS Common Runtime HTTP client.</p> 
<h3 id="step-1-add-the-aws-common-runtime-http-client-to-your-java-dependencies.">Step 1: Add the AWS Common Runtime HTTP client to your Java dependencies.</h3> 
<p>Add the AWS Common Runtime HTTP client to your Maven dependencies. We recommend using the latest available version. Use version 2.30.22 or greater to enable the use of ML-KEM.</p> 
<div> 
 <pre><code class="lang-java">&lt;dependency&gt;
    &lt;groupId&gt;software.amazon.awssdk&lt;/groupId&gt;
    &lt;artifactId&gt;aws-crt-client&lt;/artifactId&gt;
    &lt;version&gt;2.30.22&lt;version&gt;
&lt;/dependency&gt;

</code></pre> 
</div> 
<h3 id="step-2-enable-post-quantum-tls-in-your-java-sdk-client-configuration">Step 2: Enable post-quantum TLS in your Java SDK client configuration</h3> 
<p>When configuring your AWS service client, use the AwsCrtAsyncHttpClient configured with post-quantum TLS.</p> 
<div> 
 <pre><code class="lang-java">// Configure an AWS Common Runtime HTTP client with Post-Quantum TLS enabled
SdkAsyncHttpClient awsCrtHttpClient = AwsCrtAsyncHttpClient.builder()
          .postQuantumTlsEnabled(true)
          .build();

// Create an AWS service client that uses the AWS Common Runtime client
KmsAsyncClient kmsAsync = KmsAsyncClient.builder()
         .httpClient(awsCrtHttpClient)
         .build();

// Make a request over a TLS connection that uses post-quantum key agreement
ListKeysReponse keys = kmsAsync.listKeys().get();
</code></pre> 
</div> 
<p>See the <a href="https://github.com/aws-samples/aws-kms-pq-tls-example" rel="noopener" target="_blank"><u>KMS PQ TLS example application</u></a> for an end-to-end example of a post-quantum TLS setup.</p> 
<h2 id="things-to-try">Things to try</h2> 
<p>Here are some ideas about how to use this post-quantum-enabled client:</p> 
<ul> 
 <li><strong>Run load tests and benchmarks.</strong> The AwsCrtAsyncHttpClient is heavily optimized for performance and uses AWS Libcrypto on Linux-based environments. If you aren’t already using the AwsCrtAsyncHttpClient, try it today to see the performance benefits compared to the default SDK HTTP client. After using AwsCrtAsyncHttpClient, enable post-quantum TLS support. See if using AwsCrtAsyncHttpClient with post-quantum TLS is an overall performance gain to using the default SDK HTTP client without post-quantum TLS.</li> 
 <li><strong>Try connecting from different network locations.</strong> Depending on the network path that your request takes, you might discover that intermediate hosts, proxies, or firewalls with <a href="https://en.wikipedia.org/wiki/Deep_packet_inspection" rel="noopener" target="_blank">deep packet inspection (DPI)</a> block the request. If this is the case, you might need to work with your security team or IT administrators to update firewalls in your network <a href="https://tldr.fail/" rel="noopener" target="_blank">to unblock these new TLS algorithms</a>. We want to <a href="mailto:post-quantum-aws@amazon.com" rel="noopener" target="_blank">hear from you</a> about how your infrastructure interacts with this new variant of TLS traffic.</li> 
</ul> 
<h2 id="conclusion">Conclusion</h2> 
<p>Support for ML-KEM-based hybrid key agreement has been deployed to three security-critical AWS service endpoints. The performance impact of enabling hybrid post-quantum TLS is likely to be negligible when TLS connection reuse is enabled. Our measurements showed only a 0.05 percent decrease to maximum transactions per second when calling AWS KMS <code style="color: #000000;">GenerateDataKey</code>.</p> 
<p>Starting with version <code style="color: #000000;">2.30.22</code>, the AWS SDK for Java v2 now supports ML-KEM-based hybrid key agreement on Linux-based platforms when using the AWS Common Runtime HTTP client. Try <a href="https://sdk.amazonaws.com/java/api/latest/software/amazon/awssdk/http/crt/AwsCrtHttpClient.Builder.html#postQuantumTlsEnabled(java.lang.Boolean)" rel="noopener" target="_blank">enabling post quantum key agreement for TLS</a> in your Java SDK client configuration today.</p> 
<p>AWS plans to deploy support for ML-KEM-based hybrid post-quantum key agreement to every AWS service HTTPS endpoint over the coming years as part of our <a href="https://aws.amazon.com/blogs/security/aws-post-quantum-cryptography-migration-plan/" rel="noopener" target="_blank">post-quantum cryptography migration plan</a>. <a href="https://aws.amazon.com/blogs/security/customer-compliance-and-security-during-the-post-quantum-cryptographic-migration/" rel="noopener" target="_blank">AWS customers will be responsible</a> for updating their TLS clients and SDKs to help ensure that ML-KEM key agreement is offered when connecting to AWS service HTTPS endpoints. This will protect against future <em>harvest now, decrypt later</em> threats posed by quantum computing advancements.</p> 
<p>For additional information, blog posts, and periodic updates on our post-quantum cryptography migration, keep watching the <a href="https://aws.amazon.com/security/post-quantum-cryptography/" rel="noopener" target="_blank">AWS Post-Quantum Cryptography page</a>. To learn more about post-quantum cryptography with AWS, contact the <a href="mailto:post-quantum-aws@amazon.com" rel="noopener" target="_blank">post-quantum cryptography team</a>.</p> 
<p>If you have feedback about this post, submit comments in the Comments section below. If you have questions about this post, start a new thread on the <a href="https://repost.aws/topics/TAEEfW2o7QS4SOLeZqACq9jA/security-identity-compliance%3Fsc_ichannel=ha&amp;sc_ilang=en&amp;sc_isite=repost&amp;sc_iplace=hp&amp;sc_icontent=TAEEfW2o7QS4SOLeZqACq9jA&amp;sc_ipos=0" rel="noopener" target="_blank">AWS Security, Identity, &amp; Compliance re:Post</a> or <a href="https://console.aws.amazon.com/support/home" rel="noopener" target="_blank">contact AWS Support</a>.</p> 
<p>Additional resources:</p> 
<ul> 
 <li><a href="https://aws.amazon.com/security/post-quantum-cryptography/" rel="noopener" target="_blank"><u>AWS Post-Quantum Cryptography</u></a></li> 
 <li><a href="https://aws.amazon.com/blogs/security/aws-post-quantum-cryptography-migration-plan/" rel="noopener" target="_blank"><u>AWS post-quantum cryptography migration plan</u></a></li> 
 <li><a href="https://aws.amazon.com/blogs/security/customer-compliance-and-security-during-the-post-quantum-cryptographic-migration/" rel="noopener" target="_blank"><u>Customer compliance and security during the post-quantum cryptographic migration</u></a></li> 
 <li><a href="https://aws.amazon.com/blogs/security/aws-lc-fips-3-0-first-cryptographic-library-to-include-ml-kem-in-fips-140-3-validation/" rel="noopener" target="_blank"><u>AWS-LC FIPS 3.0: First cryptographic library to include ML-KEM in FIPS 140-3 validation</u></a></li> 
 <li><a href="https://eprint.iacr.org/2024/176" rel="noopener" target="_blank"><u>The impact of data-heavy, post-quantum TLS 1.3 on the Time-To-Last-Byte of real-world connections</u></a></li> 
 <li><a href="https://catalog.workshops.aws/using-pq-crypto-on-aws/en-US" rel="noopener" target="_blank"><u>AWS Workshop: Using Post-Quantum Cryptography on AWS</u></a></li> 
 <li><a href="https://doi.org/10.6028/NIST.FIPS.203" rel="noopener" target="_blank"><u>NIST FIPS 203, Module-Lattice-Based Key-Encapsulation Mechanism Standard (ML-KEM)</u></a></li> 
</ul> 
<p>If you have feedback about this post, submit comments in the <strong>Comments</strong> section below.</p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Author" class="aligncenter size-full wp-image-13246" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/04/10/Alex-Weibel-Author.png" width="120" /> 
  </div> 
  <h3 class="lb-h4">Alex Weibel</h3> 
  <p>Alex is a Senior Software Development Engineer in AWS Cryptography. He’s a contributor to the Amazon TLS Library s2n-tls, the Amazon Corretto Crypto Provider (ACCP), and AWS Libcrypto (AWS-LC). Previously, Alex worked on TLS termination and HTTP request proxying for Amazon S3 and Elastic Load Balancing, developing new features for customers. Alex holds a Bachelor of Science degree in Computer Science from the University of Texas at Austin.</p> 
  <p></p>
 </div> 
</footer>
