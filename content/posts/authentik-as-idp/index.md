---
title: 'Authentik as Idp'
date: '2025-04-29'
draft: false
categories:
  - default
tags:
  - selfhosting
---

After my previous post about Mailcow, I want to talk about another piece of my homelab infrastructure: Authentik, my identity provider of choice. I'll cover why I chose it, how I set it up, and how I've integrated it with various services.

# Why an identity provider?

Let's first discuss why you might want a centralized identity provider in your homelab.

## The problem with multiple accounts

When you start self-hosting services, each one typically comes with its own authentication system. This quickly becomes unmanageable:

- *Password fatigue*: Creating and remembering different passwords for each service is tedious.
- *User management overhead*: Adding or removing access means updating accounts across multiple services.
- *Inconsistent security*: Different services have different security capabilities and settings. Some have 2FA, some don't.

## What an identity provider solves

A good identity provider centralizes authentication and offers:

- *Single sign-on (SSO)*: One login for all your services.
- *Consistent security policies*: Apply the same password requirements and 2FA across all services.
- *Centralized user management* and central logs for all things authentication.

# Why I chose Authentik

There are several identity providers in the self-hosting space. The most popular ones are:

### Keycloak

[Keycloak](https://www.keycloak.org/) is an established open-source solution backed by Red Hat. It's enterprise-grade, feature-rich, and has excellent documentation. However, it's quite resource-intensive and the UI can be a bit much for home use. I first went with Keycloak, but gave up after a couple hours of understanding the lingo. It also has many features, which your average homelab will never require.

### Authelia

[Authelia](https://www.authelia.com/) is lightweight and simpler than Keycloak. It's a good option if you need basic authentication features without the overhead. What I really appreciate is the single config file required! However, it lacks some of the more advanced features like directory synchronization and flexible flows.

### Authentik

[Authentik](https://goauthentik.io/) sits somewhere in the middle. It offers most of the features you'd want from an enterprise solution while remaining approachable. The UI is modern and user-friendly (once you understand the lingo), and it has LDAP, SSO, pyoxying, and more built-in. It also has a lot of flexibility thanks to the concept of editable flows, which essentially define every process (such as signup or password reset).

The [docs](https://docs.goauthentik.io/docs) are also very comprehensive and even include articles for how to connect common selfhosted apps to Authentik. 

# My Authentik setup

I deployed Authentik using Docker Compose, following the [official documentation](https://docs.goauthentik.io/docs/install-config/install/docker-compose). 

I'm using Traefik as my reverse proxy, therefore, I've added a couple lies to the authentik server:

```yaml
labels:
  - "traefik.enable=true"
  ## HTTP Routers
  - "traefik.http.routers.authentik-rtr.rule=Host(`auth.leonmuscat.de`)"
  # - "traefik.http.services.authentik-rtr.loadbalancer.server.port=9000"
  - "traefik.http.routers.authentik-rtr.entrypoints=websecure"
  - "traefik.http.routers.authentik-rtr.tls.certresolver=myresolver"
  ## Individual Application forwardAuth regex (catch any subdomain using individual application forwardAuth)  
  - "traefik.http.routers.authentik-rtr-outpost.rule=HostRegexp(`{subdomain:[a-z0-9-]+}.leonmuscat.de`) && PathPrefix(`/outpost.goauthentik.io/`)"
  - "traefik.http.routers.authentik-rtr-outpost.entrypoints=websecure"
  - "traefik.http.routers.authentik-rtr-outpost.tls.certresolver=myresolver"
  ## This is for proxying to a docker service
  - "traefik.http.middlewares.authentik.forwardauth.address=http://authentik_server:9000"
  - "traefik.http.middlewares.authentik.forwardauth.trustForwardHeader=true"
  - "traefik.http.middlewares.authentik.forwardauth.authResponseHeaders=X-authentik-username,X-authentik-groups,X-authentik-email,X-authentik-name,X-authentik-uid,X-authentik-jwt,X-authentik-meta-jwks,X-authentik-meta-outpost,X-authentik-meta-provider,X-authentik-meta-app,X-authentik-meta-version,authorization"
  ## HTTP Services
  - "traefik.http.routers.authentik-rtr.service=authentik-svc"
  - "traefik.http.services.authentik-svc.loadBalancer.server.port=9000"
  - "kuma.authentik.http.name=authentik"
  - "kuma.authentik.http.url=https://auth.leonmuscat.de"
```

To make Authentik's proxy work, I also defined a middleware in Traefik as follows:

```yaml
middlewares-authentik:
  forwardAuth:
    address: "http://authentik_server:9000/outpost.goauthentik.io/auth/traefik"
    trustForwardHeader: true
    authResponseHeaders:
      - X-authentik-username
      - X-authentik-groups
      - X-authentik-email
      - X-authentik-name
      - X-authentik-uid
      - X-authentik-jwt
      - X-authentik-meta-jwks
      - X-authentik-meta-outpost
      - X-authentik-meta-provider
      - X-authentik-meta-app
      - X-authentik-meta-version
```

With that I can secure any application with Authentik. A lot of services support SSO, which in my view is the best option, as it's usually a single button press to login. If that's not the case, as with Jellyfin or Grocy because it breaks stuff, I fallback to LDAP. Finally, some apps don't require accounts because I'm the only that uses them or there's no need to differentiate users, in which case I use the proxy.

I customized some things, like branding and flows for user invitation. The guides by [Cooptonian](https://www.youtube.com/@cooptonian) are fantastic, so I'm not going to go over the same things here. However, I do want to mention my integration with [Ntfy](ntfy.sh), my selfhosted notification service.

## Ntfy notifications

To get Ntfy working, I defined a webhook notfication transport using the webhook URL `https://ntfy.leonmuscat.de?auth=token` (as explained [here](https://docs.ntfy.sh/publish/#access-tokens)). However, per default, the notfications will just be a long JSON. To improve that, I defined a webhook mapping under property mappings with the following expression:

```py
fields = [
  {
    "title": "Severity",
    "value": notification.severity
  }
]
if notification.event:
  if notification.event.user:
    fields.append({
      "title": "User",
      "value": str(notification.event.user.get("username"))
    })
  for key, value in notification.event.context.items():
    if not isinstance(value, str):
      continue
    fields.append({"title": key[:256], "value": value[:1024]})
    
formatted_string = "\n".join(f"**{field['title']}**: {field['value']}" for field in fields) + "\n"

body = {
  "topic": "authentik",
  "title": notification.body.split(":")[0],
  "markdown": True,
  "tags": ["lock", "warning"],
  "message": formatted_string,
  "actions": [{"action": "view", "label": "Admin panel", "url": "https://auth.leonmuscat.de"}],
  "icon": "https://auth.leonmuscat.de/media/homelab-logo-small.png"
}

return body
```

With that, the notifications are now neatly formatted.

## Access control

Authentik allows to define groups and roles. I use these to give users access to specific groups of services. I have a set of default services that every user has access to, which simply require a generic user group. For media streaming, I have a user group (with access to Jellyfin and Jellyseerr via LDAP as explained [here](https://docs.goauthentik.io/integrations/services/jellyfin/)) and an admin group (with additional access to Sonarr, Radarr, and Jackett). You get the idea. 

This makes it easy to control who has access to which services in my homelab.

# Conclusion

Implementing Authentik has significantly improved the usability and security of my homelab. It also makes adding friends and family to my homelab services very easy!

Is it necessary for everyone? If you only have a couple of services, probably not. But as soon as you find yourself managing users across multiple applications, an identity provider becomes invaluable.

