---
title: "Caching KMS data keys in multi-thread environments: Per-tenant encryption for event-driven systems at scale"
url: "https://aws.amazon.com/blogs/security/caching-kms-data-keys-in-multi-thread-environments-per-tenant-encryption-for-event-driven-systems-at-scale/"
date: "2026-08-06"
author: "Maria Gutovsky"
feed_url: "https://aws.amazon.com/blogs/security/category/security-identity-compliance/aws-key-management-service/feed/"
---
This post assumes familiarity with envelope encryption and the AWS Encryption SDK. When your encryption system generates millions of duplicate API calls per hour, costs spiral and performance degrades. That’s exactly the challenge NICE Actimize faced while operating their global-scale, event-driven financial crime detection platform on Amazon Web Services (AWS).
