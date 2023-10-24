# ExamUnit Proctoring

Documentation for the OpenAPI Connector

## Definitions

- **Service** - ExamUnit proctoring service
- **Service provider** - A legal entity that provides the **Service**
- **Service API** - An API for programmatically managing the **Service**
- **Client** - A third-party application connecting to the **Service API**
- **Candidate application** - A web application where a candidate undergoes examination
- **Administration application** - A restricted web application accessible to proctors and administrators. Some sections are exclusively available to administrators.
- **Incident** - An event that occurs during the candidate examination process. It may be triggered by the candidate, the proctor, or an automated action.

This document describes the usage of the **Service API**.

## Prerequisites

- `Secret key` and `Access key`
    - Request access to the administration by contacting the **Service provider**.
    - Keys to access the **Service API** can be found in the **Administration application** under `Institute settings -> General`.
    - Ensure the confidentiality of your `Secret key`.

## Authorization

Access to the functionality of the **Service API** is granted to **Clients** implementing the following authentication mechanism.

- Include your `Access key` in the `Authorization` header using the `token` authentication scheme.
    - Example: `Authorization: token 7Q89vDKu7izp5Zd4QGHSdByUTNxcI68GIi0v7zZRrwr4LgNWvfnRBr`
- Sign the request and provide the result in the `signature` field.
    - The signature is an HMAC-SHA256 hash (using the `Secret key` as the key) of the string built with all the request parameters in the format `name=value`, joined by `?`, and ordered alphabetically by name.
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

// Fetch the secret key from secure storage - such as the environment, database, ...
$secretKey = 'dummyValue';

// Example of a request data
$request = [
    'timestamp' => 1698130780.0,
];

// Append the generated signature to the request
$request['signature'] = \createSignature($secretKey, $request);

