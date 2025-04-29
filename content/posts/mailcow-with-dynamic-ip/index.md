---
title: 'Mailcow With Dynamic IP'
date: '2025-04-28'
draft: false
categories:
  - default
tags:
  - selfhosting
  - email
---

I've been hosting my own email with Mailcow for over 2 years now. I want to go over general considerations when it comes to selfhosting email and the setup I went for.

# Motivation

Before we go the technical bits, let's first discuss the reasons to selfhost email.

## Why you should not selfhost mail

Selfhosting email is quite a daunting task, which is why it's often not encouraged in selfhosting communities. There are many good reasons for that:

- *Deliverability* can be a pain and a half: There are many stories on r/selfhosted about people having their mailserver blocked by Google, Microsoft, and others. If anything goes wrong - or simply for no reason at all - your IP might end up on a spam list. Resolving these issues takes a lot of time and is very frustrating.
- *Configuration* is not easy: Compared to many other selfhosting projects, email is quite cumbersome to set up. You have to correctly configure SPF, DKIM, DMARC, etc.
- Email is a *critical service*: If you screw up the config or experience any data loss, you face the loss communication records, files, and contacts. If there's no proper backup system in place, then absolutely do not selfhost email.

Besides these points, there are many great services out there that are (likely) privacy respecting and budget-friendly, including [ProtonMail](https://proton.me/mail) and [Tutanota](https://tuta.com). 

## Why I still went for it

Email is such an essential service and there's so much private information that is tight to it. There is likely not other service that provides such a complete profile of the user. This naturally makes a strong argument to move from Gmail and Outlook to a selfhosted service, where you control the data.

But more than that, you get features that are often behind paywalls, such as no storage space restrictions, large attachments, aliases and catch-all addresses and of course a custom domain.

# Selfhosting email 

So, let's selfhost email! We'll first explore the options, then I'll go over my setup using Mailcow.

## Comparison of selfhosted email options

There are (as always) many options to choose from. I don't want to go over each in detail, but want to highlight the most popular pieces of software.

### Custom setup with Postfix and Dovecot

Postfix and Dovecot are the two fundamental services often used for email. Postfix is a mail transfer agent, meaning it is responsible for sending and receiving emails via SMTP. Dovecot is IMAP and POP3 server that allows to store and retrieve emails via a client. Together they form a reliable and secure email solution, but configuring them yourself can be difficult, therefore, preconfigured options have been created. 

### Mail-in-a-Box

[Mail-in-a-Box](https://mailinabox.email/) is an established solution and aims to be an easy-to-deploy mail server. It runs on Postfix and Dovecot, but also Roundbox as a web mail client, calendar and contacts via a NextCloud, automatic backups, and more. It has a neat admin interface, which, while not feature-rich, exposes all important configuration. It's fully community-driven and has good documentation.

### Stalwart

[Stalwart](https://stalw.art/) is rather new to the scene with its first release in September 2022. It's meant to be modern, fast, and flexible. What's cool is the single-binary format. It combines everything you need in one place compared to the other options. 

### Docker Mailserver

[Docker Mailserver](https://docker-mailserver.github.io/docker-mailserver/latest/) aims to be a product-ready option with a single docker container. Everything can be configured via environment variables, which is nice! There's no admin interface included, instead everything is configurable via a CLI.

### Mailcow

Finally, we've reached [Mailcow](https://mailcow.email/). It is a popular choice that bundles many different services, similar to Mail-in-a-Box and Docker Mailserver. It includes SoGO, which serves as webmail client that also manages calendars and contacts. Compared to Mail-in-a-Box it also has additional features, such as temporary aliases, routing rules, and identity provider support, as well as an extensive admin UI. The community behind it is very large and helpful.

I went with Mailcow because its established (compared to Stalwart at the time), has a large community (compared to Mail-in-a-Box and Docker Mailserver), its admin interface, and its features.

## My Setup

I don't want to go over how to set up Mailcow, since there are many tutorials (like [this](https://community.hetzner.com/tutorials/setup-mailserver-with-mailcow)) out there and [the docs](https://docs.mailcow.email/) also cover everthing you need to know. However, I do want to cover 3 things: Dealing with a dynamic IP, running behind traefik, and integration with Authentik.

### Dealing with a dynamic IP

Email deliverability relies on trust, which largely comes down to your IP. As a result, a static IP with a good reputation is necessary. Unfortunately, my ISP doesn't offer static IPs for residential plans, therefore, I need a workaround. 

There are 2 options: 

- Run an SMTP relay on a VPS that sends and receives emails for the mailcow instance running at home. 
- Use an SMTP relay service.

Both options make me dependent on third-party providers and I went with the latter. Some will rightfully argue that the former is closer to the spirit of selfhosting, and maybe I'll switch in the future. However, using an SMTP relay service is convenient because it not only handles the dynamic IP problem, but also reputation as a whole. 

I'm using [SendGrid](https://sendgrid.com), which had a free forever tier for 100 emails/day before it was bought by Twilio. Now it's only free for 60 days. Luckely, all existing free-tier users are not affected by this change. If you're considering using an SMTP relay service, have a look at [SMTP2GO](https://www.smtp2go.com), who offer a free-tier with up to 200 emails/day and 1000 emails / month. I might switch to it in the future when SendGrid kills my free-tier. For me SendGrid has allowed me to circumvent any deliverability issues and it has some nice statistics included in their web interface.

As for how to configure an SMTP relay in Mailcow, there's a great [guide](https://docs.mailcow.email/manual-guides/Postfix/u_e-postfix-relayhost/) in their docs. 

### Integration with Traefik

My homelab uses Traefik as reverse proxy for all services. To integrate, we'll need 2 things. First, we need to modify the nginx container bundled with Mailcow to route using Traefik. Second, we need to dump the certificates by Traefik for SSL. 

The steps are detailed in the [docs](https://docs.mailcow.email/post_installation/reverse-proxy/r_p-traefik2/) and still work with Traefik v3. 

Don't forget to disable the built-in ACME!

### Authentik as identity provider

I use Authentik as identity provider across all my services. For a long time, Mailcow had its completely cut-off account system. You have the option to use Mailcow as an identity provider, but not the other way around. This changed with an update earlier in 2025. Now, you can set up an external identity provider that Mailcow users can use to authenticate. The steps are detailed [here](https://github.com/mailcow/mailcow-dockerized/issues/5445#issuecomment-1928600434). Unfortunately, I'm getting a generic "login failed" every time. I'll post an update if I figure out what I'm doing wrong.

If you're considering selfhosting email, I recommend taking the time to understand the fundamentals before diving in. Start with a proper backup strategy, expect to spend time on the initial configuration, and be prepared for occasional maintenance. Having said that, once properly set up, Mailcow has been remarkably stable with minimal intervention required.

# Conclusion

Selfhosting email has been an interesting experience and taught me a lot about the protocols involved. For those on dynamic IPs like me, don't let that be a deterrent - there are workable solutions. And if you do go down this path, the Mailcow community is quite helpful when you run into issues.

Is selfhosting email for everyone? Absolutely not. But if you value privacy, want complete control over your data, and don't mind the initial learning curve, it can be a rewarding addition to your selfhosted services.
