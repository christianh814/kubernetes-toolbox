# Kubernetes Authentication

Authentication for Kubernetes can be a bit of complex subject. If you're reading this for the first time; you'll need to get familiar with [how authentication works in k8s](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#users-in-kubernetes) before proceeding.

In short; k8s doesn't have a concept of "traditional users and groups" (excluding `serviceAccounts` of course), and relies on "external" authenticaion to provide "users/groups".

Some of the common methods of authentication are `X509 Client Certs`, `Static Password File`, and `OpenID Connect Tokens`. Usually people go with OIDC since it's the most flexable. 

There are many tools that are OIDC compliant; and can "front end" authentication to "back end" systems. The most popular ones are [Dex](https://github.com/dexidp/dex#dex---a-federated-openid-connect-provider) and [Keycloak](https://github.com/keycloak/keycloak#keycloak). These OIDC can front end (and also federate) things like LDAP, Active Directory, Google Auth, and Github (to name a few)

In this howto we will set up Dex to broker the authentication for k8s and GitHub. This assumes that you have installed kubernetes with [kops](k8s-kops.md#kubernetes-with-kops) and are using TLS with [let's encrypt](k8s-ingress-helm.md#tls). Most of the installs are done with [helm](../../README.md#helm) as well.

I'll try and make it as generic as possible; but if you're reading this, I'm assuming you know your way around k8s already.


Labs:

* [Prerequisites](#prerequisites)
* []()

## Prerequisites
