# Quay Enterprise OIDC Auth with Tectonic Identity 

## Export certificate information from cluster for TLS verification of Tectonic Identity: 

```
kubectl -n tectonic-system get secret tectonic-ca-cert-secret -o json | jq -r '.data[]' | base64 -d >> tectonic-ca-ingress.crt
kubectl -n tectonic-system get secret tectonic-ingress-tls-secret -o json | jq -r '.data["tls.crt"]' | base64 -d >> tectonic-ca-ingress.crt
```
## Upload Cert to QE

Upload the `tectonic-ca-ingress.crt` certificate to QE via the superuser panel or follow the [documentation](https://coreos.com/quay-enterprise/docs/latest/insert-custom-cert.html) upload manually. 

## Configure Tectonic Identity

Edit the tectonic-identity configmap: 

```
kubectl -n tectonic-system edit cm tectonic-identity
```

Add an client definition under the `staticClients` section for Quay Enterprise: 

```
  - id: quay-enterprise
    name: Quay Enterprise
    secret: rEpLaCeThIs
    redirectURIs:
      - ''
```

`secret:` is a user generated string that will later be passed to Quay Enterprise. Use `date | md5sum` or `openssl rand -base64 15` to generate a random string. 
`redirectURIs:` URLs that must be passed to the tectonic-identity configmap. These are provided by Quay Enterprise in the next step.

Leave this terminal open for use later

## Add an OIDC Provider for Tectonic Identity

From Quay Enterprise, navigate to the superuser panel, then to External Authorization, click the button: `Add OIDC Provider`, and type `tectonicidentity` in the prompt.

`ODIC Server:` is the `issuer:` parameter from the tectonic-identity configmap with a forwardslash at the end: 'https://tectonic.example.com/identity/'
`Client ID`: `quay-enterprise`
`Client Secret:` String passed as `secret` to tectonic-identity configmap
`Service Name:`	will be displayed on the login page. Suggested Value: `Tectonic Identity`

## Configure Redirect URIs

Navigate back to editing the `tectonic-identity` configmap. Add the provided CallbackURLs as redirectURIs:

```
  - id: quay-enterprise
    name: Quay Enterprise
    secret: rEpLaCeThIs
    redirectURIs:
      - 'https://reg.example.com/oauth2/tectonicidentity/callback'
      - 'https://reg.example.com/oauth2/tectonicidentity/callback/attach'
      - 'https://reg.example.com/oauth2/tectonicidentity/callback/cli'
```


Save the changes to the `tectonic-identity` configmap and issue a `kubectl patch` to update the deployment: 

```
kubectl --namespace tectonic-system patch deployment tectonic-identity \
    --patch "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}"
```

Navigate back to the Quay Enterprise superuser panel click "Save Configuration Changes", then "Save Configuration, and finally restart the Quay Enterprise container with a `docker restart` or `kubectl delete pod <quay-enterprise>`. 






