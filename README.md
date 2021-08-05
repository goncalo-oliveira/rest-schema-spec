# RESTful API Schema Extensions

This document describes the specification for *REST Schema Extensions* (REST-SCHEMA). The idea behind this spec is to provide a set of utilities on top of RESTful APIs, reducing bandwidth and reducing the number of requests.

The available operations are as follows

- Schema-Mapping
- Schema-Include

All of the above use a schema spec to describe the operation, which can be defined in *Json* or *plain text*. See more about the schema formats ahead in the document.
The schema spec is then encoded as *base64* or *base64url* and passed through a request header.

## Headers vs Query Parameters

The schema operations can be requested via headers or query parameters. Using the headers leaves the request url cleaner, but it might create issues with cache. Using the query parameters can lead to a longer url, but it will work better with cache. The server SHOULD implement both, but it is up to the client to choose which one to use.

> The examples in this document will show the mapping in both headers and query parameters.

## Query Parameters Naming Conventions

Although it is not to be enforced, it is recommended to follow these conventions. In general, query parameter names should be camel case or snake case. Additionally, query parameters can have two distinct roles: *filters* and *actions*.

Filters are used to reduce the results by matching a certain criteria. There is no additional criteria to the naming conventions.

Actions are used to perform operations or transformations over the result. The names for these parameters should start with an underscore (`_`).

```json
GET /cars?color=blue&_orderBy=year
```

In the above example, `color` is a filter and `_orderBy` is an action.

## Schema-Mapping

```none
Header name   : X-Schema-Map
Parameter name: _map
```

This operation uses the schema data to map the results of a resource. Only properties that match the schema are retrieved.

For example, let's consider the following request and the default response

```json
GET /users/10
---
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

And add the mapping to the request, we would get the following response instead

```json
GET /users/10
X-Schema-Map: ewogICAgInNwZWMiOiB7CiAgICAgICAgIl8iOiBbIm5hbWUiLCAiZW1haWwiXQogICAgfQp9Cg
---
GET /users/10?_map=ewogICAgInNwZWMiOiB7CiAgICAgICAgIl8iOiBbIm5hbWUiLCAiZW1haWwiXQogICAgfQp9Cg
---
{
    "name": "John Doe",
    "email": "johndoe@email.com"
}
```

The same works if the response is a list of users, instead of a single user

```json
GET /users
X-Schema-Map: ewogICAgInNwZWMiOiB7CiAgICAgICAgIl8iOiBbIm5hbWUiLCAiZW1haWwiXQogICAgfQp9Cg
---
GET /users?_map=ewogICAgInNwZWMiOiB7CiAgICAgICAgIl8iOiBbIm5hbWUiLCAiZW1haWwiXQogICAgfQp9Cg
---
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

```none
Header name   : X-Schema-Include
Parameter name: _include
```

This operation is to be used in place of `Schema-Mapping`. If both `Schema-Mapping` and `Schema-Include` are specified, the latter is ignored.

The purpose of this operation is to include information that isn't returned by default. Let's consider the previous example, but we'll say that `teams` property isn't returned by default. So our base request and response would be

```json
GET /users/10
---
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
---
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

So, to include the teams and perform a single request, we would use the `Schema-Include` with the following data

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
---
GET /users/10?_include=ewogICAgInNwZWMiOiB7CiAgICAgICAgIl8iOiBbInRlYW1zIl0KICAgIH0KfQ
---
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

The spec for the schema data can be defined in *Json* or *plain text*. For *Json*, the schema data is then encoded as *base64* or *base64url*. For plain text schema data, there is no encoding.

Let's consider the following resource as an example

```json
GET /users/10
---
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

Let's see a basic example of schema data, to use on a `Schema-Mapping` operation

**Json**
```json
{
    "spec": {
        "_": ["name", "email"]
    }
}
```
**Plain Text**
```text
_[name,email]
```

The first (root) element is the mapping for our result and it can be named whatever we want - the name is indifferent.

We can also specify additional schemas for complex (objects) properties. Still using the above example, we can further describe the properties to retrieve for the teams, saying we are only interested in their ids, not the names. Here, the name of the schema needs to match the name of the property.

**Json**
```json
{
    "spec": {
        "_": ["name", "email", "teams"],
        "teams": ["id"]
    }
}
```
**Plain Text**
```text
_[name,email,teams],teams[id]
```

If there are collisions in the names of the schemas (two properties from different schemas with the same name), we can use the full name instead.

**Json**
```json
{
    "spec": {
        "user": ["name", "email", "teams"],
        "user.teams": ["id"]
    }
}
```
**Plain Text**
```text
user[name,email,teams],user.teams[id]
```

## Schema Data Encoding

As addressed before, *Json* schema data needs to be encoded as a *base64* or a *base64url* string before it can be used on a request.

Let's consider the following schema data

```json
{
    "spec": {
        "_": ["name", "email"]
    }
}
```

The *base64* encoded value for this schema is `eyJzcGVjIjp7Il8iOlsibmFtZSIsICJlbWFpbCJdfX0=`. We would pass this value on a header or query parameter along with the request, for example

```
GET /users
X-Schema-Map: eyJzcGVjIjp7Il8iOlsibmFtZSIsICJlbWFpbCJdfX0=
```

If we are using a query string parameter, we need to either url-encode that value, since *base64* uses characters that aren't safe on a url (`/`, `-` and `=`), or encode the value as a *base64url* instead. The above schema data as a *base64url* is exactly the same, but without the ending `=`.

```
GET /users?_map=eyJzcGVjIjp7Il8iOlsibmFtZSIsICJlbWFpbCJdfX0
```

## Filtering

Introduced in spec version 0.2, filtering allows us to retrieve only data matching a certain criteria. This can be done in two ways: 1. Through the schema data, if we are using *Json* and 2. Through the query string parameters.

An example of a filter in the schema data

```json
{
    "spec": {
        "_": ["name", "email", "address"]
    },
    "filters": {
        "address.city": "london"
    }
}
```

The above schema, used in a `Schema-Mapping` operation, will only retrieve users from the city of London. 

The default operator is `equal` but the following SHOULD also be supported

- `==` (default)
- `!=`
- `>`
- `>=`
- `<`
- `<=`

Another example, with a `greater than` operator

```json
{
    "spec": {
        "_": ["name", "email", "age"]
    },
    "filters": {
        "age": ">25"
    }
}
```

Multiple criteria can be applied as an AND operation.

Filtering through the query string uses a more natural approach. Our first filtering example, through the query string, would be

```json
GET /users?address.city=london
```

As for the additional operators, they are also supported, by using a suffix

- `_ne`
- `_gt`
- `_gte`
- `_lt`
- `_lte`

The age filter example, through the query string

```json
GET /users?age_gt=25
```

Filters can be defined in both the *Json* schema data and the query string, however, the recommendation is to use filters in the schema *when* using a *Json* schema and to use filters in the query string when using *plain text* schema data, since the latter doesn't have a filter spec.

> Filtering through the query string DOES NOT require a `Schema-Mapping` or `Schema-Include` operation.

## Schema Version

The versioning in the schema data is optional, but could be useful to avoid unsupported actions. The way the API reacts is up to the implementation; it can respond with a 400 (bad request) or simply ignore the schema.

The client can also send the schema version in the `X-Schema-Version` header, but it does not take precedence if also defined in the schema data.

The API response MUST include the `X-Schema-Version` header if the request includes schema data, even if the version is not supported and the schema ignored.
