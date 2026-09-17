---
stand_alone: true
ipr: none
cat: std
submissiontype: IETF
wg: OpenID AuthZEN

docname: draft-schwartz-authzen-policy-store

title: AuthZEN Policy Store API
abbrev: policy-store-api
lang: en
kw:
 - Authorization
 - API
 - Policy Store
 - PDP
 - AuthZEN
 - CJAR

author:
- role: editor
  ins: M. Schwartz
  name: Michael Schwartz
  org: Gluu
  email: mike@gluu.org
- role: editor
  ins: D. Desai
  name: Dhaval Desai
  org: Gluu
  email: dhaval@gluu.org

contributor:
- name: Victor Moreno
  org: Independent Contributor
  email: victor.moreno@gmail.com

normative:
 RFC8259:
 RFC7519:
 RFC8615:
 RFC2119:
 RFC7578:

informative:
 CEDAR:
   title: Cedar Policy Language
   target: https://docs.cedarpolicy.com/
   author:
   - name: Cedar Team
     org: Amazon Web Services
 CERBOS:
   title: Cerbos Policy Decision Point
   target: https://docs.cerbos.dev/
   author:
   - name: Cerbos Project
     org: Cerbos
 AUTHZEN-API:
   title: Authorization API 1.0
   target: https://openid.net/specs/authorization-api-1_0.html
   author:
   - name: OpenID AuthZEN Working Group
     org: OpenID Foundation
 BCP-14:
   title: BCP 14
   target: https://www.rfc-editor.org/info/bcp14/
   author:
   - name: IETF
     org: IETF      
 AVP:
   title: Amazon Verified Permissions
   target: https://aws.amazon.com/verified-permissions/
   author:
   - name: Amazon Web Services
     org: Amazon Inc.

--- abstract

This document defines the AuthZEN Policy Store API and the Policy Store format for distributing authorization policies and their associated evaluation artifacts.

The Policy Store API is implemented by a conforming Policy Decision Point (PDP). It provides a standard way to supply authorization policies and the artifacts required to evaluate them to a PDP.

The Policy Store format defines a canonical, PDP-neutral directory structure for organizing authorization policies and their associated metadata. It provides a common structure for policies, schemas, default entities, trusted token issuers, custom token issuers, and other artifacts required for policy evaluation. Using a well-known structure reduces the need for PDP-specific configuration. It also improves the discoverability and portability of these artifacts across ecosystem tools like policy authoring tools, management, and deployment tools.

This specification also defines a compressed archive format (.cjar, Constraint JAR) for packaging a Policy Store for distribution, versioning, and deployment. 

--- middle

# Introduction

Organizations often have multiple teams or areas that use different policy languages to define authorization policies. Along with policies, they maintain schemas, default entities, trusted token issuers, and other metadata required to evaluate those policies using Policy Decision Points (PDPs).

Today, these artifacts are typically organized according to the requirements of the particular PDP or policy engine in use. As a result, moving to a different PDP—even one that supports the same policy language—may require restructuring the artifacts or configuring the new PDP to use the existing structure. PDPs and policy engines such as Cedar ({{CEDAR}}), Amazon Verified Permissions ({{AVP}}), and Cerbos ({{CERBOS}}) do not currently share a common structure for organizing, packaging, and versioning these artifacts. This limits portability, auditability, and interoperability between PDPs and ecosystem tools such as policy authoring tools.

The Policy Store API, together with the Policy Store directory structure and .cjar archive format, provides a common way to organize, distribute, and consume authorization policies and their associated evaluation artifacts.

The Policy Store directory structure is independent of any particular PDP or policy engine. It can be used with different policy languages, and conforming PDPs can consume a Policy Store when they support the policy language used by that store.

For example, an organization may use the canonical Policy Store structure for Cedar policies, schemas, and related metadata. A conforming Cedar-based PDP, such as Cedarling or Amazon Verified Permissions, can then consume those artifacts. Similarly, a Policy Store containing CEL policies can be consumed by a conforming PDP that supports CEL.

A Policy Store realization is necessarily specific to the policy language it contains. However, it remains independent of the PDP or policy engine used to evaluate that language.

The OpenID AuthZEN Working Group defines protocols and patterns for authorization interoperability, including the Authorization API ({{AUTHZEN-API}}). This specification complements AuthZEN by defining a Policy Store API and a portable, PDP-neutral, self-contained Policy Store format that conforming PDPs and tools can load, validate, and exchange without relying on proprietary layout conventions.

This document defines version 1.0 of the AuthZEN Policy Store API.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 ({{BCP-14}}) and when, and only when, they appear in all capitals, as shown here.

## Terminology

Policy Store API:
: API implemented by conforming PDPs to accept policy store data

Policy Store API Endpoint:
: Endpoint `/access/v1/policy-store` exposed by PDPs that implement Policy Store API

