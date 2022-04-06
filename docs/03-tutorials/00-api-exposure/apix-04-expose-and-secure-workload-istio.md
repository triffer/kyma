---
title: Expose and secure a workload with Istio
---

This tutorial shows how to expose and secure a workload using Istio built-in security features. You will expose the workload by creating a [Virtual Service](https://istio.io/latest/docs/reference/config/networking/virtual-service/). Then, you will secure access to your workload by adding the JWT validation verified by the Istio security configuration with [Authorization Policy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) and [Request Authentication](https://istio.io/latest/docs/reference/config/security/request_authentication/).

## Prerequisites

To follow this tutorial, use Kyma 2.0 or higher.

This tutorial is based on a sample HttpBin service deployment and a sample Function. To deploy or create one of those, follow the [Create a workload](./apix-02-create-workload.md) tutorial. It can also be a follow up to the [Use a custom domain to expose a workload](./apix-01-own-domain.md) tutorial.

## Get a JWT

1. In your OpenID Connect-compliant (OIDC-compliant) identity provider, create an application to get your client credentials such as Client ID and Client Secret. Export your client credentials as environment variables. Run:

   ```bash
   export CLIENT_ID={YOUR_CLIENT_ID}
   export CLIENT_SECRET={YOUR_CLIENT_SECRET}
   ```

2. Encode your client credentials and export them as an environment variable:

   ```bash
   export ENCODED_CREDENTIALS=$(echo -n "$CLIENT_ID:$CLIENT_SECRET" | base64)
   ```

3. In your browser, go to `https://YOUR_OIDC_COMPLIANT_IDENTITY_PROVIDER_INSTANCE/.well-known/openid-configuration`, save values of the **token_endpoint**, **jwks_uri** and **issuer** parameters, and export them as environment variables:

   ```bash
   export TOKEN_ENDPOINT={YOUR_TOKEN_ENDPOINT}
   export JWKS_URI={YOUR_JWKS_URI}
   export ISSUER={YOUR_ISSUER}
   ```

4. Get the JWT:

   ```bash
   curl -X POST "$TOKEN_ENDPOINT" -d "grant_type=client_credentials" -d "client_id=$CLIENT_ID" -H "Content-Type: application/x-www-form-urlencoded" -H "Authorization: Basic $ENCODED_CREDENTIALS"
   ```

5. Save the token and export it as an environment variable:

   ```bash
   export ACCESS_TOKEN={YOUR_ACCESSS_TOKEN}
   ```

## Expose your workload using a Virtual Service

Follow the instructions in the tabs to expose the HttpBin workload or the Function using a Virtual Service.

<div tabs>

  <details>
  <summary>
  Expose the HttpBin workload
  </summary>

1. Export your domain name and gateway name as environment variables:

   ```shell
   export DOMAIN_TO_EXPOSE_WORKLOADS={DOMAIN_NAME} # This is a Kyma domain or your custom subdomain, for example: api.mydomain.com.
   export GATEWAY=$NAMESPACE/httpbin-gateway # If you don't want to use your custom domain but a Kyma domain, use the following Kyma Gateway: `kyma-system/kyma-gateway`.
   ```

2. Create a Virtual Service:

   ```shell
   cat <<EOF | kubectl apply -f -
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: httpbin
     namespace: $NAMESPACE
   spec:
     hosts:
     - "httpbin.$DOMAIN_TO_EXPOSE_WORKLOADS"
     gateways:
     - $GATEWAY
     http:
     - match:
       - uri:
           prefix: /
       route:
       - destination:
           port:
             number: 8000
           host: httpbin.$NAMESPACE.svc.cluster.local
   EOF
   ```
  </details>

  <details>
  <summary>
  Expose the Function
  </summary>

1. Export your domain name and gateway name as environment variables:

   ```shell
   export DOMAIN_TO_EXPOSE_WORKLOADS={DOMAIN_NAME} # This is a Kyma domain or your custom subdomain, for example: api.mydomain.com.
   export GATEWAY=$NAMESPACE/httpbin-gateway # If you don't want to use your custom domain but a Kyma domain, use the following Kyma Gateway: `kyma-system/kyma-gateway`.
   ```

2. Create a Virtual Service:

   ```shell
   cat <<EOF | kubectl apply -f -
   apiVersion: networking.istio.io/v1alpha3
   kind: VirtualService
   metadata:
     name: function
     namespace: $NAMESPACE
   spec:
     hosts:
     - "function.$DOMAIN_TO_EXPOSE_WORKLOADS"
     gateways:
     - $GATEWAY
     http:
     - match:
       - uri:
           prefix: /
       route:
       - destination:
           port:
             number: 80
           host: function.$NAMESPACE.svc.cluster.local
   EOF
   ```

  </details>
</div>

## Secure a workload or the Function using a JWT

To secure the Httpbin workload or the Function using a JWT, create a Request Authentication with Authorization Policy. Workloads that have the **matchLabels** parameter specified require a JWT for all requests. Follow the instructions in the tabs:

<div tabs>

  <details>
  <summary>
  Secure the Httpbin workload
  </summary>

1. Create the Request Authentication and Authorization Policy resources:

   ```shell
   cat <<EOF | kubectl apply -f -
   apiVersion: security.istio.io/v1beta1
   kind: RequestAuthentication
   metadata:
     name: jwt-auth-httpbin
     namespace: $NAMESPACE
   spec:
     selector:
       matchLabels:
         app: httpbin
     jwtRules:
     - issuer: $ISSUER
       jwksUri: $JWKS_URI
   ---
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: httpbin
     namespace: $NAMESPACE
   spec:
     selector:
       matchLabels:
         app: httpbin
     rules:
     - from:
       - source:
           requestPrincipals: ["*"]
   EOF
   ```

2. Access the workload you secured. You will get the `403 Forbidden` error.

   ```shell
   curl -ik -X GET https://httpbin.$DOMAIN_TO_EXPOSE_WORKLOADS/status/200
   ```

3. Now, access the secured workload using the correct JWT. You will get the `200 OK` response.

   ```shell
   curl -ik -X GET https://httpbin.$DOMAIN_TO_EXPOSE_WORKLOADS/status/200 --header "Authorization:Bearer $ACCESS_TOKEN"
   ```
  </details>

  <details>
  <summary>
  Secure the Function
  </summary>

1. Create the Request Authentication and Authorization Policy resources:

   ```shell
   cat <<EOF | kubectl apply -f -
   apiVersion: security.istio.io/v1beta1
   kind: RequestAuthentication
   metadata:
     name: jwt-auth-function
     namespace: $NAMESPACE
   spec:
     selector:
       matchLabels:
         app: function
     jwtRules:
     - issuer: $ISSUER
       jwksUri: $JWKS_URI
   ---
   apiVersion: security.istio.io/v1beta1
   kind: AuthorizationPolicy
   metadata:
     name: function
     namespace: $NAMESPACE
   spec:
     selector:
       matchLabels:
         app: function
     rules:
     - from:
       - source:
           requestPrincipals: ["*"]
   EOF
   ```

2. Access the workload you secured. You will get the `403 Forbidden` error.

   ```shell
   curl -ik -X GET https://function.$DOMAIN_TO_EXPOSE_WORKLOADS/status/200
   ```

3. Now, access the secured workload using the correct JWT. You will get the `200 OK` response.

   ```shell
   curl -ik -X GET https://function.$DOMAIN_TO_EXPOSE_WORKLOADS/status/200 --header "Authorization:Bearer $ACCESS_TOKEN"
   ```
  </details>
</div>