---
layout: default
title: Document-level security
parent: Access control
nav_order: 85
redirect_from:
- /security/access-control/document-level-security/
- /security-plugin/access-control/document-level-security/
---

# Document-level security
Document-level security lets you restrict a role to a subset of documents in an index. The easiest way to get started with document- and field-level security is to open OpenSearch Dashboards and choose **Security**. Then choose **Roles**, create a new role, and review the **Index Permissions** section, shown in the following image.

![Document- and field-level security screen in OpenSearch Dashboards]({{site.url}}{{site.baseurl}}/images/security-dls.png)

The maximum size for the document-level security configuration is 1024 KB (1,048,404 characters). 
{: .warning}

## Simple roles

Document-level security uses OpenSearch query domain-specific language (DSL) to define which documents a role grants access to. In OpenSearch Dashboards, choose an index pattern and provide a query in the **Document-level security** section:

```json
{
  "bool": {
    "must": {
      "match": {
        "genres": "Comedy"
      }
    }
  }
}
```

This query specifies that for the role to have access to a document, its `genres` field must include `Comedy`.

A typical request sent to the `_search` API includes `{ "query": { ... } }` around the query, but in this case, you only need to specify the query itself.

## Updating roles by accessing the REST API 

In the REST API, you provide the query as a string, so you must escape your quotes. This role allows a user to read any document in any index with the field `public` set to `true`:

```json
PUT _plugins/_security/api/roles/public_data
{
  "cluster_permissions": [
    "*"
  ],
  "index_permissions": [{
    "index_patterns": [
      "pub*"
    ],
    "dls": "{\"term\": { \"public\": true}}",
    "allowed_actions": [
      "read"
    ]
  }]
}
```

These queries can be as complex as you want, but we recommend keeping them simple to minimize the performance impact that the document-level security feature has on the cluster.
{: .warning }

### A note on Unicode special characters in text fields

Due to word boundaries associated with Unicode special characters, the Unicode standard analyzer cannot index a [text field type]({{site.url}}{{site.baseurl}}/opensearch/supported-field-types/text/) value as a whole value when it includes one of these special characters. As a result, a text field value that includes a special character is parsed by the standard analyzer as multiple values separated by the special character, effectively tokenizing the different elements on either side of it. This can lead to unintentional filtering of documents and potentially compromise control over their access.

The examples below illustrate values containing special characters that will be parsed improperly by the standard analyzer. In this example, the existence of the hyphen/minus sign in the value prevents the analyzer from distinguishing between the two different users for `user.id` and interprets them as one and the same:

```json
{
  "bool": {
    "must": {
      "match": {
        "user.id": "User-1"
      }
    }
  }
}
```

```json
{
  "bool": {
    "must": {
      "match": {
        "user.id": "User-2"
      }
    }
  }
}
```

To avoid this circumstance when using either Query DSL or the REST API, you can use a custom analyzer or map the field as `keyword`, which performs an exact-match search. See [Keyword field type]({{site.url}}{{site.baseurl}}/opensearch/supported-field-types/keyword/) for the latter option.

