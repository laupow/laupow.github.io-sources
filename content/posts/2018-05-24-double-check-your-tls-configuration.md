---
title: "TLS misconfiguration: how CloudFront noticed first"
date: 2018-05-24
draft: false
tags: ["tls", "aws", "deployment"]
---
One day after a routine deployment our CloudFront URLs broke. The website was completely useless. 
Requests to CloudFront, mostly for CSS and JS, were returning an HTTP 502 Bad Gateway error. 
Thanks to Kubernetes, we quickly rolled back the botched deployment.
But how did a deployment break our CloudFront distribution?
<!--more-->
## What changed after the deploy?
During our build process, we generate a unique build number added to all CloudFront URLs (e.g. `/scripts/assets.min.js?build=2018.05.25.1`).
Putting the build number in the CloudFront query string forces CloudFront to fetch new resources after deploying code.
The CloudFront distribution used our web server as the primary origin:

**Origin URL**   
`https://www.example.com/scripts/assets.min.js`

**CloudFront URL, before deploy**   
`https://dk0ls632913f.cloudfront.net/scripts/assets.min.js?build=2018.05.25.1`

**CloudFront URL, after deploy**   
`https://dk0ls632913f.cloudfront.net/scripts/assets.min.js?build=2018.05.25.2` 

Usually, CloudFront isn't a problem following a deploy, and all the URLs above would continue to work.
This time, however, the cached objects in CloudFront masked a problem we didn't know we had. 

## The CloudFront Error 
We didn't change any settings on our distribution, so why can't CloudFront connect to our origin servers? 
There was a little nugget of hope in the error
message: a [link to a AWS documentation page](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/http-502-bad-gateway.html)
with common reasons why CloudFront returns 502 Bad Gateway.

Between the deployments, we had also done a Google Kubernetes Engine upgrade. An unintended side-effect likely reverted the TLS certificates to an
older version of the certificates. The swap may have been related to the resolution of a [known TLS issue](https://github.com/kubernetes/ingress-gce/issues/46)
which limited the number of certificates propagated through Google Cloud resources.

Initially, I didn't think the certificates were an issue. I naïvely opened a web browser, pointed it at our server, 
and the pages loaded over TLS. Our certificate was still valid, but according to CloudFront, a misconfiguration existed.
Just in case, I tried to load the same URL over the using another tool and immediately saw a `SSLError: CERTIFICATE_VERIFY_FAILED`. 

Now I could somewhat reproduce the issue CloudFront identified.
I ran a couple more tests against the server to examine the TLS configuration.
 
First, [SSL Labs](https://ssllabs.com/) and next [ciperscan](https://github.com/mozilla/cipherscan).
 
Ciperscan said the certificate was untrusted but didn't provide details. The SSL Labs results were interesting because that report identified a
couple of issues that matched issues described in the AWS documentation. Notably: *the intermediate certificates were missing*. 

(Web browsers must mitigate missing intermediate certificates. I won't try to describe the mechanism here, but the website worked
for my browser and I suspect most other users, too.)

I went back to the AWS documentation to focused on intermediate certificates. In the [middle of the documentation](
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/http-502-bad-gateway.html#ssl-certificate-expired)
there is a critical statement:

**Note: If the full chain of certificates, including the intermediate certificate, is not present, CloudFront drops the TCP connection.**

It's a match. Our TLS configuration didn't include the intermediate certificates, and therefore CloudFront didn't load the new assets
for the recent code deploy. 

This missing certificates probably existed for almost 24 hours before the deployment issue. But since the cached resources 
continued to work normally, CloudFront masked the underlying problem.

The fix was simple: update our TLS certificates to include the sender's certificate first, followed by all intermediate certificates, in the order they are signed. CloudFront could connect to our origin servers once again.

## Lessons Learned
   
- Web Browsers should not be the only test to validate a TLS configuration.

Using a tool like [SSL Labs](https://ssllabs.com/) to analyze your certificates and server configuration is essential. 
Even dumping the raw output with `openssl s_client –connect domainname:443` provides valuable information such as certificate presence and order.

- Don't ignore configuration warnings even if web browsers work OK.

I've seen the message "Wrong Order" warning for the certificates on other servers, but didn't have my head
fully wrapped around why it mattered. I understand the bigger picture now: other software
implementing SSL/TLS per the spec may not tolerate configuration issues, like certificates being out of order.
CloudFront's TLS policy is more strict than a typical web browser, and therefore, the secure connections to our origin failed.

## Viewing the certificates from your server
OpenSSL can get the certificate information from a server. 

Client supports SNI   
`openssl s_client –connect domainname:443 –servername domainname`

Client doesn't support SNI   
`openssl s_client –connect domainname:443`

If the client doesn't support SNI, typically you receive the certificate of the primary/default web server as
configured at `domainname:443`.

## Note about root certificates
If you use the `openssl s_client -connect` you might see a note about a self-signed certificate:
```
depth=3 C = SE, O = AddTrust AB, OU = AddTrust External TTP Network, CN = AddTrust External CA Root
verify error:num=19:self signed certificate in certificate chain
verify return:0
```

The root certificate issued by a CA is a self-signed certificate.
This [Stack Overflow post](https://stackoverflow.com/a/4106224/841203) describes the condition in more detail.

Our certificate provider adds the root CA certificate to its set, by default. 
So we leave it intact for simplicity and consistency across deployments. 
According to the TLS 1.2 spec, adding the root certificate is acceptable:
```
certificate_list
...Because certificate validation requires that root keys be distributed
independently, the self-signed certificate that specifies the root certificate
authority MAY be omitted from the chain, under the assumption that the
remote end must already possess it in order to validate it in any case.
```
