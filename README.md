# Fabric / Power BI Key Rotation

Automatically pushes rotated Google Cloud service account keys into Microsoft Fabric / Power BI gateway data sources.

When a service account key stored in **Google Cloud Secret Manager** is rotated, this Cloud Function picks up the change and updates the credentials on one or more Power BI gateway data sources. Nobody has to paste the new key into the Power BI portal by hand, and refreshes keep working after a rotation.

## How it works

```text
Secret Manager ──(secret event)──► Pub/Sub topic ──► Cloud Function (main.main)
                                                          │
                                                          ├─ 1. Read latest secret version from Secret Manager
                                                          ├─ 2. Get an Entra ID (Azure AD) token via MSAL (service principal)
                                                          ├─ 3. GET the gateway (and its public key) from the Power BI REST API
                                                          ├─ 4. Encrypt credentials with the gateway public key (on-prem gateways)
                                                          └─ 5. PATCH each configured gateway data source with the new credentials
```

1. **Trigger:** Secret Manager publishes [event notifications](https://cloud.google.com/secret-manager/docs/event-notifications) to a Pub/Sub topic, and the function is subscribed to that topic. `main.main(event, context)` reads the `eventType`, `secretId` and `versionId` message attributes.
2. **Event handling:**

   | Event | Action |
   | --- | --- |
   | `SECRET_VERSION_ADD`, `SECRET_UPDATE`, `SECRET_VERSION_ENABLE` | Rotate credentials on all configured data sources |
   | `SECRET_VERSION_DISABLE` | Log (info) |
   | `SECRET_VERSION_DESTROY` | Log (warning) |
   | `SECRET_DELETE` | Log (error) and fail |
   | Message without the expected attributes | Log and exit, no rotation |

3. **Credentials:** the latest secret version is expected to be a service account key JSON. The function sends:
   - **username:** the key's `client_email`
   - **password:** the full key JSON (newlines escaped)

   These go to the data source as `Basic` credentials with privacy level `Organizational`.
4. **Power BI update:** the function authenticates to Power BI as an Entra ID service principal, fetches the gateway definition and calls [Gateways - Update Datasource](https://learn.microsoft.com/rest/api/power-bi/gateways/update-datasource) for each data source ID. For on-premises gateways the credentials are encrypted with the gateway's RSA public key (RSA-OAEP, or hybrid AES/HMAC for keys larger than 1024 bits) before they are sent.

## Project layout

```text
keyrotation/
├── main.py              # Cloud Function entry point (Pub/Sub event handler)
├── get_secret.py        # Reads the secret ID from the event and fetches the latest secret version
├── update_creds.py      # Orchestrates token → gateway lookup → credential update
├── config.py            # Power BI / Entra ID settings (service principal secret loaded from Secret Manager)
├── app.py               # Holds the loaded config
├── utils.py             # Config validation and credential serialisation
├── logging_config.json  # Logging configuration
├── requirements.txt
├── services/
│   ├── aadservice.py               # MSAL token acquisition (ServicePrincipal or MasterUser)
│   ├── getdatasource.py            # Power BI gateway / data source lookups
│   ├── updatecredentialsservice.py # PATCH data source credentials
│   ├── addcredentialsservice.py    # Create a gateway data source (not used by the rotation flow)
│   ├── asymmetrickeyencryptor.py   # Chooses the encryption helper based on the gateway key size
│   ├── datavalidationservice.py    # Request validation
│   └── cloudlogger.py              # Google Cloud Logging setup
├── helper/              # RSA / AES credential encryption (ported from the Power BI C# SDK)
└── models/              # Request body models for the Power BI REST API
```

## Configuration

These environment variables are read when the function starts:

| Variable | Description |
| --- | --- |
| `PBI_TENANT_ID` | Entra ID tenant that hosts the service principal and the Power BI tenant |
| `AZ_CLIENT_ID` | Application (client) ID of the Entra ID app registration |
| `AZ_CLIENT_SECRET_ID` | Secret Manager resource name holding the app's client secret, e.g. `projects/<project>/secrets/<name>`. The latest version is used. |
| `PBI_GATEWAY_ID` | ID of the Power BI gateway that owns the data sources |
| `PBI_DATASOURCE_IDS` | Comma-separated list of gateway data source IDs to update |

Authentication mode, scopes and API URLs are set in [keyrotation/config.py](keyrotation/config.py). The default is `ServicePrincipal`.

## Prerequisites

### Microsoft side

- An Entra ID app registration (service principal) with a client secret.
- [Service principals allowed to use Power BI APIs](https://learn.microsoft.com/power-bi/developer/embedded/embed-service-principal) in the Fabric / Power BI admin portal.
- The service principal added as an **admin** on the gateway / data sources it needs to update.

### Google Cloud side

- The service account key (the secret being rotated) stored in Secret Manager, with **event notifications** configured to a Pub/Sub topic.
- The Entra ID client secret stored in Secret Manager.
- A runtime service account for the Cloud Function with `roles/secretmanager.secretAccessor` on both secrets and `roles/logging.logWriter`.

## Deployment

Deploy as a Pub/Sub-triggered Cloud Function from the `keyrotation/` directory, for example:

```bash
cd keyrotation
gcloud functions deploy pbi-key-rotation \
  --runtime=python311 \
  --entry-point=main \
  --trigger-topic=<secret-events-topic> \
  --service-account=<function-sa>@<project>.iam.gserviceaccount.com \
  --set-env-vars=PBI_TENANT_ID=<tenant-id>,AZ_CLIENT_ID=<client-id>,AZ_CLIENT_SECRET_ID=projects/<project>/secrets/<pbi-client-secret>,PBI_GATEWAY_ID=<gateway-id>,PBI_DATASOURCE_IDS=<ds-id-1>,<ds-id-2>
```

> Note: `PBI_DATASOURCE_IDS` contains commas, so you may need gcloud's [alternate delimiter syntax](https://cloud.google.com/sdk/gcloud/reference/topic/escaping) (e.g. `--set-env-vars=^@^PBI_DATASOURCE_IDS=a,b@PBI_GATEWAY_ID=...`) or an `--env-vars-file`.

To test, add a new version to the monitored secret and check Cloud Logging for `Finished updating datasource <id>`.

## Acknowledgements

The Power BI API client, credential serialisation and encryption helpers are adapted from Microsoft's MIT-licensed Power BI developer samples. The encryption helpers are ports of the [PowerBI-CSharp](https://github.com/microsoft/PowerBI-CSharp) SDK extensions.

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft trademarks or logos is subject to and must follow Microsoft's Trademark & Brand Guidelines. Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship. Any use of third-party trademarks or logos are subject to those third-party's policies.
