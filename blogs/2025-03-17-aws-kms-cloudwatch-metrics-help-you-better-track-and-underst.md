---
title: "AWS KMS CloudWatch metrics help you better track and understand how your KMS keys are being used"
url: "https://aws.amazon.com/blogs/security/aws-kms-cloudwatch-metrics-help-you-better-track-and-understand-how-your-kms-keys-are-being-used/"
date: "Mon, 17 Mar 2025 17:35:16 +0000"
author: "Norman Li"
feed_url: "https://aws.amazon.com/blogs/security/category/security-identity-compliance/aws-key-management-service/feed/"
---
<p><a href="https://aws.amazon.com/kms/" rel="noopener" target="_blank">AWS Key Management Service (AWS KMS)</a>&nbsp;is pleased to launch key-level filtering for AWS KMS API usage in <a href="https://aws.amazon.com/cloudwatch/" rel="noopener" target="_blank">Amazon CloudWatch</a> metrics, providing enhanced visibility to help customers improve their operational efficiency and aid in security and compliance risk management.</p> 
<p>AWS KMS currently publishes account-level AWS KMS API usage metrics <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Service-Quota-Integration.html" rel="noopener" target="_blank">to Amazon CloudWatch</a>, enabling you to monitor and manage your API usage. However, if you’re using numerous KMS keys, pinpointing the ones with the highest request rate quota usage or significant API costs becomes challenging. For example, if you have more than 10 active KMS keys in your account, prior to this launch you would have needed to build a custom <a href="https://docs.aws.amazon.com/athena/latest/ug/cloudtrail-logs.html" rel="noopener" target="_blank">CloudTrail and Amazon Athena based solution</a> to locate which specific keys are driving the majority of API usage and costs. With the new CloudWatch metrics, which are available under the <code style="color: #000000;">AWS/KMS</code> namespace in CloudWatch, you can track, understand, and set alerts on detailed API usage at the individual KMS key level without building a costly customized solution.</p> 
<p>This blog post explores several use cases to help you better take advantage of these newly introduced CloudWatch metrics to manage your AWS KMS API usage and costs. The use cases cover viewing and understanding your API usage at the key level, and creating CloudWatch alerts to detect unintentional runaway usage.</p> 
<h2>Overview of new CloudWatch metrics for KMS keys</h2> 
<p>With CloudWatch metrics for KMS keys, you can now do the following:</p> 
<ol> 
 <li>View the API usage for a specific KMS key, filtered by individual API operations (for example, <code style="color: #000000;">Encrypt</code>, <code style="color: #000000;">Decrypt</code>, or <code style="color: #000000;">GenerateDataKey</code>).</li> 
 <li>See the aggregated usage across cryptographic operations for a given KMS key.</li> 
 <li>Set up an alarm if a specific KMS key exceeds a specified threshold on a single API operation, or a set of API operations.</li> 
