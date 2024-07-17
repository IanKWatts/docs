# Cert-Manager Usage for Self-Serve Certificate Management

## Table of Contents
1. [Summary](#summary)
2. [Overview](#overview)
3. [Requirements](#requirements)
4. [Create an Issuer](#create-an-issuer)
5. [Create a Certificate with Route Integration](#create-with-route)
6. [Create a Certificate, Standalone Method](#create-standalone)
7. [Prepare for Production](#prepare-for-production)
8. [Certificate Renewal](#certificate-renewal)
9. [Troubleshooting](#troubleshooting)
9. [Remove a Certificate](#remove-certificate)
9. [References](#references)

## Summary<a name="summary"></a>
The cert-manager operator allows OpenShift users to request and apply TLS certificates to their web applications using automation, including GitOps.

## Overview<a name="overview"></a>
The cert-manager operator allows users to create and manage their own TLS certificates by way of custom resources in their namespaces.  A user can create an `Issuer` resource, which tells the operator how to request certificates.  Certificates may be requested and automatically applied to a Route in a single process or they may be requested and stored in a Secret from which to be applied as needed or for multiple Routes.

To create a certificate and apply it to a Route in one step, the user adds annotations to the Route, specifying the Issuer to use and other information for the certificate.  The operator uses that information to interact with the certificate authority, automatically manages the certificate request and verification steps, and applies the certificate to the Route.

To create a certificate separately from a Route, the user creates a Certificate resource.

## Requirements<a name="requirements"></a>
* NetworkPolicy that allows traffic into the namespace from the Internet
* NetworkPolicy that allows traffic within the namespace
* Account with certificate provider, if using a non-free service
* Outbound access to the certificate authority (NSX clusters)
* For testing: an application with a Service and a Route

## Create an Issuer<a name="create-an-issuer"></a>
An Issuer defines how cert-manager will request TLS certificates. They are specific to a single namespace, so if you use certificates in multiple namespaces, you will need to create the Issuer in each of them.

The Issuer CRD is provided by the 'cert-manager' operator and defines the type of issuer, which is one of acme, ca, selfSigned, vault, or venafi.  For Let's Encrypt, the type is 'acme'.  It will be configured with a contact email address, the name of the Secret for storing the issuer certificate, the URL of the issuer, and the verification method.  For other issuer types, consult the [cert-manager Issuer documentation](https://cert-manager.io/docs/configuration/).

### Example Issuer using Let's Encrypt
**Important note about Let's Encrypt:** Because Let's Encrypt has _strict rate limits_ on certificate requests, use their staging service for any development and testing; only use their production service when ready to configure your own production environment.

Start by creating an Issuer for the Let's Encrypt staging environment.  Set the email field to a suitable value.  Example filename: `issuer.lets-encrypt-staging.yaml`
```
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: lets-encrypt-staging
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: YOUR_EMAIL_ADDRESS
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: lets-encrypt-staging
    # Enable the HTTP-01 challenge provider
    solvers:
      - http01:
          ingress:
            serviceType: ClusterIP
```
_Note: The default serviceType for .spec.solvers.http01.ingress is NodePort.  In our Private Cloud clusters, the serviceType must be set to ClusterIP._

Create the Issuer: `oc apply -f issuer.lets-encrypt-staging.yaml`

After creation, check on the status of the issuer:
```
$ oc get issuer
NAME                   READY   AGE
lets-encrypt-staging   True    4m

$ oc describe issuer lets-encrypt-staging
Name:         lets-encrypt-staging
Namespace:    e95e89-dev
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Issuer
Metadata:
  Creation Timestamp:  2024-07-03T16:43:04Z
  Generation:          2
  Resource Version:    2933692963
  UID:                 3bc428d9-d6b7-4733-92c4-cc80a0e3d88d
Spec:
  Acme:
    Email:  ian.1.watts@gov.bc.ca
    Private Key Secret Ref:
      Name:  lets-encrypt-staging
    Server:  https://acme-staging-v02.api.letsencrypt.org/directory
    Solvers:
      http01:
        Ingress:
          Service Type:  ClusterIP
Status:
  Acme:
    Last Private Key Hash:  lDuSkAfBrlPmL3c+qe/Cgunk9E9qf/sBI2szo3/dlqA=
    Last Registered Email:  ian.1.watts@gov.bc.ca
    Uri:                    https://acme-staging-v02.api.letsencrypt.org/acme/acct/154489483
  Conditions:
    Last Transition Time:  2024-07-03T16:43:04Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   2
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```
If the Issuer is not Ready, check the status message for more information.

## Create a Certificate with Route Integration<a name="create-with-route"></a>
Assuming you have a Service and Route already prepared, begin the certificate creation process by modifying your Route.

Add annotations to your Route.  Only the `issuer` field is required.  Other information can be included as needed.  For a full list of options, see the [openshift-routes documentation](https://github.com/cert-manager/openshift-routes/tree/main).
```
  annotations:
    cert-manager.io/issuer: lets-encrypt-staging                     # Required: the name of your Issuer
    cert-manager.io/subject-organizations: "CITZ"                    # Optional, no default
    cert-manager.io/subject-organizationalunits: "Platform Services" # Optional, no default
    cert-manager.io/subject-countries: "Canada"                      # Optional, no default
    cert-manager.io/subject-provinces: "BC"                          # Optional, no default
    cert-manager.io/subject-localities: "Victoria"                   # Optional, no default
    cert-manager.io/subject-streetaddresses: "808 Douglas Street"    # Optional, no default
```

### Check order status
After the Route has been modified, the operator will create several custom resources:
* CertificateRequest - Used to request a signed certificate from one of the configured issuers
* Order - Represents an Order with an ACME server
* Challenge - Represents a Challenge request with an ACME server

Check on your order by first getting its name, then reviewing its status.
```
$ oc get order
$ oc describe order <order_name>
<snip>
Events:
  Type    Reason    Age  From                 Message
  ----    ------    ---  ----                 -------
  Normal  Complete  34m  cert-manager-orders  Order completed successfully

$ oc get certificaterequest
NAME          APPROVED   DENIED   READY   ISSUER                 REQUESTOR                                                          AGE
nginx-56hgc   True                True    lets-encrypt-staging   system:serviceaccount:cert-manager:cert-manager-openshift-routes   34m
```
If the order is not Approved and Ready, check the status message from 'oc describe' for more information.

### Test your route
If you have used the Let's Encrypt **staging** service, you will see a warning about an untrusted certificate, but this is sufficient for testing your setup.  Verify that you can connect to the URL of your Route using HTTPS.

### Certificate-related resources
After successful application of the certificate, the `CertificateRequest` and `Order` resources will still exist in your namespace, but the `Challenge` resource will be removed.

## Create a Certificate, Standalone Method<a name="create-standalone"></a>
Unlike the scenario described above, a certificate can be created by itself without integration with a Route.  This is done by creating a Certificate resource.  Because the operator will not be getting information from a Route for certain certificate parameters, all necessary information is set in the Certificate manifest.

For details on Certificate creation, see the [cert-manager Certificate documentation](https://cert-manager.io/docs/usage/certificate/).

After the certificate is procured, a Secret will be created having the name set in the Certificate's `.spec.secretName` field and its public and private keys will be added to the Secret.

Verify successful certificate creation by checking the output of `oc get orders` and `oc get certificates`, and check the contents of the relevant Secret.

## Prepare for Production<a name="prepare-for-production"></a>
For users of Let's Encrypt, once you have verified the configuration, create an Issuer for prod.  Here is an example of an Issuer using the production Let's Encrypt service.  Note that the server URL is different.
```
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: lets-encrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: YOUR_EMAIL_ADDRESS
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: lets-encrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
      - http01:
          ingress:
            serviceType: ClusterIP
```

To use the production Issuer in a Route, make sure to set the 'issuer' annotation to the name of the production Issuer.  Other annotations could be the same as in your non-prod environments.

`cert-manager.io/issuer: lets-encrypt-prod`

## Certificate Renewal<a name="certificate-renewal"></a>
cert-manager automatically renews certificates based on the duration of the certificate.  For more information, see [Issuance triggers](https://cert-manager.io/docs/usage/certificate/#reissuance-triggered-by-expiry-renewal).

## Troubleshooting<a name="troubleshooting"></a>
Depending on your issuer, there can be a variety of reasons for failure of certificate procurement or application.  Check the [troubleshooting docs](https://cert-manager.io/docs/troubleshooting/) for more information.

## Remove a Certificate<a name="remove-certificate"></a>
A standalone certificate can be removed by deleting the Certificate resource, then deleting the associated Secret.  The Secret is not automatically deleted.

To remove a certificate that was created by Route integration, delete the Route.  The Route may be recreated without the cert-manager annotations.

## References<a name="references"></a>
##### cert-manager
https://cert-manager.io/docs/

##### Troubleshooting
https://cert-manager.io/docs/troubleshooting/

##### OpenShift Route Support for cert-manager
https://github.com/cert-manager/openshift-routes

