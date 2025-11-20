# Auto Labeler Admission Webhook

This project provides a Kubernetes admission webhook that enforces consistent labeling for namespaces and (later) workloads. Many organizations already maintain a source of truth for team ownership, asset identifiers, and cost centers. The webhook consumes that data and guarantees that every created namespace receives the proper labels so logging, monitoring, Kubecost, and other tooling can reliably attribute resources to the right team.

## What It Does
- Watches namespace creation requests and looks up the owning team in your chosen directory (JSON file, ServiceNow, etc.).
- Creates a `NamespaceOwner` custom resource per namespace and applies labels such as team name, asset ID, and asset group ID.
- Surfaces actionable feedback when the directory cannot be reached or when a namespace is not registered.
- Provides a `LabelerTeamInfo` custom resource so platform teams can describe how to query their source of truth without rebuilding the webhook.
- Can optionally block namespace creation (`enforce: true`) when an owner record cannot be found.

## Custom Resources
Both CRDs are part of the `labeler.nailabx.com` API group.

### NamespaceOwner
Represents ownership metadata for a namespace and mirrors the labels applied to it.

| Field | Type | Description |
| --- | --- | --- |
| `id` | UUID | Auto-generated identifier. |
| `name` | string | Namespace name (required). |
| `teamName` | string | Owning team (optional). |
| `assetId` | string | Team asset identifier (optional). |
| `assetGroupId` | string | Project or asset group identifier (optional). |
| `metadata` | object | Arbitrary extra data about the team. |

### LabelerTeamInfo
Instructs the webhook on how to query your directory of teams.

| Field | Type | Description |
| --- | --- | --- |
| `url` | string | Endpoint or file URL that returns team info (required). |
| `method` | string | HTTP method to use when calling `url` (default `GET`). |
| `enforce` | boolean | Reject namespace creation when the lookup fails (default `false`). |
| `headers` | map | Extra headers required to fetch the data (optional). |
| `headers.apikey` | string | Inline API key if the endpoint is protected. |
| `headers.apikeyHeaderKey` | string | Header name to hold the API key. |
| `headers.apikeyFromSecret` | string | Reference to a Kubernetes secret that stores the API key. |
| `retrievalOption` | enum | `Loop` (default), `Path`, or `Query`. Determines how the namespace is used to find a team record. |
| `queryKey` | string | Query-string key used when `retrievalOption` is `Query`. |

Retrieval options:
- **Loop**: Fetch a list once and iterate to find the namespace.
- **Path**: Call `{url}/{namespace}` directly.
- **Query**: Call `{url}?{queryKey}={namespace}`.

## Team Directory options
Different organizations manage team inventory differently. Two common scenarios are supported today:

1. **Static JSON**: host a file in GitHub, GitLab, or any static endpoint. Example schema:

    ```json
    {
      "teams": [
        {
          "namespace": "team-a-namespace",
          "teamName": "teamA",
          "assetId": "1234",
          "applicationName": "1234",
          "metadata": {
            "assetGroupName": "",
            "members": [
              { "name": "John Doe", "role": "Lead developer" }
            ]
          }
        }
      ]
    }
    ```

    Fetching from a private GitHub repository:

    ```sh
    curl -L \
      -H "Accept: application/vnd.github.v3.raw" \
      -H "Authorization: Bearer YOUR_PAT" \
      "https://raw.githubusercontent.com/OWNER/REPO/BRANCH/path/to/teams.json"
    ```

    Header values (API keys, Accept headers, etc.) are stored in the `LabelerTeamInfo` resource, either inline or via referenced secrets.

2. **ServiceNow (planned)**: the same CRD will describe how to query ServiceNow. When schemas differ we will add a mapping layer so the webhook can translate ServiceNow responses into `NamespaceOwner` records.

## Namespace Labeling Workflow
1. Install the Helm chart (or apply manifests) and supply the connection details for your directory:

    ```yaml
    nameOverride: labeler-webhook
    url: https://raw.githubusercontent.com/OWNER/REPO/BRANCH/path/to/teams.json
    method: GET
    retrievalOption: Loop
    enforce: false
    headers:
      apikeyFromSecret: secretName
      Accept: application/vnd.github.v3.raw
    ```

2. The installation step creates a `LabelerTeamInfo` resource populated with the values above.
3. When a developer runs `kubectl create ns <namespace>`, the mutation webhook:
   - Looks up the namespace in the configured directory.
   - Creates or updates the matching `NamespaceOwner` resource.
   - Applies ownership labels to the namespace before it is persisted.
4. Error handling:
   - **Connection error**: namespace is created but no `NamespaceOwner` resource is stored; an event describes the connection issue so you can update credentials and retry.
   - **Namespace not found (enforce: false)**: namespace is created without labels and the event stream explains that the namespace is missing from your directory.
   - **Namespace not found (enforce: true)**: namespace creation is rejected until the namespace is registered in your directory.

## Roadmap
- Workload (Deployment/Pod) labeling that mirrors namespace metadata.
- Additional directory integrations beyond JSON files and ServiceNow.
- Validation webhook to block unregistered namespaces when desired.