// JSON encode the whole payload
$signedPayload = \json_encode($request);
```

## Endpoints and Fields

The entire functionality of the **Service API** is described using the [OpenAPI specification](https://swagger.io/specification/). The specification covers all aspects of the API, including endpoints, request structure, response structure, and is also accompanied by descriptions and examples. The current version of the schema can be found in a [YAML file](https://github.com/webthinx/examunit-proctoring/blob/main/openapi.yaml) located in the root of this repository.

## Webhooks

Initially, the functions of the **Service API** are limited to pulling data, which can be somewhat impractical. Webhooks introduce the possibility to receive real-time notifications about events happening in the **Service**. Webhooks are triggered when an **Incident** occurs in the candidate's flow. **Incidents** have various types depending on the nature of the underlying event. Configuration allows the **Client** to limit listening only to a relevant subset of incident types.

### Configuration

Webhooks are set up and managed using the **Administration application** under `Institute settings -> Webhooks`.

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
| `triggeredAt`   | Time information describing when the event was originally triggered. This value differs from the timestamp in the case of a Retry (see below). | string in RFC 3339 format              | `false`  |
| `candidateId`   | ID of a candidate                                                                                                                        | integer                                | `false`  |
| `incidentType`  | Type of an event                                                                                                                         | string enum                            | `false`  |
| `additionalData`| Some additional data                                                                                                                     | varies depending on the `incidentType` | `true`   |

### Incident Types

Here's a list of all possible incident types and their corresponding additional data.

| Incident Type                        | Fired when                                                             | Additional data               | Note                                                                                                                                                        |
|--------------------------------------|------------------------------------------------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `MANUAL`                             | Proctor reports a custom incident                                      | string with the message       |                                                                                                                                                             |
| `SYSTEM_CHECK_STEP_CHANGED`          | A candidate changes step in system-check flow                          | string enum                   |                                                                                                                                                             |
| `IDENTITY_CHECK_STEP_CHANGED`        | A candidate changes step in indentity-check flow                       | string enum                   |                                                                                                                                                             |
| `SESSION_JOINED`                     | A candidate starts their examination process                           | null                          |                                                                                                                                                             |
| `SESSION_APPROVAL_REQUESTED`         | A candidate completes their setup and requests to be allowed into the exam   | null                          |                                                                                                                                                             |
| `SESSION_APPROVED`                   | A proctor approves the candidate to start their exam                   | null                          |                                                                                                                                                             |
| `SESSION_APPROVAL_REVERTED`          | A candidate performs an action that voids their approval               | null                          |                                                                                                                                                             |
| `SESSION_STARTED`                    | A candidate starts their exam                                          | null                          |                                                                                                                                                             |
| `SESSION_FINISHED`                   | A candidate successfully completes their exam                          | null                          | Triggered by `Candidate::finish` request to the **Service API**, which notifies the system that the exam was successfully finished                          |
| `SESSION_DISMISSED`                  | A proctor dismisses the candidate                                      | null                          |                                                                                                                                                             |
| `SESSION_CLOSED`                     | A proctor marks the candidate examination as completed                 | null                          |                                                                                                                                                             |
| `SESSION_CLOSED_AUTOMATICALLY`       | The **Service** detects the session to be closed after an hour of inactivity   | null                          |                                                                                                                                                             |
| `EVALUATION_CREATED`                 | A proctor creates their proctoring evaluation                          | null                          |                                                                                                                                                             |
| `SESSION_WAITING_DETECTED`           | A candidate is waiting for approval for too long                       | null                          |                                                                                                                                                             |
| `CONNECTED`                          | A candidate connects                                                   | null                          |                                                                                                                                                             |
| `DISCONNECTED`                       | A candidate disconnects                                                | null                          |                                                                                                                                                             |
| `MOBILE_CONNECTED`                   | A candidate's mobile (secondary camera) connects                       | null                          |                                                                                                                                                             |
| `MOBILE_DISCONNECTED`                | A candidate's mobile (secondary camera) disconnects                    | null                          |                                                                                                                                                             |
| `CAMERA_STARTED`                     | A candidate's web camera started streaming                             | null                          |                                                                                                                                                             |
| `CAMERA_STOPPED`                     | A candidate's web camera stopped streaming                             | null                          |                                                                                                                                                             |
| `AUDIO_STARTED`                      | A candidate's microphone started streaming                             | null                          |                                                                                                                                                             |
| `AUDIO_STOPPED`                      | A candidate's microphone stopped streaming                             | null                          |                                                                                                                                                             |
| `MOBILE_CAMERA_STARTED`              | A candidate's mobile camera started streaming                          | null                          |                                                                                                                                                             |
| `MOBILE_CAMERA_STOPPED`              | A candidate's mobile camera stopped streaming                          | null                          |                                                                                                                                                             |
| `SCREENSHARE_STARTED`                | A candidate's screenshare started streaming                            | null                          |                                                                                                                                                             |
| `SCREENSHARE_STOPPED`                | A candidate's screenshare stopped streaming                            | null                          |                                                                                                                                                             |
| `RECORDINGS_STARTED`                 | A candidate initiated recording of streams                             | null                          |                                                                                                                                                             |
| `PROCTOR_ASSIGNED`                   | The **Service** assigned proctor to candidate                          | null                          |                                                                                                                                                             |
| `PROCTOR_CONNECTED`                  | A proctor connects                                                     | null                          |                                                                                                                                                             |
| `PROCTOR_DISCONNECTED`               | A proctor disconnects                                                  | null                          |                                                                                                                                                             |
| `PROCTOR_LOSING_CONNECTION_DETECTED` | The **Service** detects that the proctor is inactive during the proctoring session    | null                          |                                                                                                                                                             |
| `ADMIN_SUBSCRIBED`                   | An administrator joins the proctoring session                                 | null                          |                                                                                                                                                             |
| `ADMIN_UNSUBSCRIBED`                 | An administrator leaves the proctoring session                                | null                          |                                                                                                                                                             |
| `INVITATION_EMAIL_SENT`              | The **Service** sends an invitation email to the candidate                    | null                          | Triggered by the `Exam::send_emails` request to the **Service API**                                                                                                   |
| `SYSTEM_CHECK_EMAIL_SENT`            | The **Service** sends a follow-up email with a system check reminder          | null                          | Enabled by the `Exam::send_emails` request to the **Service API**, the email is sent 48 hours before the exam starts and only if the system check was not already completed |
| `INVITATION_EMAIL_RESENT`            | An administrator resends the invitation email to the candidate                | null                          |                                                                                                                                                             |

Enum values of SystemCheck / IdentityCheck steps:

- `START`
- `MICROPHONE`
- `SPEAKERS`
- `BROWSER_TABS`
- `SCREENSHARE`
- `WEB_CAM`
- `MOBILE_CAM`
- `ROOM_CHECK`
- `FACE_PHOTO`
- `ID_CARD`
- `FINISH`

### Signature

To validate that the webhook was sent from a trusted service, apply the following mechanism.

- Validate the webhook signature, which is present in the `X-Signature` header.
    - The signature is an HMAC-SHA256 hash (using the `Secret key` as the key) of the request body.
- Ensure that the `timestamp` field is not older than 1 hour (or other reasonable amount of time).

### Status handling and retries

- The expected response status is any 2xx status.
- Webhooks do not follow redirects.
- When a 4xx status is encountered, the webhook is considered sent and will not be retried.
- When a 5xx status is encountered, the **Service** will perform multiple additional attempts after a reasonable time interval.
    - The `timestamp` field in the payload changes to the time when each attempt was sent.
    - The `triggeredAt` field maintains its original value.
