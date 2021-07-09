# RESTful API Schema Extensions

This is a proof of concept for REST Schema Extensions (REST-SCHEMA). The idea behind this spec is to provide a set of utilities on top of RESTful APIs, aimed at reducing bandwidth and reducing the number of requests.

The available operations are as follows

- Schema-Mapping
- Schema-Include

All of the above use a schema spec to describe the operation, which can be defined in JSON, YAML or plain text. See more about the schema formats ahead in the document.
The schema spec is then encoded as base64 or base64url and passed through a request header.

## Schema-Mapping

Header name   : X-Schema-Map
Parameter name: schemaMap

This operation uses the schema data to map the results of a resource. Only properties that match the schema are retrieved.

For example, let's consider the following request and the default response

```json
GET /users/10
{
    "id": 10,
    "name": "John Doe",
    "dob": "1990-01-23",
    "phoneNumber": "55000000000",
    "email": "johndoe@email.com",
    "teams": [
        {
            "id": 13,
            "name": "Marketing"
        },
        {
            "id": 18,
            "name": "Employees"
        }
    ]
}
```

If we use a mapping with only the following fields

```json
{
    "spec": {
        "_": ["name", "email"]
    }
}
```

And add the `X-Schema-Map` header, we would get the following response instead

```json
GET /users/10
X-Schema-Map: ewogICAgInNwZWMiOiB7CiAgICAgICAgIl8iOiBbIm5hbWUiLCAiZW1haWwiXQogICAgfQp9Cg
{
    "name": "John Doe",
    "email": "johndoe@email.com"
}
```

The same works if the response is a list of users, instead of a single user

```json
GET /users
X-Schema-Map: ewogICAgInNwZWMiOiB7CiAgICAgICAgIl8iOiBbIm5hbWUiLCAiZW1haWwiXQogICAgfQp9Cg
[
    {
        "name": "John Doe",
        "email": "johndoe@email.com"
    },
    {
        "name": "Jane Doe",
        "email": "janedoe@email.com"
    }
```

## Schema-Include

Header name   : X-Schema-Include
Parameter name: schemaInclude

This operation is to be used in place of `Schema-Mapping`. If both `Schema-Mapping` and `Schema-Include` are specified, the latter is ignored.

The purpose of this operation is to include information that isn't returned by default. Let's consider the previous example, but we'll say that `teams` property isn't returned by default. So our base request and response would be

```json
GET /users/10
{
    "id": 10,
    "name": "John Doe",
    "dob": "1990-01-23",
    "phoneNumber": "55000000000",
    "email": "johndoe@email.com"
}
```

In order to get the user's teams, we would typically require a second request

```json
GET /users/10/teams
[
    {
        "id": 13,
        "name": "Marketing"
    },
    {
        "id": 18,
        "name": "Employees"
    }
]
```

So, in order to include the teams and perform a single request, we would use the `Schema-Include` with the following data

```json
{
    "spec": {
        "_": ["teams"]
    }
}
```

This would instruct the API to get the dependency data and we would get the following response

```json
GET /users/10
X-Schema-Include: ewogICAgInNwZWMiOiB7CiAgICAgICAgIl8iOiBbInRlYW1zIl0KICAgIH0KfQ
{
    "id": 10,
    "name": "John Doe",
    "dob": "1990-01-23",
    "phoneNumber": "55000000000",
    "email": "johndoe@email.com",
    "teams": [
        {
            "id": 13,
            "name": "Marketing"
        },
        {
            "id": 18,
            "name": "Employees"
        }
    ]
}
```

## Schema Format

The spec for the schema data can be defined in either JSON or YAML, both are supported. The schema data is then encoded as base64 or base64url.

You can have a look at the full schema in [schema.json](schema.json) or [schema.yaml](schema.yaml).

Let's consider the following resource as an example

```json
GET /users/10
{
    "id": 10,
    "name": "John Doe",
    "dob": "1990-01-23",
    "phoneNumber": "55000000000",
    "email": "johndoe@email.com",
    "teams": [
        {
            "id": 13,
            "name": "Marketing"
        },
        {
            "id": 18,
            "name": "Employees"
        }
    ]
}
```

A basic example of schema data, to use on a `Schema-Mapping` operation, would be

```json
{
    "spec": {
        "_": ["name", "email"]
    }
}
```

The first (root) element is the mapping for our result and it can be named whatever we want - the name is indifferent.

We can also specify additional schemas for complex (objects) properties. Still using the above example, we can further describe the properties to retrieve for the teams, but we are only interested in their ids, not the names. Here, the name of the schema needs to match the name of the property.

```json
{
    "spec": {
        "_": ["name", "email", "teams"],
        "teams": ["id"]
    }
}
```

The schema data can also be defined in plain text, but this is NOT a spec requirement and its implementation is OPTIONAL.

Plain text schema data doesn't use encoding and if implemented, it should only be used for testing. In a production scenario, it is recommended to use JSON or YAML.

Here's an example of the same schema data above in plain text

```text
user=name,email,teams;teams=id
```

Multiple schemas should be separated by a `;` character. The root schema name is optional, so the above could simply be written like this

```text
name,email,teams;teams=id
```

## Schema Data Encoding

As addressed before, the schema data needs to be encoded as a base64 or a base64url string before it can be used on a request.

Let's consider the following schema data

```json
{
    "spec": {
        "_": ["name", "email"]
    }
}
```

The base64 encoded value for this schema is `eyJzcGVjIjp7Il8iOlsibmFtZSIsICJlbWFpbCJdfX0=`. We would pass this value on a header along with the request, for example

```
GET /users
X-Schema-Map: eyJzcGVjIjp7Il8iOlsibmFtZSIsICJlbWFpbCJdfX0=
```

If we are using a query string parameter, we need to either url-encode that value, since base64 uses characters that aren't safe on a url (`/`, `-` and `=`), or encode the value as a base64url instead. The above schema data as a base64 url is exactly the same, but without the ending `=`.

```
GET /users?schemaMap=eyJzcGVjIjp7Il8iOlsibmFtZSIsICJlbWFpbCJdfX0
```

## Schema Version

The versioning in the schema data is optional, but could be useful to avoid unsupported actions. The way the API reacts is up to the implementation; it can respond with a 400 (bad request) or simply ignore the schema. See the full spec for details.
The client can also send the schema version in the `X-Schema-Version` header, but it does not take precedence if also defined in the schema data.

The API response MUST include the `X-Schema-Version` header if the request includes schema data, even if the version is not supported and the schema ignored.