</ol> 
<p>This streamlined approach allows you to quickly monitor, understand, and troubleshoot the API usage patterns of your KMS keys, without the overhead of the previous multi-step process. Let’s detail how these key-level API usage metrics can be used in two real-world examples.</p> 
<h2>Example 1: How to locate the KMS keys that consume the most API usage quota or contribute the most API charges</h2> 
<p>When you surpass your AWS KMS API request rate quotas, you can view your AWS KMS API utilization within the <a href="https://console.aws.amazon.com/servicequotas" rel="noopener" target="_blank">Service Quotas console.</a> However, you might still find it cumbersome to identify the KMS keys that consume the largest amount of your request quota. When you receive the AWS KMS API charges that exceed your expectation, you can check the detailed billing usage in each AWS Region in Cost Explorer, but you cannot easily locate the KMS keys with the most API charges. This process becomes even more challenging when you manage a large number of KMS keys.</p> 
<p>With the key-level API usage CloudWatch metrics, you can use the advanced <a href="https://docs.aws.amazon.com/grafana/latest/userguide/CloudWatch-using-the-metric-query.html" rel="noopener" target="_blank">metric query</a> option to query CloudWatch Metrics Insights data with a user-friendly dialect of SQL to locate the KMS keys that consume the largest portion of the API usage quota or contribute the most API charges.</p> 
<h3>Walkthrough</h3> 
<p>To use Amazon CloudWatch Metrics Insights to identify the top 20 KMS keys that have the most cryptographic API usage up to the last 3 hours, complete the following steps:</p> 
<ol> 
 <li>Open the <a href="https://console.aws.amazon.com/cloudwatch/" rel="noopener" target="_blank">CloudWatch console</a>.</li> 
 <li>In the navigation pane, choose <strong>Metrics</strong>, and then choose <strong>All metrics</strong>.</li> 
 <li>Choose the <strong>Multi source query</strong> tab.</li> 
 <li>For the data source, choose <strong>CloudWatch Metrics Insights</strong>.</li> 
 <li>You can enter the following example query in <strong>Editor</strong> view:<br /> 
  <blockquote>
   <p><strong>Note:</strong> In <strong>Builder</strong> view, the metric namespace, metric name, filter by, group by, order by, and limit options are shown. In <strong>Editor</strong> view, the same options as in <strong>Builder</strong> view are shown in query format.</p>
  </blockquote> 
  <div class="hide-language"> 
   <pre class="unlimited-height-code"><code class="lang-text">	SELECT SUM(SuccessfulRequest)
	FROM SCHEMA("AWS/KMS", KeyArn, Operation)
	GROUP BY KeyArn
	ORDER BY MAX () DESC
	LIMIT 20</code></pre> 
  </div> </li> 
 <li>Choose <strong>Run</strong> in the <strong>Editor</strong> view or <strong>Graph query</strong> in the <strong>Builder</strong> view.</li> 
</ol> 
<h2>Example 2: How to set a new detailed alarm on unintentional runaway AWS KMS API usage</h2> 
<p>Running big data processing workflows that read <a href="https://aws.amazon.com/s3/" rel="noopener" target="_blank">Amazon Simple Storage Service (Amazon S3)</a> files encrypted by KMS keys is a common scenario for analytics, business reporting, or machine learning projects. Typically, these workflows read a limited number of files from S3 on each invocation. However, misconfigured workflows could unintentionally read a large number of S3 files, which could result in exceeding your AWS KMS API request rate quotas or incurring undesirable charges due to spiky AWS KMS API usage. Historically, to address this issue, you would have had to build a customized alarm system by following these steps: 1) send AWS CloudTrail events generated by AWS KMS to Amazon CloudWatch Logs; 2) write queries in Amazon CloudWatch Logs Insights to track your API request usage; and 3) enable anomaly detection on the corresponding CloudWatch Log Insights math expression.</p> 
<p>Now, with key-level API usage CloudWatch metrics, you can directly enable anomaly detection on these metrics to set up alarms for anomalous AWS KMS API usage patterns. This provides a more streamlined and efficient way to monitor and detect potential runaway workflows. By using these CloudWatch metrics and anomaly detection capabilities, you can proactively identify and address unintended increases in AWS KMS API usage, helping to avoid unexpected charges or service disruptions in your analytics, reporting, or machine learning pipelines.</p> 
<h3>Walkthrough</h3> 
<p>Consider a scenario where you have an analytics workflow that runs frequently, which uses the <code style="color: #000000;">Decrypt</code> AWS KMS API operation on a KMS key to decrypt and read data from S3. You would like to enable anomaly detection on the KMS key to trigger an alarm when the <code style="color: #000000;">Decrypt</code> call volume to the specific KMS key sees a discernible trend or pattern. To do so, complete the following steps:</p> 
<ol> 
 <li>Open the <a href="https://console.aws.amazon.com/cloudwatch/" rel="noopener" target="_blank">CloudWatch console</a>.</li> 
 <li>In the navigation pane, choose <strong>Metrics</strong>, and then choose <strong>All metrics</strong>.</li> 
 <li>Choose <strong>KMS</strong>, and then choose <strong>KeyArn</strong>, <strong>Operation</strong>.</li> 
 <li>In the search bar, enter the Amazon Resource Name (ARN) of the key, and then choose <strong>Search</strong>. Select the CloudWatch metric you would like to enable anomaly detection for.</li> 
 <li>Navigate to <strong>Graphed metrics</strong>, and using the <strong>Statistic</strong> and <strong>Period</strong> drop-down lists, choose the statistic and period that you would like to monitor. Then you can enable anomaly detection by selecting the <strong>Pulse</strong> icon. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_37442" style="width: 634px;">
   <img alt="Figure 1: How to enable anomaly detection on a SuccessfulRequest metric" class="size-full wp-image-37442" height="78" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/02/12/img1-1.png" style="border: 1px solid #bebebe;" width="624" />
   <p class="wp-caption-text" id="caption-attachment-37442">Figure 1: How to enable anomaly detection on a SuccessfulRequest metric</p>
  </div><p></p> </li> 
 <li>You can <a href="https://aws.amazon.com/blogs/mt/operationalizing-cloudwatch-anomaly-detection/" rel="noopener" target="_blank">adjust the anomaly detection</a> by setting the sensitivity to adjust the bandwidth, if needed. <p style="line-height: 1.25em;"></p>
  <div class="wp-caption aligncenter" id="attachment_37443" style="width: 634px;">
   <img alt="Figure 2: Anomaly detection is enabled on the SuccessfulRequest metric. The gray band illustrates the expected range of values and the anomaly is in red" class="size-full wp-image-37443" height="307" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/02/12/img2.png" style="border: 1px solid #bebebe;" width="624" />
   <p class="wp-caption-text" id="caption-attachment-37443">Figure 2: Anomaly detection is enabled on the SuccessfulRequest metric. The gray band illustrates the expected range of values and the anomaly is in red</p>
  </div><p></p> </li> 
