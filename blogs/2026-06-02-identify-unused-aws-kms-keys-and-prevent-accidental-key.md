---
title: "Identify unused AWS KMS keys and prevent accidental key deletions"
url: "https://aws.amazon.com/blogs/security/identify-unused-aws-kms-keys-and-prevent-accidental-key-deletions/"
date: "2026-06-02"
author: "Andrea Rossi and Poojil Tripathi"
feed_url: "https://aws.amazon.com/blogs/security/category/security-identity-compliance/aws-key-management-service/feed/"
---
AWS KMS has introduced the GetKeyLastUsage API, enabling organizations to quickly identify when each key last performed cryptographic operations for improved audit capabilities and key lifecycle management. The article demonstrates two primary use cases: cost optimization through unused key cleanup, and preventing accidental key deletion using policy controls with the kms:TrailingDaysWithoutKeyUsage condition key. The tracking period started April 23, 2026, and the solution supports auditing key usage across multiple AWS accounts and regions.
