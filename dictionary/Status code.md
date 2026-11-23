---
description: The three-digit number in an HTTP response that tells the client whether the request worked and, if not, whose fault it was.
---

A status code is the three-digit number at the start of an [HTTP](./HTTP.md) response. It tells the client the outcome in a form software can act on, and the first digit gives the class.

| Class | Meaning      | Common examples                                                  |
| ----- | ------------ | ---------------------------------------------------------------- |
| 1xx   | Informational | 101 Switching Protocols                                         |
| 2xx   | Success      | 200 OK, 201 Created, 202 Accepted, 204 No Content                |
| 3xx   | Redirection  | 301 Moved Permanently, 304 Not Modified, 307 Temporary Redirect  |
| 4xx   | Client error | 400, 401, 403, 404, 409 Conflict, 422, 429 Too Many Requests     |
| 5xx   | Server error | 500, 502 Bad Gateway, 503 Unavailable, 504 Gateway Timeout       |

The class drives automated behaviour. Clients commonly retry 503 and other temporary failures, and they should not retry most 4xx responses, because the same request will fail the same way. Monitoring usually alerts on the 5xx rate. An API that returns `200 OK` with `{"error": "failed"}` in the body hides failures from all of this: dashboards stay green while users see errors.

Some codes are often confused. 401 means the request lacks valid credentials, so it is an [authentication](./Authentication.md) failure despite being named "Unauthorized". 403 means the server knows who you are and refuses, which is an [authorization](./Authorization.md) failure. Some APIs return 404 instead of 403 so they do not reveal that a private resource exists.

The 5xx codes also point to where a failure happened. 500 usually comes from your application. 502 and 504 usually come from a proxy in front of it, reporting a bad response or no response from the application. If logs show no error for a 502, check the proxy and the connection between it and the app, including [keep-alive](./Keep-alive.md) timeouts.

_Usage:_

"Error rate is flat but support says checkout is broken."

"Checkout returns a 200 with an error in the body, so the alert never fires. Return a proper status code."