Policy Store:
: A structured collection of policies, schema, and related artifacts bound together for evaluation, as defined in this document.

Policy Language:
: The policy language to author policies in the policy store (for example, Cedar or CEL). Identified in `metadata.json` by `policy_language` and `policy_language_version`.

Directory Format:
: A policy store represented as a folder hierarchy on a filesystem.

Archive Format:
: A policy store packaged as a ZIP archive with the `.cjar` file extension.

Constraint JAR (CJAR):
: The Archive Format; a ZIP archive whose extension `.cjar` denotes a **constraint jar** — a self-contained package of authorization constraints (policies, schema, and related configuration) for distribution and deployment.

Policy Decision Point (PDP):
: A component that evaluates authorization requests against policies. See {{AUTHZEN-API}}.

Policy Administration Point (PAP):
: A component or system used by administrators to manage the lifecycle of policies and related artifacts, including the policy store.

Custom Token:
: A token that requires specialized processing. For example: non-JWT tokens, API keys, or tokens that use a custom encryption algorithm.

Custom Issuer:
: The issuer that issues one or more custom tokens. These issuers are declared in `custom-issuers/`. Refer to Custom Issuers ({{custom-issuers}}) for more details.

Token Processor:
: The component that validates and processes a custom token. Its interface and registration mechanism are out of scope for this specification.

# Overview

## Policy store API

The policy store API defines a single POST‑only HTTP API endpoint that accepts a **`.cjar`** (Constraint JAR) file containing a policy store. The policy store API specifies `/access/v1/policy-store` endpoint, which MUST be implemented by any PDP that conforms to this specification. This endpoint enables authorized PDP administrators or a PAP system to upload updated versions of the policy stores to the PDP. API does not expose HTTP methods that allow updates or deletion of the existing policy stores. 

## Policy store

A **policy store** bundles together the artifacts that a PDP needs to evaluate authorization requests against a known schema and configuration baseline. These artifacts include:

* Required policy store metadata (`metadata.json`)
* Required Policies (one policy document per file under `policies/`)
* Optional schema (`schema/`)
* Optional policy templates (`templates/`)
* Optional default entities (`entities/`)
* Optional trusted issuer configuration (`trusted-issuers/`)
* Optional custom issuer configuration (`custom-issuers/`)

This document does not define the syntax and semantics of policy documents, schema files, and entity types. These are defined by the declared policy language. This specification defines the container layout, metadata, and interchange formats only.

Implementations MAY support either the Directory Format or the Archive Format(.cjar), or both. Tools that produce or consume policy stores SHOULD support conversion between formats without loss of content.

The Archive Format is a ZIP archive containing the same relative paths as the Directory Format. Archive files MUST use the `.cjar` extension.

# Policy Store API Specification

Policy Store API defines `/access/v1/policy-store` endpoint for uploading the policy store.

## Usage

- Conforming PDPs should implement the `/access/v1/policy-store` endpoint
- A policy administrator or a PAP should use this PDP endpoint to upload the policy store (cjar) to the PDP
- Before uploading the policy store to the PDP, the policy administrator MUST ensure the validity of the policy store and correct version management. 
- PDP implementations MUST not implement policy store lifecycle management capability using this endpoint.

## Relationship to NMOP Policy Sharing Model

