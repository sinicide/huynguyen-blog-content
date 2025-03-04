---
layout: blog
title: Email Challenges
date: 2025-03-04T14:13:15-06:00
lastMod: 2025-03-04T17:08:40-06:00
categories:
  - email
tags:
  - email
  - spf
  - dkim
  - dmarc
  - smtp
description: Configuring email for a custom domain can be a mystery, but with a little research and a few hours spent, knowledge is gained!
disableComments: true
draft: false
---
So recently I decided to tackle learning a bit about Email as I needed to troubleshoot why emails from my custom domain were failing to send and why I was getting phishing emails routed through by my custom domain. I'll admit, I have very little knowledge in the area of Emails. I originally had a setup for years that kinda just worked without issue, this was when G Suite was free. But now I really needed to tackle this.

## The Problem
{{< image src="https://r2.huynguyen.me/blog/2025/03/04/email-challenges/images/phishing_email.jpg" alt="Phishing Email" >}}
I started noticing emails arriving in my inbox that tried to look weirdly legitimate. For example the above email which I manually flagged for spam was mailed by my custom domain.

How do I know the email was a phishing attempt? For 1, my name's not Joel. 2, If you check things like the phone number to customer service there, it shows up as a suspicious Paypal scammer number.

I later found out that emails I was sending was getting flag for spam as well. So I needed to fix my setup and do some research.

## The Setup
So how this works is that my custom domain which is hosted by Cloudflare, is configured to forward emails from my custom domain to a gmail account. That works without issue, because Cloudflare has you verify the destination email address before it'll send it there. This covered receiving email for my custom domain, which is really what I need it for.

Now sending email as my custom domain is not something I do frequently, but I do have a need to be able to legitimately do so. I had originally configured Gmail to send mail as this custom domain, via the `Send mail as` configuration under the `Accounts and Import` settings within Gmail. Originally I was using Gmail's smtp servers to send mail. Unfortunately when inspecting email sent by my custom domain, it was failing DKIM and DMARC.

## Terminology
Alright let's go over some terminology so before going any further.

SPF - Sender Policy Framework
DKIM - Domain Keys Identified Mail
DMARC - Domain Based Message Authentication Reporting and Conformance

### SPF
Sender Policy Framework is a way to list or identify the approved domains/IPs which can send emails from. This is configured as a `TXT` DNS Record for your custom domain. So this allows mail servers to verify if the email should come from an expected IP(s). This is simple enough to setup through Cloudflare and I've included both Cloudflare's spf records as well as Gmail's spf records.

Below is what you typically see mentioned online.
```
v=spf1 include:_spf.mx.cloudflare.net include:_spf.google.com ~all
```

### DKIM
Domain Keys Identified Mail, is a way to use Public Key Cryptography to verify that the email is actually sent from an approved sender. So when email is sent, it'll contain a header that signs the email using a Private Key, then mail servers receiving the email can look up the public key via another DNS record to validate that the mail is legitimate. The problem here with my setup is that the way that I had configured `Send mail as` in Gmail doesn't actually sign my emails with DKIM because the free Gmail Account doesn't offer this with `smtp.gmail.com` I believe this worked fine when I had a free G Suite account.

This is configured as a `TXT` DNS Record who's key is typically `xxxx._domainkey` or something like that. Now you unfortunately cannot just add some random thing here, because you need an actual mail server to hold the Private Key, so that you can specify the Public key in this DNS record.

### DMARC
Domain Based Message Authentication Reporting and Conformance, this is basically a rule/policy/instruction for mail servers on how to handle email received by your domain. This is configured as a `TXT` DNS record with the key `_dmarc`, which the mail servers can lookup.

For example the contents of a `_dmarc` record might look like
```
v=DMARC1; p=quarantine; sp=quarantine; rua=mailto:xxxx@example.com; ruf=mailto:xxxx@example.com
```

The `v` specifies the version, the `p` specifies the policy, which accepts, `reject` which will reject the email, `quarantine` which will accept it but send it to Spam, and finally `none` which will take no action.

The `sp` is the sub domain policy, which follows the same options. There's also an additional default configuration here `pct` which defaults to a value of `100` the `pct` configuration marks the percentage of failed DMARC email to apply the policy to. 100 being all email vs say 75 would mean 75 percent of failed DMARC emails would have the policy specified applied, so if you set the policy to be rejected, then 75 percentage of failed DMARC email would be rejected.

The email addresses here are used for configuring DMARC reports, the `rua` which indicates Aggregate reports and the `ruf` which indicates Forensic reports. These get periodically sent to those `mailto:xxx@example.com` addresses. This can be helpful for periodic review, but we'll see if it's extremely useful or another alert to ignore.

Originally I had my policy set to `p=none` but unfortunately that was not a solution for passing DMARC with Gmail.

If you want to learn more? Check out Cloudflare's excellent article on it [here](https://www.cloudflare.com/learning/email-security/dmarc-dkim-spf/)

## Tools are Cool
{{< image src="https://r2.huynguyen.me/blog/2025/03/04/email-challenges/images/learndmarc.jpg" alt="learndmarc.com" >}}
In my research, I stumbled across a pretty cool website tool called [www.learndmarc.com](https://learndmarc.com/) This was an interactive way to understand how SPF, DKIM, and DMARC were passing or failing. This was nicer than just looking at the email headers and trying to make sense of the gibberish. 

To get started you'd send an email to the email address that the tool tells you to. Once it receives your email, it'll go through how it identifies it's legitimacy and even breaks down the DMARC policy you may have configured.

In my testing SPF was succeeding because it was being sent out by Google's smtp server, but DKIM was failing, which ultimately resulted in DMARC failing. DKIM failed because the sender domain differ from the custom domain. This is due to me sending emails through Gmail's smtp server, because it doesn't sign the email with a Private Key for DKIM.

## The Solution
So clearly my main issue here is that while SPF can succeed, DKIM wasn't configured and my DMARC configuration would essentially fail if the policy I configured told it to quarantine email (e.g. send to spam). 
So it was pretty clear from the report that I needed to configure DKIM and that out of the box Free gmail didn't offer this so I couldn't configure my DNS records for DKIM. I could only configure SPF and DMARC policies.

What I needed was a SMTP server that I could send mail through that would sign it with a Private Key. For my use case I don't really need to send a ton of emails, nor do I need a bunch of email addresses, so I don't want to pay $7 USD a month for the lowest tier offering with Google Workspace.

What I ended up finding for a solution was [SMTP2GO](https://smtp2go.com/) which provided a Free Plan (without credit card) that would allow up to 1000 emails sent a month and up to 200 emails a day. This was perfect for my low use case needs, cause who doesn't love free-ninty-free?

{{< image src="https://r2.huynguyen.me/blog/2025/03/04/email-challenges/images/smtp2go.jpg" alt="SMTP2GO" >}}

Setup was extremely easy, they provided me with all the DNS record configuration that I needed to make on Cloudflare, the only trouble I ran into was that I initially created the DNS records with the proxy status enabled, which did not correctly resolve the specific smtp2go endpoints. Once that was fixed, the verification process succeeded.

I then created my accounts for their smtp server. Went back into gmail and configured my `Send mail as` with the new smtp server from smtp2go's documentation. Finally I was ready to re-test with [learndmarc.com](https://learndmarc.com) which now passed DKIM and ultimately passed DMARC, policy. I can now actually configure the DMARC policy to correctly quarantine or reject failed mail.

Not bad for a few hours worth of research, this has certainly been a long time coming and I'm glad I was able to learn something new and fix a problem. Hopefully you've learned something new as well.