For a list of characters that should be avoided when field type is `text`, see [Word Boundaries](https://unicode.org/reports/tr29/#Word_Boundaries).


## Parameter substitution

A number of variables exist that you can use to enforce rules based on the properties of a user. For example, `${user.name}` is replaced with the name of the current user.

This rule allows a user to read any document where the username is a value of the `readable_by` field:

```json
PUT _plugins/_security/api/roles/user_data
{
  "cluster_permissions": [
    "*"
  ],
  "index_permissions": [{
    "index_patterns": [
      "pub*"
    ],
    "dls": "{\"term\": { \"readable_by\": \"${user.name}\"}}",
    "allowed_actions": [
      "read"
    ]
  }]
}
```

This table lists substitutions.

Term | Replaced with
:--- | :---
`${user.name}` | Username.
`${user.roles}` | A comma-separated, quoted list of user backend roles.
`${user.securityRoles}` | A comma-separated, quoted list of user security roles. 
`${attr.<TYPE>.<NAME>}` | An attribute with name `<NAME>` defined for a user. `<TYPE>` is `internal`, `jwt`, `proxy` or `ldap`


## Attribute-based security

You can use roles and parameter substitution with the `terms_set` query to enable attribute-based security.

> Note that the `security_attributes` of the index need to be of type `keyword`.

#### User definition

```json
PUT _plugins/_security/api/internalusers/user1
{
  "password": "asdf",
  "backend_roles": ["abac"],
  "attributes": {
    "permissions": "\"att1\", \"att2\", \"att3\""
  }
}
```

#### Role definition

```json
PUT _plugins/_security/api/roles/abac
{
  "index_permissions": [{
    "index_patterns": [
      "*"
    ],
    "dls": "{\"terms_set\": {\"security_attributes\": {\"terms\": [${attr.internal.permissions}], \"minimum_should_match_script\": {\"source\": \"doc['security_attributes'].length\"}}}}",
    "allowed_actions": [
      "read"
    ]
  }]
}
```
## Use term-level lookup queries (TLQs) with DLS 

You can perform term-level lookup queries (TLQs) with document-level security (DLS) using either of two modes: adaptive or filter level. The default mode is adaptive, where OpenSearch automatically switches between Lucene-level or filter-level mode depending on whether or not there is a TLQ. DLS queries without TLQs are executed in Lucene-level mode, whereas DLS queries with TLQs are executed in filter-level mode.

By default, the Security plugin detects if a DLS query contains a TLQ or not and chooses the appropriate mode automatically at runtime.

To learn more about OpenSearch queries, see [Term-level queries]({{site.url}}{{site.baseurl}}/query-dsl/term/index/).

### How to set the DLS evaluation mode in `opensearch.yml`

By default, the DLS evaluation mode is set to `adaptive`. You can also explicitly set the mode in `opensearch.yml` with the `plugins.security.dls.mode` setting. Add a line to `opensearch.yml` with the desired evaluation mode. 
For example, to set it to filter level, add this line:
```
plugins.security.dls.mode: filter-level
```

#### DLS evaluation modes

| Evaluation mode | Parameter | Description | Usage |
:--- | :--- | :--- | :--- |
Lucene-level DLS | `lucene-level` | This setting makes all DLS queries apply to the Lucene level. | Lucene-level DLS modifies Lucene queries and data structures directly. This is the most efficient mode but does not allow certain advanced constructs in DLS queries, including TLQs.
Filter-level DLS | `filter-level` | This setting makes all DLS queries apply to the filter level. | In this mode, OpenSearch applies DLS by modifying queries that OpenSearch receives. This allows for term-level lookup queries in DLS queries, but you can only use the `get`, `search`, `mget`, and `msearch` operations to retrieve data from the protected index. Additionally, cross-cluster searches are limited with this mode.
Adaptive | `adaptive-level` | The default setting that allows OpenSearch to automatically choose the mode. | DLS queries without TLQs are executed in Lucene-level mode, while DLS queries that contain TLQ are executed in filter- level mode.

## DLS and multiple roles

OpenSearch combines all DLS queries with the logical `OR` operator. However, when a role that uses DLS is combined with another security role that doesn't use DLS, the query results are filtered to display only documents matching the DLS from the first role. This filter rule also applies to roles that do not grant read documents.

### DLS and write permissions

Make sure that a user that has DLS-configured roles does not have write permissions. If write permissions are added, the user will be able to index documents which they will not be able to retrieve due to DLS filtering.

### When to enable `plugins.security.dfm_empty_overrides_all`

When to enable the `plugins.security.dfm_empty_overrides_all` setting depends on whether you want to restrict user access to documents without DLS. 


To ensure access is not restricted, you can set the following configuration in `opensearch.yml`:

```
plugins.security.dfm_empty_overrides_all: true
```
{% include copy.html %}


The following examples show what level of access roles with DLS enabled and without DLS enabled, depending on the interaction. These examples can help you decide when to enable the `plugins.security.dfm_empty_overrides_all` setting.

#### Example: Document access

This example demonstrates that enabling `plugins.security.dfm_empty_overrides_all` is beneficial in scenarios where you need specific users to have unrestricted access to documents despite being part of a broader group with restricted access.

**Role A with DLS**: This role is granted to a broad group of users and includes DLS to restrict access to specific documents, as shown in the following permission set:

```
{
  "index_permissions": [
    {
      "index_patterns": ["example-index"],
      "dls": "[.. some DLS here ..]",
      "allowed_actions": ["indices:data/read/search"]
    }
  ]
}
```

**Role B without DLS:** This role is specifically granted to certain users, such as administrators, and does not include DLS, as shown in the following permission set:

```
{
  "index_permissions" : [
    {
      "index_patterns" : ["*"],
      "allowed_actions" : ["indices:data/read/search"]
    }
  ]
}
```
{% include copy.html %}

Setting `plugins.security.dfm_empty_overrides_all` to `true` ensures that administrators assigned Role B can override any DLS restrictions imposed by Role A. This allows specific Role B users to access all documents, regardless of the restrictions applied by Role A's DLS restrictions.

#### Example: Search template access

In this example, two roles are defined, one with DLS and another without DLS, granting access to search templates:

**Role A with DLS:**

```
{
  "index_permissions": [
    {
      "index_patterns": [
        "example-index"
      ],
      "dls": "[.. some DLS here ..]",
      "allowed_actions": [
        "indices:data/read/search",
      ]
    }
  ]
}
```
{% include copy.html %}

**Role B, without DLS**, which only grants access to search templates:

```
{
  "index_permissions" : [
    {
      "index_patterns" : [ "*" ],
      "allowed_actions" : [ "indices:data/read/search/template" ]
    }
  ]
}
```
{% include copy.html %}

When a user has both Role A and Role B permissions, the query results are filtered based on Role A's DLS, even though Role B doesn't use DLS. The DLS settings are retained, and the returned access is appropriately restricted. 

When a user is assigned both Role A and Role B and the `plugins.security.dfm_empty_overrides_all` setting is enabled, Role B's permissions Role B's permissions will override Role A's restrictions, allowing that user to access all documents. This ensures that the role without DLS takes precedence in the search query response.
