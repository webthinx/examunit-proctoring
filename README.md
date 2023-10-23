# ExamUnit proctoring

Documentation for OpenAPI connector

## Definitions

- **Service** - ExamUnit proctoring service
- **Service provider** - Legal entity providing the **Service**
- **Service API** - API to programatically manage the **Service**
- **Candidate** - 
- **Exam** - 
- **Institute** -
- **Proctor** - 
- **Institute administrator** - 
- **Candidate application** - Web application where a **Candidate** is being examined
- **Administration application** - Restricted web application accessible for **Proctors** and **Institute administrators**. Some sections are only available for **Institute administrators**.

This document is describing the usage of **Service API**.

## Prerequisities

- `Secret key` and `Access key`
    - Request access to administration by contacting **Service provider**.
    - Keys to access the **Service API** for your **Institute** can be found in **Aministration application** under `Institute settings -> General`.
    - Make sure that your `Secret key` remains private.

## Authorization

Access to the **Service API** is granted by following the given mechanism.

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
        $strings[] = $key . '=' . self::getStringValue($value);
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