</ol> 
<h2>Conclusion</h2> 
<p>This blog post highlighted the newly introduced key-level filtering capability for the AWS KMS API usage in CloudWatch. We showed two real-world use cases to demonstrate how you can use the new CloudWatch metrics. These use cases include improving operational visibility, setting up proactive alarms on anomalies in KMS API usage patterns, and potentially tracking detailed key usage for compliance purposes.</p> 
<p>If you have feedback about this blog post, submit comments in the <strong>Comments</strong> section below. If you have questions about this blog post, start a new thread in the <a href="https://forums.aws.amazon.com/forum.jspa?forumID=182" rel="noopener" target="_blank">AWS Key Management Service re:Post</a>.<br />&nbsp;</p> 
<footer> 
 <div class="blog-author-box">
  <img alt="Norman Li" class="alignleft size-full" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/02/12/yemzn.jpg" style="margin-left: 12px; margin-right: 18px; margin-top: 12px; margin-bottom: 6px; width: 93.750px; height: 125px;" />
  <span class="lb-h4" style="line-height: 2.1em; padding-top: 12px; margin-top: 24px;">Norman Li</span>
  <br />Norman is a Software Development Manager for AWS KMS. In this role, Norman leads the development of visibility features, as well as internal scalability initiatives. Outside of work, Norman likes to spend time in the beautiful Pacific Northwest mountains.
 </div> 
 <div class="blog-author-box">
  <img alt="Haiyu Zhen" class="alignleft size-full" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2025/02/12/zhaiyu.jpg" style="margin-left: 12px; margin-right: 18px; margin-top: 12px; margin-bottom: 6px; width: 93.750px; height: 125px;" />
  <span class="lb-h4" style="line-height: 2.1em; padding-top: 12px; margin-top: 24px;">Haiyu Zhen</span>
  <br />Haiyu is a Senior Software Development Engineer for AWS KMS. She specializes in building secure, large-scale distributed systems and is passionate about enhancing cloud-native application security without compromising performance.
 </div> 
</footer>
