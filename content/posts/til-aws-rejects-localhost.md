---
title: "TIL: AWS WAF Rejects Request with localhost URLs in Body"
date: 2024-05-09T08:00:00
draft: False
---


While testing out a new feature for an AWS app, I encountered 403 errors on HTTP requests being placed to the service I had built. The error response body contained HTML for a generic AWS Cloudfront 403 Forbidden error page prompting me to try again later, or contact my sys admin. I was able to confirm through the Cloudfront UI that the request was indeed blocked, but for some reason, other requests from the same source to the same host were being let through on other routes. For example `POST /resource1` was ok but `POST /resource2` was not even though the source and host information for both requests were the same.

The error page provides a cloudfront request id that allows you to look up the individual request in logs for cloudfront and AWS Web Application Firewall (WAF). The WAF logs indicated that my blocked request violated the `EC2MetaDataSSRF_BODY` rule in the [`AWSManagedRulesCommonRuleSet`](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-baseline.html). The description of this rule is not particularly helpful:

	Inspects for attempts to exfiltrate Amazon EC2 metadata from the request body.

A [reddit post](https://www.reddit.com/r/aws/comments/g9trin/what_do_the_aws_managed_waf_rules_actually_mean/) shines a bit more light on the subject and indicated that this rule will block requests with URLs that point to `localhost`, `127.0.0.1`, etc. The request I was making included a callback URL (think webhook) that was defaulted to `http://localhost` from testing on my local machine. Changing the URL to something else (or blanking it out for that matter) allowed the request to get through the WAF unimpeded.
