# API Changes

## 1.1

- For the `cv` command, if the supplied parameter is not formatted properly or we cannot return all the changes from that cursor, the response will now be `cv:?`. Previously, the server responded with `c:?`

- If the server determines the supplied connection credentials are no longer valid, previously the server responded with two messages:
    -   `auth:expired`
    -   `auth:{"msg":"Token invalid","code":401}`
-  Now, the server will only respond in the error case with the second message (the JSON error object).

