# ExamUnit Proctoring

Documentation for the OpenAPI Connector

## Definitions

- **Service** - ExamUnit proctoring service
- **Service provider** - A legal entity that provides the **Service**
- **Service API** - An API for programmatically managing the **Service**
- **Client** - A foreign application connecting to the **Service API**
- **Candidate application** - A web application where a candidate is being examined
- **Administration application** - A restricted web application accessible to proctors and administrators. Some sections are only available to administrators
- **Incident** - An event that occurs in the process of candidate examination. It may be an action triggered by the candidate, the proctor, or an automated action.

This document describes the usage of the **Service API**.

## Prerequisites

- `Secret key` and `Access key`
    - Request access to administration by contacting the **Service provider**.
    - Keys to access the **Service API** can be found in the **Administration application** under `Institute settings -> General`.
    - Ensure that your `Secret key` remains private.

## Authorization

Access to the functionality of the **Service API** is granted to **Clients** implementing the following authentication mechanism.

- Provide your `Access key` in the `Authorization` header using the `token` auth scheme.
    - Example: `Authorization: token 7Q89vDKu7izp5Zd4QGHSdByUTNxcI68GIi0v7zZRrwr4LgNWvfnRBr`
- Sign the request and provide the result in the `signature` field.
    - The signature is an HMAC-SHA256 hash (using the `Secret key` as a key) of the string built with all the request parameters in the format `name=value`, joined by `?`, and ordered alphabetically by name.
    - An example of code generating the signature can be found below.
- Ensure that the `timestamp` field is not older than 1 hour. The expected timestamp timezone is UTC+2.

```php
/**
 * @param string                $secretKey Your secret key
 * @param array<string, scalar> $data      All other request fields in an associative array, including timestamp.
 * @return string                          The generated signature, which is appended to the data.
 */
function createSignature(string $secretKey, array $data) : string
{
    // Alphabetically sort by key
    \ksort($data);

    $strings = [];

    foreach ($data as $key => $value) {
        // Generate the name=value strings
        $strings[] = $key . '=' . \getStringValue($value);
    }

    // Join the strings using ?
    $stringToSign = \implode('?', $strings);

    // Generate the signature
    return \hash_hmac('sha256', $stringToSign, $secretKey);
}

function getStringValue(string|int|float|bool $value) : string
{
    // Booleans are converted to their textual form
    if (\is_bool($value)) {
        return $value
            ? 'true'
            : 'false';
    }

    // Integers and floats are converted to strings
    return (string) $value;
}
```

## Endpoints and Fields

The entire functionality of the **Service API** is described using the [OpenAPI specification](https://swagger.io/specification/). The specification covers all aspects of the API, including endpoints, request structure, response structure, and is also accompanied by descriptions and examples. The schema in its current version can be found in a [YAML file](https://github.com/webthinx/examunit-proctoring/blob/main/openapi.yaml) located in the root of this repository.

## Webhooks

Introduction and Configuration To Be Announced (TBA)

### Content

The content of the webhook payload follows this structure:

```json
{
    "timestamp": "1985-04-12T23:20:50.52Z",
    "candidateId": 255,
    "incidentType": "SESSION_STARTED",
    "additionalData": null
}
```

| Field           | Description                                    | Nullable |
|-----------------|-----------------------------------------------|----------|
| `timestamp`     | Datetime information as a string in RFC 3339 format | `false`  |
| `candidateId`   | ID of a candidate as an integer              | `false`  |
| `incidentType`  | IncidentType as a string enum value          | `false`  |
| `additionalData`| Some additional data, which varies depending on the IncidentType | `true` |

#### Incident Types

TBA: List of enum values and corresponding additionalData

### Signature

To validate that the webhook was sent from a trusted service, the following mechanism should be applied.

- Validate the webhook signature, which is present in the `X-Signature` header.
    - The signature is an HMAC-SHA256 hash (using the `Secret key` as a key) of the request body.
- Ensure that the `timestamp` field is not older than 1 hour.

### Retries

TBA (To Be Announced)
