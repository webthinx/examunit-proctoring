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
 * @param array<string, scalar> $data      All request fields in an associative array, including timestamp
 * @return string                          The generated signature
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

// Fetch secret key from secure storage - such as environment, database, ...
$secretKey = 'dummyValue';

// Example of an requst data
$request = [
    'timestamp' => 1698130780.0,
];

// Append generated signature to the request
$request['signature'] = \createSignature($secretKey, $request);

// Json encode whole payload
$signedPayload = \json_encode($request);
```

## Endpoints and Fields

The entire functionality of the **Service API** is described using the [OpenAPI specification](https://swagger.io/specification/). The specification covers all aspects of the API, including endpoints, request structure, response structure, and is also accompanied by descriptions and examples. The schema in its current version can be found in a [YAML file](https://github.com/webthinx/examunit-proctoring/blob/main/openapi.yaml) located in the root of this repository.

## Webhooks

Functions of the **Service API** are limited to pulling the data which is quite unpractical. Webhooks introduce possibility to recieve real-time notifications about events happening in the **Service**. Webhooks are fired when an **Incident** occurs in candidate flow. **Incidents** have various types depending on the nature of the underlying event. Configuration allows to limit listening only to subset of incident types which are relevant to the **Client**.

### Configuration

Webhooks are set up and managed using **Administration application** under `Institute settings -> Webhooks`.

### Content

The content of the webhook payload follows this structure:

```json
{
    "timestamp": "1985-04-12T23:20:50.52Z",
    "triggeredAt": "1985-04-12T23:20:50.52Z",
    "candidateId": 255,
    "incidentType": "SESSION_STARTED",
    "additionalData": null
}
```

| Field           | Description                                                                                                                              | Format                                 | Nullable |
|-----------------|------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------|----------|
| `timestamp`     | Request time information                                                                                                                 | string in RFC 3339 format              | `false`  |
| `trigerredAt`   | Time information describing when the event was originally triggered. This value differs from timestamp in a case of a Retry (see below). | string in RFC 3339 format              | `false`  |
| `candidateId`   | ID of a candidate                                                                                                                        | integer                                | `false`  |
| `incidentType`  | Type of an event                                                                                                                         | string enum value                      | `false`  |
| `additionalData`| Some additional data                                                                                                                     | varies depending on the `incidentType` | `true`   |

### Incident Types

List of all possible incident types and their corresponsing additional data.

| Incident Type                      | Fired when                                                             | Additional data               | Note                                                                                                                                                        |
|------------------------------------|------------------------------------------------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| MANUAL                             | Proctor reports custom incident                                        | string containing the message |                                                                                                                                                             |
| SYSTEM_CHECK_STEP_CHANGED          |                                                                        | null                          |                                                                                                                                                             |
| IDENTITY_CHECK_STEP_CHANGED        |                                                                        | null                          |                                                                                                                                                             |
| SESSION_JOINED                     | A candidate starts his examination process                             | null                          |                                                                                                                                                             |
| SESSION_APPROVAL_REQUESTED         | A candidate completes his setup and requests to be allowed into exam   | null                          |                                                                                                                                                             |
| SESSION_APPROVED                   | A proctor approves candidate to start his exam                         | null                          |                                                                                                                                                             |
| SESSION_APPROVAL_REVERTED          | A candidate performs action which voids his approval                   | null                          |                                                                                                                                                             |
| SESSION_STARTED                    | A candidate starts his exam                                            | null                          |                                                                                                                                                             |
| SESSION_FINISHED                   | A candidate successfully completes his exam                            | null                          | Trigged by `Candidate::finish` request to **Service API** which notifies the system that exam was successfully finished                                     |
| SESSION_DISMISSED                  | A proctor dismisses the candidate                                      | null                          |                                                                                                                                                             |
| SESSION_CLOSED                     | A proctor marks the candidate examination as completed                 | null                          |                                                                                                                                                             |
| SESSION_CLOSED_AUTOMATICALLY       | **Service** detects session to be closed after an hour of inactivity   | null                          |                                                                                                                                                             |
| EVALUATION_CREATED                 | A proctor creates his proctoring evaluation                            | null                          |                                                                                                                                                             |
| SESSION_WAITING_DETECTED           | A candidate is waiting for approval for too long                       | null                          |                                                                                                                                                             |
| CONNECTED                          | A candidate connects                                                   | null                          |                                                                                                                                                             |
| DISCONNECTED                       | A candidate disconnects                                                | null                          |                                                                                                                                                             |
| MOBILE_CONNECTED                   | A candidates mobile (secondary camera) connects                        | null                          |                                                                                                                                                             |
| MOBILE_DISCONNECTED                | A candidates mobile (secondary camera) disconnects                     | null                          |                                                                                                                                                             |
| CAMERA_STARTED                     |                                                                        | null                          |                                                                                                                                                             |
| CAMERA_STOPPED                     |                                                                        | null                          |                                                                                                                                                             |
| AUDIO_STARTED                      |                                                                        | null                          |                                                                                                                                                             |
| AUDIO_STOPPED                      |                                                                        | null                          |                                                                                                                                                             |
| MOBILE_CAMERA_STARTED              |                                                                        | null                          |                                                                                                                                                             |
| MOBILE_CAMERA_STOPPED              |                                                                        | null                          |                                                                                                                                                             |
| SCREENSHARE_STARTED                |                                                                        | null                          |                                                                                                                                                             |
| SCREENSHARE_STOPPED                |                                                                        | null                          |                                                                                                                                                             |
| RECORDINGS_STARTED                 |                                                                        | null                          |                                                                                                                                                             |
| PROCTOR_ASSIGNED                   |                                                                        | null                          |                                                                                                                                                             |
| PROCTOR_CONNECTED                  | A proctor connects                                                     | null                          |                                                                                                                                                             |
| PROCTOR_DISCONNECTED               | A proctor disconnects                                                  | null                          |                                                                                                                                                             |
| PROCTOR_LOSING_CONNECTION_DETECTED | **Service** detects proctor is inactivite during proctoring session    | null                          |                                                                                                                                                             |
| ADMIN_SUBSCRIBED                   | Administrator joins proctoring session                                 | null                          |                                                                                                                                                             |
| ADMIN_UNSUBSCRIBED                 | Administrator leaves proctoring session                                | null                          |                                                                                                                                                             |
| INVITATION_EMAIL_SENT              | **Service** sends invitation email to candidate                        | null                          | Trigged by `Exam::send_emails` request to **Service API**                                                                                                   |
| SYSTEM_CHECK_EMAIL_SENT            | **Service** sends follow-up email with systemcheck reminder            | null                          | Enabled by `Exam::send_emails` request to **Service API**, the email is send 48 hours before exam starts and only if system check was not already completed |
| INVITATION_EMAIL_RESENT            | Administrator resends the invitation email to candidate                | null                          |                                                                                                                                                             |

### Signature

To validate that the webhook was sent from a trusted service, the following mechanism should be applied.

- Validate the webhook signature, which is present in the `X-Signature` header.
    - The signature is an HMAC-SHA256 hash (using the `Secret key` as a key) of the request body.
- Ensure that the `timestamp` field is not older than 1 hour.

### Retries

TBA (To Be Announced)
