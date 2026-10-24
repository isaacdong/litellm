# Credential boundary

Every provider that lets a caller pass credentials per request has the same hole. A request can pair a resource it controls, such as `api_base`, `scope`, `authority`, `project_id` or `space_id`, with a credential the host owns, and the proxy then sends the host's secret to a destination the caller picked. Azure native auth and Azure AI OCR each hand-rolled this check with slightly different rules. Watsonx, Vertex AI OCR and every tokenization host coming after them need it too, so `src/boundary.rs` holds the one rule and providers only choose which inputs it applies to

## The rule

Inputs arrive as `Sourced<T>`, a value plus the `InputSource` the host assigned to it: `Request`, `Deployment` or `Environment`. `CredentialBoundary::new` takes the sources of every credential component and accepts a bundle that is entirely request-owned or entirely host-owned. `validate_resources` then rejects any request-controlled resource once the credentials are host-owned

```rust
use litellm_auth_types::{CredentialBoundary, InputSource, Sourced};

let tenant = Sourced::new("t", InputSource::Deployment);
let secret = Sourced::new("s", InputSource::Environment);
let scope = Sourced::new("https://attacker.example/.default", InputSource::Request);

let boundary = CredentialBoundary::new(&[tenant.source(), secret.source()])?;
assert_eq!(boundary.source(), InputSource::Deployment);
boundary.validate_resources([scope.source()])  // Err(UntrustedCredentialContext)
```

| credentials | resource | result |
| --- | --- | --- |
| all request | request | ok, the caller spends its own credential |
| all request | deployment or environment | ok |
| deployment, environment or both | deployment or environment | ok, `source()` prefers `Deployment` |
| deployment or environment | request | rejected |
| request mixed with deployment or environment | any | rejected |
| empty | any | rejected, an empty bundle authorizes nothing |

`host_only` is `new` plus a rejection of request-owned bundles. Use it for identities a caller must never supply: ambient credentials, client assertions and host file references

The error is the unit struct `UntrustedCredentialContext`. It carries no values or secrets, and each provider maps it onto its existing public error so no error contract changes

## Provider usage

Azure native client-secret auth, in `auth-azure/src/native.rs`

```rust
let boundary = CredentialBoundary::new(&[tenant_id.source(), client_id.source(), client_secret.source()])
    .map_err(|_| mixed_sources())?;
boundary
    .validate_resources(std::iter::once(scope.source()).chain(authority.as_ref().map(Sourced::source)))
    .map_err(|_| mixed_sources())?;
```

Azure AI OCR, in `llms/src/azure_ai/common_utils.rs`, has a single api-key credential and one protected resource

```rust
CredentialBoundary::new(&[credential_source])
    .and_then(|boundary| boundary.validate_resources(endpoint_source))
```

Watsonx, in the next stack layer, protects `api_base`, `project_id`, `space_id` and `api_version`, and reads the sources of every supplied alias before precedence resolution can drop one

Pass every supplied spelling of a protected input, not only the one that wins precedence. A missing deployment value never authorizes a request override, and an equal request value is still request-controlled. Run the check before credential acquisition or any provider IO. It validates provenance only, so URL safety, credential completeness and upstream permissions stay with the provider

## Host rules

Hosts assign sources and must not let a request forge them. HTTP JSON never carries an `InputSource` or an authorization flag. Python proxy and Rust HTTP hosts label everything the request supplied as `Request`, including aliases, nested fields and connection headers. SDK callers and deployment config produce `Deployment`, environment lookup produces `Environment`. A Router keeps per-call overrides apart from deployment credentials until this check runs. Existing opt-ins, `general_settings.allow_client_side_credentials` and deployment `configurable_clientside_auth_params`, are decided by the host from server-owned configuration before it constructs `Sourced` inputs, and this crate neither reads nor grants them
