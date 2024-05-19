---
title: "OAuth2-Proxy on Kubernetes w/ Traefik"
date: "2024-05-19"
summary: "A Tutorial showing how to use OAuth2-Proxy on Kubernetes with Traefik's ForwardAuth. This setup features automatic redirects to both the signin and the originally accessed page."
categories: ["cloud"]
tags: ["kubernetes", "traefik"]
ShowToc: true
---

# Preamble
Recently, I wanted to protect a dashboard with my pre-existing Keycloak SSO. However, in order to protect routes, another component is needed: [oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/).
Unfortunately, the setup proved to be difficult, featuring a multitude of weird problems related to redirects.
After tinkering for a while, I finally found a configuration that works and felt like sharing it.

# The Configuration
In this blog post, `example.com` is assumed to be the root domain. OAuth2-Proxy resides under `oauth.example.com`, Keycloak under `keycloak.example.com`.
The namespace in the example is `app`, make sure to change references appropriately.
## OAuth2-Proxy
First, configuring the oauth2-proxy.
Personally, I used bitnami's helm chart, however, you can just use the `extraArgs` portion here as CLI flags.
```yml
ingress:
  enabled: true
  hostname: oauth.example.com
  tls: true
redis:
  enabled: false
extraArgs:
  - --provider=keycloak-oidc
  - --provider-display-name="Keycloak SSO"
  - --reverse-proxy=true
  # automatically redirects the user to the login rather than
  # waiting for them to click the only button available
  - --skip-provider-button=true
  - --code-challenge-method=S256
  - --allowed-role=application-admin
  # make sure to redirect to oauth2-proxy, it'll handle redirects from there
  - --redirect-url=https://oauth.example.com/oauth2/callback
  - --whitelist-domain=".example.com"
  - --cookie-domain=".example.com"
  # avoids 404 errors. use 200; apparently, 204 can lead to "204 doesn't allow body" in logs
  - --upstream="static://200"
configuration:
  existingSecret: oauth2-proxy-client
  oidcIssuerUrl: https://keycloak.example.com/realms/master
```

## Traefik Middleware
```yml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: oauth2-proxy
  namespace: app
spec:
  forwardAuth:
    address: http://oauth2-proxy.app.svc.cluster.local:80/oauth2/
    trustForwardHeader: true
```
Notice the lack of `/auth` at the end. This makes redirects work correctly; with the suffix, it would just show a weird "Found." link.

In the `address`, `oauth2-proxy` is the service name, `app` the namespace. The port may be different for you (the bitnami helm chart specifies `80` as the service port for the target port `4180`).

Change these values according to your setup.

## Ingress
```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: app
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: app-oauth2-proxy@kubernetescrd
```
The `app-` prefix of the middleware specifies the namespace the middleware is in; make sure to change it accordingly.

# Closing Thoughts
Massive thanks to [@curlup](https://github.com/curlup) for providing the last puzzle piece that finally lead to the completion of my setup in [this issue comment](https://github.com/oauth2-proxy/oauth2-proxy/issues/1297#issuecomment-2004788570) ❤
