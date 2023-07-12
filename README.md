# Motivation

## Protocol-specific details in a CloudEvent

Currently, event definitions are a format on their own. As a consequence, a message can either be defined as a
CloudEvent or for a specific protocol. Event definitions are meant to be independent of a specific protocol. On the
other hand, when exposing events on a specific endpoint, additional protocol-specific information such as paths might be
needed.

## Event Definitions as Re-Usable Information

The key value proposition of CloudEvents is the interoperability across multiple infrastructures, protocols and formats.
An event may be propagated across multiple endpoints. For this scenario it is necessary to separate the abstract
CloudEvent definition from the concrete endpoint specification that might contain additional, protocol-specific details.

# Proposal

To address the needs described above, two new concepts are proposed.

## Adding CloudEvents to Message Definitions

As CloudEvents are not a message format on their own, they must be combined with message definitions to be exposed over
endpoints. Therefore, the original formats get extended with protocol specific information, such as path, http method,
or AMQP properties.

## Generator

Generators describe how to generate a RESOURCE. Typically, they point to another RESOURCE and contain information about
how this original RESOURCE should be enriched. Generators can be used in places where the original RESOURCE would be
expected, if the registry's model supports it:

``` text
{
  "schema": "URI-Reference", ?         # Schema doc for the entire Registry
  "groups": [
    { "singular": "STRING",            # eg. "endpoint"
      "plural": "STRING",              # eg. "endpoints"
      "generator":"STRING",            # eg. "endpointGenerator"
      "schema": "URI-Reference", ?     # Schema doc for the group

      "resources": [
        { "singular": "STRING",        # eg. "definition"
          "plural": "STRING",          # eg. "definitions"
          "generator": "STRING", ?     # eg. "definitionsGenerator" 
          "versions": UINT, ?          # Num Versions(>=0). Def=1, 0=unlimited
          "versionId": BOOL, ?         # Supports client specific Version IDs
          "latest": BOOL ?             # Supports client "latest" selection
        } *
      ] ?
    } *
  ] ?
}
```

The REST API allows to read the generated resources, as well as the generator definitions. The generated resources will
be added to the normal list of resources. They will just contain an additional `generatedBy` attribute pointing to the
generator definition.

While the initial idea for generators is to enable re-use of CloudEvents definitions, they can, essentially, be used to
generate anything. It could even be possible to define generators for helm charts, or generators for AsyncAPI files that
take `endpoints` as input. Depending on evolving use cases, there could be a concept for generator dialects, similar to
filter dialects of the CloudEvents subscription specification.

### Generators in Registry Files and the API

As generators describe how to derive a definition or a group of definitions from existing content, registry files may
only contain the generators. For an API it depends on what the model provides, but if a registry API supports a certain
type of generator, it should be possible to access the generated resources as well as the generators.

### Generators for Definitions

A generator for `definition` could have this form:

```json lines
{
  "apply": {
  },
  // Points to a resource
  "to": "<URIRef>",
  "with": {
  }
}
```

The `apply` section contains additions that are supposed to be added to another `definition`, while the `to` attribute
links to the original definition. In the `with` section, it is possible to provide additional contextual information.
One application for this is to specify values for parameters that were used in the original definition. If the original
definition contains a constraint like

```json lines
{
  "source": {
    "type": "uritemplate",
    "value": "/crm/{tenantId}/customers",
    "required": true
  }
}
```

the `with` section could restrict the value of `source` further to a specific `tenantId`:

```json lines
{
  "with": {
    ".to.$tenantId": "1234567"
  }
}

```

### Generators for Definition Groups and Endpoints

For groups or endpoints, a generator could apply the content of the `apply` section to each `definition`:

```json lines
{
  "webhook": {
    "id": "webhook",
    "format": "HTTP/1.1+CloudEvents/1.0",
    "usage": "producer",
    "apply": {
      "path": "/",
      "method": "POST",
      "cloudevents": {
        "supportedContentModes": [
          "binary",
          "structured",
          "batch"
        ]
      }
    },
    "to": "#definitionGroups/Contoso.CRM.Events",
    "with": {}
  }
}
```

In the example above, a group of pre-defined CloudEvents is bound to an HTTP endpoint. As the HTTP endpoint represents
an event gateway, there is no assigment of different events to different paths. Instead, all events get the `path` "`\`"
and the http method `POST`. The `cloudevents` sections gets enriched by a statement specifying the supported content
modes.