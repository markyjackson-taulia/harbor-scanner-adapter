# Harbor Scanner Adapter for Anchore Engine/Enterprise

## Overview

The Harbor Scanner Adapter for Anchore is a service that translates the Harbor scanning API into the Anchore API
and allows Harbor to utilize Anchore Engine (or Enterprise) for providing vulnerability reports on images stored in Harbor as
part of its vulnerability scan feature.

This Adapter is only required if you want Harbor to use Anchore for its 'Scan' feature. If your objective is to use Anchore in a CI/CD
workflow or to provide its own analysis reports against images stored in Harbor, that can be achieved without this Adapter.


![Interaction Overview](assets/AnchoreHarborAdapter.png)

### Authentication

#### Harbor-to-Adapter

API requests between Harbor and the adapter should be protected by TLS/HTTPS at the channel level, but can also use a shared secret for authentication.
See the configuration section below on setting an API key for the Adapter. Then, when adding the adapter to Harbor via the UI, use the 'Bearer' auth type
and set the value to the value set in the adapter configuration. This will cause harbor to send they token as a Bearer token to the adapter, which will verify
the HTTP Authorization header against the expected value.

#### Adapter-To-Anchore

API calls between the Adapter and Anchore should also be protected by TLS/HTTPS and also require a username/password for HTTP Basic authentication to Anchore. For this, it 
is highly recommended to create a specific harbor user in Anchore and use those credentials in the auth secret presented to the Adapter as described below in the configuration section.

Note that these are not credentials use to pull images from Harbor, only to authenticate API calls between the adapter service and the Anchore API service.

#### Anchore-To-Harbor

API requests from Anchore services to Harbor to fetch the image data for analysis are protected by short-lived credentials generated by Harbor and passed to the Adapter service on each
new scan request. The Adapter service automatically configures Anchore to use the supplied credentials for the data fetch operations as part of requesting an image analysis to Anchore.

The adapter never makes requests to Harbor and never reads any image content, only Anchore services access any Harbor image data.

## Building the Adapter service

Run `make` to get the binary in bin/scanner-anchore
```
make
```

To build into a container: 
```
make container
```


## Configuration

Configuration of the adapter is done via environment variables at startup or via a configuration file. If both are used,
environment variable values take precedent over values from the file.

### Environment Variables

Adapter configuration values. These cannot be set in the client config file, only via environment variables.

| Env Var | Description | Example Value |
| --------|-------------|---------------|
| SCANNER_ADAPTER_LISTEN_ADDR | The address for the scanner to listen on for API calls | ":8080" |
| SCANNER_ADAPTER_APIKEY | A key value to enable authentication for the adapter API. If set, callers must provide this value as a bearer token in the Authorization header | 7d341116-99e4-4c9b-bd81-87cd4117e713 |


Anchore client configuration (also available via a json file as below):

| Env Var | Description | Example Value |
| --------|-------------|---------------|
| ANCHORE_AUTHFILE_PATH | The path to find the config file to load. No default, if omitted a file is not expected or used | /anchore_client.json |
| ANCHORE_ENDPOINT | API endpoint for calls to Anchore  | http://localhost:8228 |
| ANCHORE_USERNAME | Anchore username for api calls     | harborscanneruser |
| ANCHORE_PASSWORD | Credential for the anchore user    | foobar |
| ANCHORE_CLIENT_TIMEOUT_SECONDS | Timeout value for all API calls to Anchore | 10 |
| ANCHORE_FILTER_VENDOR_IGNORED | Boolean to ignore all vulnerabilities explicitly ignored by distros/vendors (e.g. Debian CVEs marked No-DSA). If unset default = false and all are returned | true


### Configuration file format

The configuration file must be json formatted.

```
{
  "endpoint": "http://anchore-anchore-engine-api.default.svc.cluster.local:8228",
  "username": "harbor",
  "password": "harboruserpass123",
  "timeoutSeconds": 10,
  "filterVendorIgnoredVulns": false
} 
```

## Deployment

### Requirements

This adapter requires an Anchore Engine or Enterprise deployment to operate against. The adapter can be deployed before the anchore installation,
but the endpoint url and credentials must be known to pass to the adapter.

It is highly recommended to create a new account in the anchore deployment and a new user with credentials dedicated to the Harbor adapter.


### Kubernetes

Install Harbor:

```
helm install --name harbor harbor/harbor
```

Install Anchore Engine or Enterprise (Engine as example):

**NOTE: this adapter requires Anchore Engine v0.5.1 or newer**

```
helm install --name anchore stable/anchore-engine
```

Create a harbor account and user in Anchore for the adapter to use for authenticating calls to Anchore. 

_This is not required, but recommended as it will limit the scope of the adapter's credentials within Anchore to a 
non-admin account and keep all Harbor image data in one account for easy management. This step can be skipped for demo environments where the Anchore Engine deployment is not shared, but is strongly encouraged for all production use and all cases where
Harbor will be integrated with an existing Anchore Engine/Enterprise deployment._

```
ANCHORE_CLI_USER=admin
ANCHORE_CLI_PASS=$(kubectl get secret --namespace default anchore-anchore-engine -o jsonpath="{.data.ANCHORE_ADMIN_PASSWORD}" | base64 --decode; echo)
ANCHORE_CLI_URL=http://anchore-anchore-engine-api.default.svc.cluster.local:8228/v1/
kubectl run -i --tty anchore-cli --restart=Always --image anchore/engine-cli  --env ANCHORE_CLI_USER=admin --env ANCHORE_CLI_PASS=${ANCHORE_CLI_PASS} --env ANCHORE_CLI_URL=http://anchore-anchore-engine-api.default.svc.cluster.local:8228/v1/
```

In the anchore-cli container running inside the cluster, now create a new account in Anchore, _harbor_ with a single user _harbor_ and password for this example (use a much stronger password in your install)

```
anchore-cli account add harbor
anchore-cli account user add --account harbor harbor harboruserpass123

```

Install the adapter:

```
cat > anchore-config.json << EOF
{
  "endpoint": "http://anchore-anchore-engine-api.default.svc.cluster.local:8228",
  "username": "harbor",
  "password': "harboruserpass123",
  "timeoutSeconds": 10
} 
EOF

kubectl create secret generic --name anchore-adapter-config --from-file=anchore-config=anchore-config.json
kubectl apply -f ./k8s/harbor-adapter-anchore.yaml
```

Configure the scanner in Harbor:

In the Harbor UI login as an admin and navigate to Configuration>Scanners and click "+ New Scanner".

![Add Scanner UI](assets/scanner-config.png)

In 'Endpoint', type: "http://harbor-scanner-anchore:8080"

Leave authorization empty since in this example we did not set an API key in the adapter deployment environment.

Click "Test Connection" and should work. 
You can now click "Add" to complete the setup.

## Special Thanks

Special thanks to the following for their help with the prototype adapter implementation that inspired this work:

* [cafeliker](https://github.com/cafeliker)
* [MaGaoJU](http://github.com/MaGaoJu)


