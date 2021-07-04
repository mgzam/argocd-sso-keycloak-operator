# argocd-sso-keycloak-operator
This repository aims to give detailed instructions on how to configure SSO for OpenShift GitOps (ArgoCD) using KeyCloak Operator and Yaml files configuration

## Yaml Files : 

Yaml File examples exist under OpenShift Folder.

## SSO Integration for OpenShift GitOps with OpenShift Oauth Login:

To integrate ArgoCD Instance with OpenShift Oauth Login, we need to have some sort of a connector in between as ArgoCD supports only native OIDC connections. DEX Server has this connector implemented out of the box but not yet supported by Red Hat (Not even installed when you create an ArgoCD Instance using OpenShift-Gitops Operator). The second option is to install a Red Hat SSO Server with KeyCloak to handle integration with OpenShift Oauth. In both solutions, once we have installed the DEX Server or the RHSSO, we have an easy way to integrate it with Oauth using either an OauthClient or a service account as an OAuth client. Documentation here :

[Configuring SSO for Argo CD on OpenShift - GitOps | CI/CD | OpenShift Container Platform 4.7](https://docs.openshift.com/container-platform/4.7/cicd/gitops/configuring-sso-for-argo-cd-on-openshift.html#registering-an-additional-oauth-client_configuring-sso-for-argo-cd-on-openshift)

In this repository, you will find instructions and examples on how to configure SSO for ArgoCD on OpenShift with KeyCloak Operator and only with Yaml files without accessing KeyCloak GUI using KeyCloakRealm and KeyCloakClient Custom Resources. 

### Requirements:

* Install KeyCloak Operator in a namespace including CRDs, keycloak-operator deployment, role, rolebinding and service account. Check here [Install KeyCloak Operator]([Keycloak - Guide - Keycloak Operator on Openshift](https://www.keycloak.org/getting-started/getting-started-operator-openshift))

### Instructions :

In your KeyCloak Namespace; create a KeyCloak Instance using the following yaml :

```yaml
apiVersion: keycloak.org/v1alpha1
kind: Keycloak
metadata:
  name: example-sso  # Your KeyCloak Instance Name
  labels:
    app: sso
spec:
  instances: 1
  externalAccess:
    enabled: True
```

* Wait for KeyCloak Instance to be created (not critical and if needed you can check that the keycloak instance using the following command:

```bash
$ oc get keycloak <instance> -n <namespace> -o jsonpath='{.status.ready}'
```

* Create a KeyCloakRealm and KeyCloakClient Broker using following Yamls : (Note: Realm should be created before the client)

```yaml
apiVersion: keycloak.org/v1alpha1
kind: KeycloakRealm
metadata:
  labels:
    app: idp # Your Label Selector
  name: idp-test # Name For you Realm
  namespace: idp-keycloak # Where you installed your keycloak Operator
spec:
  instanceSelector:
    matchLabels:
      app: idp
  realm:
    displayName: IDP Realm
    enabled: true
    id: idp-test
    identityProviders:
    - addReadTokenRoleOnCreate: true
      alias: openshift-v4 # Do Not Change tjos
      config:
        baseUrl: https://api.example.com:6443 # OpenShift API URL
        clientId: idp-broker # Client ID
        clientSecret: secret # CLient Secret
      displayName: login with ocp4 # Display Name
      enabled: true
      internalId: idp-broker 
      providerId: openshift-v4
    realm: idp-test # Your Realm Name
```

* Specify here the same clientID and SecretID

  
```yaml
apiVersion: keycloak.org/v1alpha1
kind: KeycloakClient
metadata:
  labels:
    app: idp # App Label for Realm Selector
  name: example-client # Name of Your KeyCloakClient
  namespace: idp-keycloak # Where you installed your keycloak Operator
spec:
  client:
    baseUrl: /applications #Base URL for ArgoCD
    clientId: idp-broker #Define your Client ID
    defaultClientScopes: # Default Scopes
    - openid
    - profile
    - email
    directAccessGrantsEnabled: false 
    implicitFlowEnabled: false
    protocol: openid-connect
    publicClient: false 
    rootUrl: https://argocd-server-customer-gitops.example.com # URL to your ArgoCD  
    secret: secret # Your Client Secret
    standardFlowEnabled: true
  realmSelector:
    matchLabels:
      app: idp # Same App Label for Realm Selector
  scopeMappings: {}

```

* Configure ArgoCD Secret and CR using following Yamls :

```bash
$ oc edit secret <argocd-name>-secret -n <customer-namespace>
```

```yaml 
apiVersion: v1
data:
  oidc.keycloak.clientSecret: c2VjcmV0 ## Add this line to the existing argocd-secret with your client secret encrypted in Base64
kind: Secret
metadata:
  labels:
    app.kubernetes.io/managed-by: argocd
    app.kubernetes.io/name: argocd-secret
    app.kubernetes.io/part-of: argocd
  name: argocd-secret
  namespace: customer-gitops
type: Opaque

```

```bash
$ oc edit argocd -n <customer-namespace>
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: argocd
  namespace: customer-gitops
spec:
  grafana:
    enabled: false
    ingress:
      enabled: false
    route:
      enabled: false
  # Configure here your OIDC Config from your REALM 
  ## name: Display 
  ## issuer: URL for your REALM
  ## clientID: Your Client ID
  ## clientSecret: Your client Secret stored in an OpenShift Secret: see argocd-yaml example 
  ## RequestedScopres: Replace with Your Scopes
  oidcConfig: "name: OpenShift Single Sign-On   
               issuer: https://keycloak-idp-keycloak.example.com/auth/realms/idp-test  
              clientID: idp-broker
              clientSecret: $oidc.keycloak.clientSecret 
              requestedScopes: [\"openid\", \"profile\", \"email\"]"
  prometheus:
    enabled: false
    ingress:
      enabled: false
    route:
      enabled: false
  server:    
    autoscale:
      enabled: false
    grpc:
      ingress:
        enabled: false
    ingress:
      enabled: false
    route:
      enabled: true
```

* Finally add the OauthClient :

```yaml
apiVersion: oauth.openshift.io/v1
grantMethod: prompt
kind: OAuthClient
metadata:
  name: idp-broker # Name of your oauthclient
redirectURIs:
- https://keycloak-idp-keycloak.example.com/auth/realms/idp-test/broker/openshift-v4/endpoint # Replace idp-test by your Realm Name
secret: secret # Client Secret
```

### Contributing

Anyone interested in contributing and maintaining Yaml Files and the documentation, feel free to fork the project, create your branch and create a pull-request.