Separation of responsibilities between policy store governance and distribution may follow [NMOP policy sharing model draft](https://www.ietf.org/archive/id/draft-cabanillas-nmop-authz-policy-sharing-model-03.html#section-6.2) recommendations. 

## API Request {#api-request}

The **`/access/v1/policy-store`** endpoint is a POST-only HTTP API that accepts a **`.cjar`** (Constraint JAR) file containing a policy store. The request body MUST be a `multipart/form-data` upload with the following part:

* **file**: The `.cjar` archive. 

**Example Request**:

~~~http
POST /access/v1/policy-store HTTP/1.1

Host: api.example.com

Content-Type: multipart/form-data; boundary=---XYZ

-----XYZ

Content-Disposition: form-data; name="file"; filename="store.cjar"

Content-Type: application/zip

<binary .cjar content>

-----XYZ--
~~~

Refer to {{transport}} for more details.


## API Response {#api-response}

The server validates the uploaded file and metadata. On success, it stores the file and returns **201 Created** with a JSON body containing the assigned identifier. On validation failure, it returns **400 Bad Request** with an error description.

**Example Response (Success)**:

~~~json
{
  "id": "store-12345",
  "message": "Policy store uploaded successfully."
}
~~~

**Example Response (Error)**:

~~~json
{
  "error": "Invalid metadata. Version does not match archive."
}
~~~

Refer to ({{transport}}) for more details.

# Policy Store Directory Structure

The root of a policy store (directory or archive) is referred to as the **policy store root**. The following layout is REQUIRED for conformant policy stores:

~~~ ascii-art
policy-store-root/
├── metadata.json
├── policies/
│   └── (one policy document per file)
├── schema/             (optional)
│   └── (schema files; format defined by policy_language)
├── templates/          (optional)
│   └── (one template document per file)
├── entities/           (optional)
│   └── *.json
├── trusted-issuers/    (optional)
│   └── *.json
└── custom-issuers/     (optional)
    └── *.json
~~~
{: title="Policy Store Directory Layout"}

## Required Files and Directories

`metadata.json`:
: REQUIRED at the policy store root. Contains policy store metadata as defined in Metadata ({{metadata}}).

`policies/`:
: REQUIRED directory. Contains one or more policy files. Each file MUST contain exactly one policy document in a format accepted by the declared policy language.

## Optional Directories

`schema/`:
: OPTIONAL. When present, contains one or more schema artifacts to support policy evaluation. File names and formats are defined by the policy language documentation. PDPs that require schema for evaluation MUST reject a policy store that omits `schema/`.

`templates/`:
: OPTIONAL. Contains policy template documents when supported by the policy language. Each file MUST contain exactly one template document.

`entities/`:
: OPTIONAL. Contains default entity definition files as defined in Default Entities ({{default-entities}}).

`trusted-issuers/`:
: OPTIONAL. Contains trusted issuer configuration files as defined in Trusted Issuers ({{trusted-issuers}}).

`custom-issuers/`:
: OPTIONAL. Contains custom issuer configuration files as defined in Custom Issuers ({{custom-issuers}}).

# File Naming and Content Requirements

## Policy and Template Files

Files under `policies/` and `templates/` MUST:

* Contain exactly one policy or template document, respectively, in a format valid for the `policy_language` declared in `metadata.json`.
* Include a stable unique identifier for that policy or template within the policy store, expressed in the manner required by the policy language (see Policy Identifiers ({{policy-identifiers}})).

Filenames SHOULD use extensions conventional for the declared policy language. Filenames SHOULD be descriptive but are not required to match policy identifiers.

## Policy Identifiers {#policy-identifiers}

Each policy and template in a policy store MUST be uniquely identifiable within that store. How identifiers are assigned and embedded is **policy language specific**. For example, Cedar PDPs typically require an `@id()` annotation in each policy file; Cerbos PDPs identify policies by resource kind, name, and version fields within YAML or JSON policy documents.

Tools that are policy language agnostic MUST preserve policy files without altering language-specific identifiers.

## Entity Files {#default-entities}

Files under `entities/` MUST:

* Use the `.json` file extension.
* Contain a JSON ({{RFC8259}}) array of entity definitions, or a single entity definition object (which implementations MAY normalize to an array).

Each entity definition MUST include:

`uid`:
: REQUIRED object with `type` and `id` string fields identifying the entity in the policy language's type system.

`attrs`:
: REQUIRED object containing entity attributes.

`parents`:
: OPTIONAL array of parent entity references for hierarchical relationships. Each reference is an object with `type` and `id` string fields.

`tags`:
: OPTIONAL object of string key-value metadata.

PDPs MAY map this interchange format to engine-native entity representations on load.

## Trusted Issuer Files

Files under `trusted-issuers/` MUST:

* Use the `.json` file extension.
* Contain a single trusted issuer configuration object as defined in Trusted Issuers ({{trusted-issuers}}).

## Custom Issuer Files

Each file under `custom-issuers/` MUST:

* Use the `.json` file extension.
* Contain a single custom issuer configuration object as defined in Custom Issuers ({{custom-issuers}}).

## Schema Files

When the `schema/` directory is present, it MUST contain all schema artifacts required by the declared policy language for the policies in that store. Layout and file naming within `schema/` are defined by the policy language. A policy store MAY omit `schema/` entirely; in that case, the PDP MAY obtain schema from another source or operate without packaged schema, according to policy engine capabilities.

# Metadata {#metadata}

The `metadata.json` file provides version and descriptive metadata for the policy store.

## Structure

The top-level JSON object MUST contain the following keys:

`policy_store_spec_version`:
: REQUIRED string. Identifies the version of the AuthZEN Policy Store specification that this policy store conforms to. This revision of the specification defines the value `"1.0"`.

: Consistent with the versioning conventions of the AuthZEN Authorization API ({{AUTHZEN-API}}), the specification version and the endpoint path segment are distinct: version `1.0` corresponds to `v1` in endpoint identifiers, such as the `/access/v1/policy-store` endpoint defined by this specification.

`policy_language`:
: REQUIRED string. Identifies the policy language (for example, `"cedar"`, `"cel"`). Values SHOULD be lowercase alphanumeric strings; hyphens MAY separate words.

`policy_language_version`:
: REQUIRED string. Version of the policy language used by artifacts in this store (for example, `"4.4.0"` for Cedar).

`policy_store`:
: REQUIRED object containing policy store metadata fields defined below.

`governance`:
: REQUIRED object containing governance-related metadata fields defined below.

### policy_store Object

`id`:
: REQUIRED string. A unique identifier for the policy store. It MUST be a URI conforming to RFC 3986 and MUST uniquely identify the policy store

`name`:
: REQUIRED string. A human-readable name for the policy store.

`description`:
: OPTIONAL string. A human-readable description.

`version`:
: OPTIONAL string. A semantic version of the policy store content (for example, `"1.2.0"`).

`created_date`:
: OPTIONAL string. ISO 8601 date-time when the policy store was created.

Implementations MUST NOT add additional top-level keys to `metadata.json` unless documented by a future revision of this specification. The `policy_store` object MUST NOT contain keys other than those defined here unless documented by a future revision.

### governance Object

`owner`:
: A URN identifier that uniquely identifies an organizational entity accountable for the policy store 

`author`:
: A URN identifier that uniquely identifies an organizational entity that creates or authors the policy store

`scope`:
: A URN identifier for the domain or the area within the organization to which the policies 
in the policy store should be applied.

### Example (non-normative)

~~~ json
{
  "policy_store_spec_version": "1.0",
  "policy_language": "cedar",
  "policy_language_version": "4.4.0",
  "policy_store": {
    "id": "http://acme.com/apps/analytics/policystore/",
    "name": "Acme Analytics Web Application",
    "description": "Policies for the analytics web application.",
    "version": "1.2.0",
    "created_date": "2025-01-15T10:30:00Z"
  },
  "governance": {
  "owner": "urn:acme:user:ownername",
  "author": "urn:acme:user:authorname",
  "scope": "urn:example:application:analytics"
  }
}
~~~
{: title="Example metadata.json"}

# Trusted Issuers {#trusted-issuers}

Trusted issuer configuration files describe identity providers whose tokens a PDP MAY accept as evidence during policy evaluation. This enables PDPs to validate JSON Web Tokens ({{RFC7519}}) and map them to principal or entity types in the policy store's schema.

## Structure

Each trusted issuer file MUST be a JSON object with:

`id`:
: REQUIRED string. Unique identifier for the issuer. It MUST be a URI conforming to RFC 3986 and MUST uniquely identify the logical Policy

`name`:
: REQUIRED non-empty string. Short human-readable name.

`description`:
: OPTIONAL string.

`configuration_endpoint`:
: REQUIRED string. URI of the issuer configuration document (for example, OpenID Provider URI per {{RFC8615}}).

`token_metadata`:
: OPTIONAL object. Maps schema entity type names to per-type configuration objects. Each key names an entity type, defined in the policy store's schema, that tokens from this issuer are materialized as (for example, `Acme::Access_token` in Cedar, or a Cerbos principal schema reference).

### token_metadata Entry

For each schema entity type key in `token_metadata`, the value object MUST include:

`required_claims`:
: REQUIRED array of JWT claim names that MUST be present for the token to be considered valid.

Organization MAY define additional fields within entity type entries or at the issuer object level. Such extensions MUST NOT alter the meaning of fields defined in this specification. Interoperable tools SHOULD preserve unknown fields when reading and writing policy stores.

### Example (non-normative)

~~~ json
{
  "id": "3af079fa58a915a4d37a668fb874b7a25b70a37c03cf",
  "name": "Acme Identity Provider",
  "description": "Corporate identity provider",
  "configuration_endpoint": "https://idp.example.com/.well-known/openid-configuration",
  "token_metadata": {
    "Acme::Access_token": {
      "required_claims": ["jti", "iss", "aud", "sub", "exp", "nbf"]
    }
  }
}
~~~
{: title="Example trusted issuer configuration"}

# Custom Issuers {#custom-issuers}

Trusted issuer configuration ({{trusted-issuers}}) describes issuers whose tokens a PDP can validate using JSON Web Token ({{RFC7519}}) mechanisms and an issuer configuration document. Non-JWT tokens require specialized processing. These tokens include API keys, opaque session handles, vendor-specific or legacy token formats, and tokens that use encryption or signature algorithms the PDP does not implement natively.

Custom issuer configuration files describe the issuers of such custom tokens. They declare the schema entity types those tokens should be mapped to during policy evaluation. A PDP validates a custom token using a token processor rather than the mechanisms defined for trusted issuers. Token processor contains the logic to interpret and validate custom tokens. This component may be owned by the organization or 
PDP.

Consistent with the scope of this specification, this section defines the configuration artifact only. It does not define how a PDP validates a custom token, how a token processor is implemented or registered, or how a validated token is materialized as an entity. 

## Structure

Each custom issuer file MUST be a JSON object with:

`id`:
: REQUIRED string. Unique identifier for the issuer. It MUST be a URI conforming to RFC 3986 and MUST uniquely identify the issuer within the policy store.

`name`:
: REQUIRED non-empty string. Short human-readable name.

`description`:
: OPTIONAL string.

`token_metadata`:
: REQUIRED non-empty object. Maps schema entity type names to per-type configuration objects. Each key names an entity type, defined in the policy store's schema, that tokens from this issuer materialize as (for example, `Acme::ApiKey` in Cedar).

Unlike a trusted issuer, a custom issuer has no `configuration_endpoint`. Token validation logic is outside the scope of this specification and may be a part of token processor implementation.

A schema entity type name MUST NOT be declared by more than one issuer within a policy store, whether custom or trusted. A policy store that violates this constraint is invalid.

### token_metadata Entry

For each schema entity type key in `token_metadata`, the value object MUST include:

`required_claims`:
: REQUIRED array of claim names that the deployment expects to be present in the processed token.

All token types declared by a custom issuer are optional for evaluation. A PDP uses the custom tokens that are present in an authorization request and ignores those that are absent.

Organizations MAY define additional fields within entries or at the issuer object level. Such extensions MUST NOT alter the meaning of fields defined in this specification. Interoperable tools SHOULD preserve unknown fields when reading and writing policy stores.

A policy store MAY contain `trusted-issuers/` and `custom-issuers/`, either, or neither.

## PDP Support

A custom issuer configuration does not identify the token processor that handles a given entity type; the binding between a declared type and a processor is deployment configuration outside the policy store.

Because of this, a PDP loading a policy store that declares custom issuers is responsible for determining whether it can process each declared entity type. A PDP SHOULD perform this check while loading the policy store and reject the store if it cannot, rather than deferring the failure to the first evaluation request that presents such a token.

### Example (non-normative)

~~~ json
{
  "id": "https://acme.example/issuers/api-keys",
  "name": "Acme API Keys",
  "description": "Opaque API keys issued by the Acme developer portal",
  "token_metadata": {
    "Acme::ApiKey": {
      "required_claims": ["sub", "scope"]
    },
    "Acme::SessionKey": {
      "required_claims": ["sid"]
    }
  }
}
~~~
{: title="Example custom issuer configuration"}

# Archive Format {#archive-format}

The Archive Format packages the Directory Format as a ZIP archive (Constraint JAR, `.cjar`).

## Requirements

* The archive MUST use the ZIP format and `.cjar` file extension.
* Paths inside the archive MUST match the Directory Format layout relative to the policy store root.
* The archive MUST NOT require extraction to a specific absolute path; relative paths MUST be preserved.
* Archive file names SHOULD follow the pattern `{policy-store-name}-{version}.cjar` where `version` matches `policy_store.version` in `metadata.json` when present.

PDPs loading `.cjar` files SHOULD validate structure and required files before evaluation.

# Policy Store Loading

PDPs and tools that load policy stores SHOULD perform the following steps:

1. Detect format (directory or `.cjar` archive) and normalize to a directory view.
2. Verify required files and directories exist.
3. Parse and validate `metadata.json`.
4. Confirm the implementation supports the declared `policy_language` and `policy_language_version`, or reject the store.
5. If present, load `schema/`; then load policies and optional templates, entities, trusted issuers, and custom issuers according to policy engine rules.
6. Verify policy engine-specific requirements if any(such as unique policy identifiers).
7. Verify that no schema entity type is declared by more than one issuer file, whether under `trusted-issuers/` or `custom-issuers/` ({{custom-issuers}}).
8. If `custom-issuers/` is present, determine whether the implementation can process each declared entity type ({{custom-issuers}}).

Failure at any REQUIRED validation step SHOULD result in rejecting the policy store for evaluation.

# Relationship to AuthZEN

The AuthZEN Authorization API ({{AUTHZEN-API}}) standardizes communication between PEPs and PDPs. This specification standardizes how policy artifacts are packaged and distributed 
so that:

* PDPs implementing AuthZEN MAY advertise or load policy stores in a portable format 
* CI/CD pipelines MAY version, review, and promote policy stores as atomic units.
* Audit systems MAY bind decision logs to a specific `policy_store.id`.

# Update to Policy Decision Point Metadata {#update-pdp-metadata}

This specification adds a new Policy Decision Point Metadata endpoint parameter to the [existing AuthZEN parameter list](https://openid.net/specs/authorization-api-1_0.html#name-endpoint-parameters). The new parameter should be added as mentioned below:

`policy_store_endpoint`: OPTIONAL. HTTPS URL of the Policy Decision Point's Policy store endpoint 

The parameter should be registered in the IANA registry as established under [IANA Considerations](#iana-considerations). The parameter should be obtainable using the AuthZEN `.well-known/authzen-configuration` in the same way as described in the [relevant section](https://openid.net/specs/authorization-api-1_0.html#name-obtaining-policy-decision-p) of the AuthZEN Authorization API Specification.

A request such as the following to the AuthZEN `.well-known` endpoint should include the `policy_store_endpoint` parameter if the Policy Decision Point supports this endpoint.

~~~
GET /.well-known/authzen-configuration HTTP/1.1
Host: pdp.example.com
~~~

A non-normative example of the successful response:

~~~
HTTP/1.1 200 OK
Content-Type: application/json

{
  "policy_decision_point": "https://pdp.example.com",
  "access_evaluation_endpoint": "https://pdp.example.com/access/v1/evaluation",
  "search_subject_endpoint": "https://pdp.example.com/access/v1/search/subject",
  "search_resource_endpoint": "https://pdp.example.com/access/v1/search/resource",
  "policy_store_endpoint": "https://pdp.example.com/access/v1/policy-store"
}
~~~

# Transport {#transport}

This specification defines an HTTPS binding using JSON serialization which MUST be implemented by a compliant PDP.

## HTTPS JSON Binding

All API requests to the policy store endpoint MUST be made via an HTTPS POST request.

Requests MUST include a `Content-Type` header with the value `multipart/form-data` [[RFC7578]]. The request body MUST conform to the request structure defined in ({{api-request}}).

| API Endpoint | Default Path | Metadata Parameter | Request Schema | Response Schema |
| :--- | :--- | :--- | :--- | :--- |
| Policy Store | /access/v1/policy-store | policy_store_endpoint | {{api-request}} | {{api-response}} |


A successful response is an HTTPS response with a status code of 201 Created and a `Content-Type` of `application/json`. The response body MUST be a JSON object that conforms to the structure defined in ({{api-response}}).

The request URL MUST be the value of the `policy_store_endpoint` parameter ({{update-pdp-metadata}}) if provided in the Policy Decision Point metadata. If the parameter is omitted, the URL SHOULD be formed by appending the default path (as defined in table-1) to the PDP’s base URL. PDP's base URL should be obtained from the value of `policy_decision_point` parameter of the Policy Decision Point metadata as defined by AuthZEN Authorization API ({{AUTHZEN-API}}), if available.

### Error Responses

Error responses use HTTPS status codes to indicate failures. The PDP MUST return a 400 Bad Request if the multipart body is malformed or if required parts are missing. Authentication and authorization failures MUST return 401 Unauthorized or 403 Forbidden, respectively.

| Code | Description | HTTPS Body Content |
| :--- | :--- | :--- |
| 400 | Bad Request | An error message string or JSON object with error details |
| 401 | Unauthorized | An error message string |
| 403 | Forbidden | An error message string |
| 500 | Internal Error | An error message string |

### Request Identification

Requests MAY include a unique identifier in the `X-Request-ID` header as defined in the AuthZEN Authorization API ({{AUTHZEN-API}}). If present, the PDP MUST include the same identifier in the `X-Request-ID` response header to assist in request tracing and debugging. 

### Example (non-normative)

The following example shows an HTTPS upload of a policy store:

~~~
POST /access/v1/policy-store HTTP/1.1
Host: pdp.example.com
Authorization: Bearer mF_9.B5f-4.1JqM
X-Request-ID: bfe9eb29-ab87-4ca3-be83-a1d5d8305716
Content-Type: multipart/form-data; boundary=---XYZ

-----XYZ
Content-Disposition: form-data; name="file"; filename="store.cjar"
Content-Type: application/zip

<binary .cjar content>
-----XYZ--
~~~


# Security Considerations

## Policy store

Policy stores contain authorization rules and may include sensitive configuration. Implementations MUST protect policy stores at rest and in transit using appropriate access controls for the deployment environment.

Deployments SHOULD use signed releases or secure artifact repositories where integrity and provenance are required.

Trusted issuer configuration determines which token issuers a PDP accepts. Incorrect issuer configuration can allow unauthorized principals. Implementations MUST validate `configuration_endpoint` URIs and token metadata before trusting tokens in production.

Policy stores SHOULD be treated as part of the trusted computing base for authorization decisions. Loading a policy store from an untrusted source without validation is NOT RECOMMENDED.

## Custom issuers

Custom issuer configuration ({{custom-issuers}}) moves credential validation out of the mechanisms defined for trusted issuers and into a deployment-supplied token processor. The token processor part of the trusted computing base for authorization decisions.

Schema entity type collisions are a privilege-escalation risk: if two issuers could declare the same entity type, a token validated under weaker rules could materialize as an entity that policies treat as higher-privilege. This is why {{custom-issuers}} requires entity type declarations to be unique across both custom and trusted issuers within a store.

Because a custom issuer has no configuration document, there is no standard mechanism for key rotation, token status, or revocation. Deployments that accept custom tokens MUST provide these controls through the token processor or surrounding infrastructure.

All declared token types are optional, so a PDP may reach a decision with fewer tokens than a deployment anticipated. Deployments SHOULD review policies that reference custom token entity types to confirm that a decision remains correct when tokens of those types are absent.

## Policy store API

PDP should protect this API endpoint by taking appropriate measures to authenticate and authorise the request in accordance with [Authorization API guidelines](https://openid.net/specs/authorization-api-1_0.html#section-11.2)

# IANA Considerations

The AuthZEN working group has defined a metadata registry for OpenID AuthZEN Policy Decision Points (PDPs), [[AUTHZEN-PDP-METADATA]](https://openid.github.io/authzen/#name-authzen-policy-decision-poi). This document extends that registry with a requirement to add a policy store API to the registry with the following metadata.

Metadata Name:
`policy_store_endpoint`

Metadata Description:
HTTPS URL of the Policy Decision Point's Policy store endpoint 

Change Controller:
OpenID Foundation AuthZEN Working Group

mailto:openid-specs-authzen@lists.openid.net

Specification Document(s):
See the [Policy Store API specification](#policy-store-api-specification) section.

--- back

# JSON Schemas (Informative) {#json-schemas}

The following JSON Schemas illustrate the structure of normative JSON artifacts. They are informative; in case of conflict, the prose requirements in this document take precedence.

## metadata.json Schema

~~~ json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": [
    "policy_store_spec_version",
    "policy_language",
    "policy_language_version",
    "policy_store",
    "governance"
  ],
  "properties": {
    "policy_store_spec_version": { "type": "string", "minLength": 1 },
    "policy_language": { "type": "string", "minLength": 1 },
    "policy_language_version": { "type": "string", "minLength": 1 },
    "policy_store": {
      "type": "object",
      "required": ["id", "name", "version"],
      "properties": {
        "id": { "type": "string", "format": "uri"},
        "name": { "type": "string" },
        "description": { "type": "string" },
        "version": { "type": "string" },
        "created_date": { "type": "string", "format": "date-time" }
      },
      "additionalProperties": false
    },
    "governance": {
      "type": "object",
      "required": ["owner", "author", "scope"],
      "properties": {
        "owner": { "type": "string", "format": "urn"},
        "author": { "type": "string", "format": "urn"},
        "scope": { "type": "string", "format": "urn"}
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": false
}
~~~

## Trusted Issuer Schema

~~~ json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["id", "name", "configuration_endpoint"],
  "properties": {
    "id": { "type": "string", "format": "uri"},
    "name": { "type": "string", "minLength": 1 },
    "description": { "type": "string" },
    "configuration_endpoint": { "type": "string", "format": "uri" },
    "token_metadata": {
      "type": "object",
      "patternProperties": {
        "^[A-Za-z_][A-Za-z0-9_]*(::[A-Za-z_][A-Za-z0-9_]*)*$": {
          "type": "object",
          "required": ["required_claims"],
          "properties": {
            "required_claims": {
              "type": "array",
              "items": { "type": "string", "minLength": 1 },
              "uniqueItems": true
            }
          },
          "additionalProperties": true
        }
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": true
}
~~~

## Custom Issuer Schema

~~~ json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["id", "name", "token_metadata"],
  "properties": {
    "id": { "type": "string", "format": "uri"},
    "name": { "type": "string", "minLength": 1 },
    "description": { "type": "string" },
    "token_metadata": {
      "type": "object",
      "minProperties": 1,
      "patternProperties": {
        "^[A-Za-z_][A-Za-z0-9_]*(::[A-Za-z_][A-Za-z0-9_]*)*$": {
          "type": "object",
          "required": ["required_claims"],
          "properties": {
            "required_claims": {
              "type": "array",
              "items": { "type": "string", "minLength": 1 },
              "uniqueItems": true
            }
          },
          "additionalProperties": true
        }
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": true
}
~~~

## Entity File Schema

~~~ json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "array",
  "items": {
    "type": "object",
    "required": ["uid", "attrs"],
    "properties": {
      "uid": {
        "type": "object",
        "required": ["type", "id"],
        "properties": {
          "type": { "type": "string", "minLength": 1 },
          "id": { "type": "string" }
        },
        "additionalProperties": false
      },
      "attrs": { "type": "object", "additionalProperties": true },
      "parents": {
        "type": "array",
        "items": {
          "type": "object",
          "required": ["type", "id"],
          "properties": {
            "type": { "type": "string", "minLength": 1 },
            "id": { "type": "string" }
          },
          "additionalProperties": false
        }
      },
      "tags": {
        "type": "object",
        "additionalProperties": { "type": "string" }
      }
    },
    "additionalProperties": false
  }
}
~~~

# Policy Engine Examples (Informative) {#examples}

The following examples illustrate how different PDPs MAY populate the same AuthZEN policy store layout. They are not normative.

## Cedar PDP Example

A Cedar-based PDP ({{CEDAR}}) might use `policy_language` value `"cedar"`, place a Cedar schema file under `schema/`, and use `.cedar` policy files:

~~~ ascii-art
todo-app-policy-store/
├── metadata.json
├── schema/
│   └── app.cedarschema
├── policies/
│   ├── alice-read-access.cedar
│   └── jack-search-access.cedar
├── entities/
│   └── default-roles.json
├── trusted-issuers/
│   └── acme-idp.json
└── custom-issuers/
    └── acme-apikeys.json
~~~

**metadata.json:**

~~~ json
{
  "policy_store_spec_version": "1.0",
  "policy_language": "cedar",
  "policy_language_version": "4.4.0",
  "policy_store": {
    "id": "http://acme.com/apps/todo/policystore/",
    "name": "todo_app_policy_store",
    "version": "1.0.0"
  },
  "governance": {
  "owner": "urn:acme:user:ownername",
  "author": "urn:acme:user:authorname",
  "scope": "urn:example:application:analytics"
  }
}
~~~

**policies/alice-read-access.cedar:**

~~~ cedar
@id("alice-read-policy")
permit(
  principal == App::User::"Alice",
  action == App::Action::"Read",
  resource == App::Application::"todo"
);
~~~

The same store MAY be distributed as `todo-app-policy-store-v1.0.0.cjar`.

## CEL PDP Example

A CEL-based Cerbos PDP ({{CERBOS}}) may consume a policy store which has `policy_language` value `"CEL"`, store JSON schemas under `schema/`, and use YAML policy files under `policies/`:

~~~ ascii-art
hr-policy-store/
├── metadata.json
├── schema/
│   └── principal.json
├── policies/
│   └── resource_policies/
│       └── leave_request.yaml
└── trusted-issuers/
    └── corp-idp.json
~~~

**metadata.json:**

~~~ json
{
  "policy_store_spec_version": "1.0",
  "policy_language": "cel",
  "policy_language_version": "0.25.0",
  "policy_store": {
    "id": "http://acme.com/hr/policystore/",
    "name": "hr_policy_store",
    "version": "2.0.0"
  },
  "governance": {
  "owner": "urn:acme:user:ownername",
  "author": "urn:acme:user:authorname",
  "scope": "urn:example:application:analytics"
  }
}
~~~

**policies/resource_policies/leave_request.yaml (excerpt):**

~~~ yaml
apiVersion: api.cerbos.dev/v1
resourcePolicy:
  version: default
  resource: leave_request
  rules:
    - actions: ["view:*"]
      effect: EFFECT_ALLOW
      roles: ["employee"]
~~~

Cerbos deployments that use a native repository layout (for example, a top-level `_schemas` directory) MAY translate to or from this AuthZEN layout when importing or exporting a `.cjar` package.

# Open Issues (Informative) {#open-issues}

The following topics may be addressed in future revisions:

* A registry of recognized `policy_language` identifier strings.
* Whether policy identifiers MUST be globally unique across policies and templates, and how that interacts with filenames.
* How policy templates are instantiated and linked to policies in multi-file layouts.
* Expression and validation of cross-policy dependencies.
* Subdirectory organization conventions under `policies/` per policy language.


# Notices

Copyright (c) 2026 The OpenID Foundation.

The OpenID Foundation (OIDF) grants to any Contributor, developer, implementer, or other interested party a non-exclusive, royalty free, worldwide copyright license to reproduce, prepare derivative works from, distribute, perform and display, this draft or final specification solely for (i) developing specifications, and (ii) implementing specifications based on such documents, provided that attribution be made to the OIDF as the source of the material, but that such attribution does not indicate an endorsement by the OIDF.

The technology described in this specification was made available from contributions from various sources, including members of the OpenID Foundation and others. Although the OpenID Foundation has taken steps to help ensure that the technology is available for distribution, it takes no position regarding the validity or scope of any intellectual property or other rights that might be claimed to pertain to the implementation or use of the technology described in this specification or the extent to which any license under such rights might or might not be available; neither does it represent that it has made any independent effort to identify any such rights. The OpenID Foundation and the contributors to this specification make no (and hereby expressly disclaim any) warranties (express, implied, or otherwise), including implied warranties of merchantability, non-infringement, fitness for a particular purpose, or title, related to this specification, and the entire risk as to implementing this specification is assumed by the implementer. The OpenID Intellectual Property Rights policy requires contributors to offer a patent promise not to assert certain patent claims against other contributors and against implementers. OpenID invites any interested party to bring to its attention any copyrights, patents, patent applications, or other proprietary rights that may cover technology that may be required to practice this specification.
