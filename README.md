# ExamUnit proctoring

Documentation for OpenAPI connector

## Definitions

- **Service** - ExamUnit proctoring service
- **Service provider** - Legal entity providing the **Service**
- **Service API** - API to programatically manage the **Service**
- **Candidate application** - Web application where a candidate is being examined
- **Administration application** - Restricted web application accessible for proctors and administrators. Some sections are only available for administrators
- **Incident** - Event happening in the process of candidate examination. May be an action triggered by candidate, by proctor or automatically.

This document is describing the usage of **Service API**.

## Prerequisities

- `Secret key` and `Access key`
    - Request access to administration by contacting **Service provider**.
    - Keys to access the **Service API** can be found in **Aministration application** under `Institute settings -> General`.
    - Make sure that your `Secret key` remains private.

## Authorization

Access to the functionality of **Service API** is granted by following the given mechanism.

- Provide your `Access key` in `Authorization` header using the `token` auth scheme.
    - `Authorization: token 7Q89vDKu7izp5Zd4QGHSdByUTNxcI68GIi0v7zZRrwr4LgNWvfnRBr`
- Sign the request and provide the result withing the `signature` field.
    - Signature is a HMAC-sha256 hash (using the `Secret key` as a key) of the string built with all the request parameters in the format `name=value` joined by `?` and ordered alphabethically by name.
    - Example of code generating the signature can be found below.
- Make sure the `timestamp` field is not older than 1 hour. Expected timestamp timezone is UTC+2.

```php
/**
 * @param string                $secretKey Your secret key
 * @param array<string, scalar> $data      All other request fields in associative array, including timestamp.
 * @return string                          Generated signature which is appended to the data.
 */
function createSignature(string $secretKey, array $data) : string
{
    // aplhabetically sort by key
    \ksort($data);

    $strings = [];

    foreach ($data as $key => $value) {
        // generate the name=value strings
        $strings[] = $key . '=' . getStringValue($value);
    }

    // join the strings using ?
    $stringToSign = \implode('?', $strings);

    // generate signature
    return \hash_hmac('sha256', $stringToSign, $secretKey);
}

function getStringValue(mixed $value) : string
{
    // booleans are converted to their textual form
    if (\is_bool($value)) {
        return $value
            ? 'true'
            : 'false';
    }

    // integers and floats are converted to string
    return (string) $value;
}
```

## Endpoints and fields

Whole functionality of **Service API** is described using [OpenAPI specification](https://swagger.io/specification/). Specification covers all aspects of the API, including endpoints, request structure, response structure and is also accompanied with descriptions and examples. The schema in its current version can be found [in a yaml file](https://github.com/webthinx/examunit-proctoring/blob/main/openapi.yaml) located in the root of this repository.

## Webhooks

TBA intro, configuration

### Content

Content of the webhook payload is following this structure:

```json
{
    "timestamp": "1985-04-12T23:20:50.52Z",
    "candidateId": 255,
    "incidentType": "SESSION_STARTED",
    "additionalData": null
}
```

|Field|Description|Nullable|
|-----|-----------|--------|
|`timestamp`|Datetime information as string in RFC 3339 format|`false`|
|`candidateId`|ID of a candidate as integer|`false`|
|`incidentType`|IncidentType as string enum value|`false`|
|`additionalData`|Some additional data, varies depending on IncidentType|`true`|

#### Incident types

TBA list enum values and corresponding additionalData

### Signature

In order to validate that webhook was indeed sent from the right service, the following mechanism should be applied.

- Validate the webhook signature, which is present in the `X-Signature` header.
    - Signature is a HMAC-sha256 hash (using the `Secret key` as a key) of the request body.
- Make sure the `timestamp` field is not older than 1 hour.

### Retries

TBA 
