---
title: Example of an OAuth2 flow (Authorization Code Grant)
date: 2024-06-11T12:29:19-03:00
draft: false
language: en
featured_image: ../assets/images/featured/featured-oauth2-flow.jpg
summary:
  Single sign-on is a must nowadays. Very few people have the patience to login into a thousand different sites with
  different logins and passwords. For example, if you are a company with many different applications, you will
  definitively gain praise for implementing a single sign-on for all of them. That said, we'll now describe an 
  example of an OAuth2 flow. Development and Infrastructure people should understand it.
categories: Tech-Blog
tags:
- oauth2
- single sign-on
- authentication
---

Single sign-on is a must nowadays. Very few people have the patience to login into a thousand different sites with
different logins and passwords. For example, if you are a company with many different applications, you will
definitively gain praise for implementing a single sign-on for all of them. That said, we'll now describe an 
example of an OAuth2 workflow. Development and Infrastructure people should understand it.

The workflow we'll see here is based on the
[Authorization Code Grant](https://oauth.net/2/grant-types/authorization-code/), which is a grant type that is
used and recommended by many.

## Step by step into the flow

1. The client accesses the login URL:

    ```plain
    GET https://www.example.com/login
    ```

2. If the user isn't logged in, it won't send any session cookies in the first item. Detecting this, the application
   creates an internal `code verifier`. I'll use python here to demonstrate:

   ```python
    code_verifier = base64.urlsafe_b64encode(os.urandom(40)).decode('utf-8')
    code_verifier = re.sub('[^a-zA-Z0-9]+', '', code_verifier)
    ```

    With this `code verifier`, the application also creates a `code challenge` using `SHA256`:

    ```python
    code_challenge = hashlib.sha256(code_verifier.encode('utf-8')).digest()
    code_challenge = base64.urlsafe_b64encode(code_challenge).decode('utf-8')
    code_challenge = code_challenge.replace('=', '')
    ```

3. The application redirects the user to the Single Sign-On server using `HTTP 302`. This single sign-on server should
   be already configured with a client/secret. The configuration is outside of the scope of this tutorial since it's
   demonstrating only the flow.

   ```plain
    HTTP/1.1 302
    Location: https://sso.example.com/oauth2/auth
    ```

    This redirect comes with these query strings on the GET method:

    ```plain
    scope="openid profile email address phone"
    response_type=code
    client_id=my-client-id
    state=NV3YV4FE5FJTNDQQXAUJ
    code_challenge=W3P1WSSG9sJFi5ysBeEf2kkYRTOKmfr6m8UZN3vNkQw
    code_challenge_method=S256
    redirect_uri
    ```

    - `scope` - Scopes to return in the token
    - `response_type` - Type of flow (code = authorization grant type)
    - `client_id` - The public client ID provided by the SSO server configuration
    - `state` - A random string which is used to verify requests are coming from the same flow between requests
    - `code_challenge` - The code challenge that was generated on step 2
    - `code_challenge_method` - The code challenge method used (SHA256)
    - `redirect_uri` - URL (encoded) which the SSO server will redirect to after the authentication is complete

    In this case, the backend URL that will receive the sucessful authentication (and thus the token) is `https://www.example.com/login/callback`.

4. The SSO server receives the request and presents the user with the authentication method. This part is totally
   dependent on the SSO server configuration. The method could be anything: username/password, with or without
   OTP (*One Time Password*), a magic link through email, etc.

    Regardless of the method, after the user completes the authentication, the SSO server redirects to the
    `redirect_uri` requested on the previous step. This is called the `callback`:

    ```plain
    HTTP/1.1 302
    Location: https://www.example.com/login/callback
    ```

    This redirect comes with these query strings on the GET method:

    ```plain
    state=NV3YV4FE5FJTNDQQXAUJ
    session_state=4a5e83f3-82df-4051-9ae3-a9102004cf64
    iss=https%3A%2F%2Fsso.example.com%2Foauth2%2Fclient
    code=1dff4f51-d4bf-439a-b89f-8b6f698004d9.4a5e83f3-82df-40
    ```

    - `state` - The random string from the previous step
    - `session_state` - Usually an ID from the SSO server session (server specific)
    - `iss` - The issuer is the SSO server URL which generated this request
    - `code` - The authorization code (most important)

    The user application receives the `code` parameter, which is the code that will be used to get the access token.

5. The application's callback function internally requests the SSO server for an access token. Notice that the client
   doesn't participate in this, the request is between the application and the SSO server, and demands that network
   access between them is also available.

    A POST request is made by the application:

    ```plain
    POST https://sso.example.com/oauth2/token
    ```

    Inside the POST requests, there's this data. Check the comment besides each line to see an explanation:

    ```plain
    grant_type=authorization_code
    client_id=my-client-id
    client_secret=my-secret
    redirect_uri=https%3A%2F%2Fwww.example.com%2Flogin%2Fcallback
    code=1dff4f51-d4bf-439a-b89f-8b6f698004d9.4a5e83f3-82df-40
    code_verifier=code_verifier
    ```

    - `grant_type=authorization_code` - Type of flow
    - `client_id=my-client-id` - The public client ID provided by the SSO server configuration
    - `client_secret=my-secret` - The private client secret provided by the SSO server configuration
    - `redirect_uri=https%3A%2F%2Fwww.example.com%2Flogin%2Fcallback` - URL (encoded) which the SSO server redirected in previous step
    - `code=1dff4f51-d4bf-439a-b89f-8b6f698004d9.4a5e83f3-82df-40` - The authorization code from the previous step
    - `code_verifier=code_verifier` - The code verifier that was generated in the step 2


6. The SSO server will return the access and id tokens (JWT format). You can read these tokens and grant
   authorization to use the application.

    An example of a response:

    ```json
    {
      "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJyenFSbklubkFXR1RISEdpN2pUXzFfTVZZbDhxaGR3cXNoZGc1X2ZiNjFJIn0.eyJleHAiOjE3MTgxMjYzNTQsImlhdCI6MTcxODEyNjA1NCwiYXV0aF90aW1lIjoxNzE4MTI1MTMxLCJqdGkiOiJlNjAzNzEzMS04YTJiLTQyNTgtOTk4Mi04MTcwMDkzOGM5MTciLCJpc3MiOiJodHRwOi8va2MubG9jYWxkb21haW46ODA4MC9yZWFsbXMvZXhhbXBsZSIsImF1ZCI6ImFjY291bnQiLCJzdWIiOiJkYzI0Mjk0Ny0yODI4LTQ5NDgtOTRmMy0zODExMTZkZjc1ZTUiLCJ0eXAiOiJCZWFyZXIiLCJhenAiOiJlaXRjaC1leGFtcGxlIiwibm9uY2UiOiJPTFM4VjBISVNVN1VaNE5MUUdWNSIsInNlc3Npb25fc3RhdGUiOiI3NzU1ZGFhYi1lZWEzLTRkZGQtOTUyOC0zZTIxZjQ2MTdjNjAiLCJhY3IiOiIwIiwiYWxsb3dlZC1vcmlnaW5zIjpbImh0dHA6Ly9vZ3MubG9jYWxkb21haW46OTAwMCJdLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJkZWZhdWx0LXJvbGVzLWV4YW1wbGUiLCJ1bWFfYXV0aG9yaXphdGlvbiJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoib3BlbmlkIGVtYWlsIGFkZHJlc3MgcGhvbmUgcHJvZmlsZSIsInNpZCI6Ijc3NTVkYWFiLWVlYTMtNGRkZC05NTI4LTNlMjFmNDYxN2M2MCIsImVtYWlsX3ZlcmlmaWVkIjp0cnVlLCJhZGRyZXNzIjp7fSwibmFtZSI6IlVzZXIgT25lIiwicHJlZmVycmVkX3VzZXJuYW1lIjoidXNlcjEiLCJnaXZlbl9uYW1lIjoiVXNlciIsImZhbWlseV9uYW1lIjoiT25lIiwiZW1haWwiOiJ1c2VyMUBleGFtcGxlLmNvbSJ9.CKiVeaic0XMqh10C_YKFN4rb9aFfX5-8cqFC2W3cmqPDqZoQP9P-z_D-suw9hwERBwyolEJvL-zW8qa617njTuiJFVKJ1EXjIY3sxNoIHoGwwPs6mANbJsgqNQCwlwuKhXw5FbJdwMES4uqulNm57NBP6ZXU-EVFLhNxRlRVp5-pjpzjs-oeHanrW8EUVFwMsCQMPCWhY1vO_2bDZMwMvfjB0CGEc7RKJ_KhpG-VpXoNb4Kte-e19DMRiPnCWB97QLyzRLtEEhjHWm67n2j83lE1ovvnEFsqsMMwQeYHLmH6SOrbC1MYQEu-ZhGRe3KJFz2XvzXHvnbNQAgiGEVUPA",
      "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJyenFSbklubkFXR1RISEdpN2pUXzFfTVZZbDhxaGR3cXNoZGc1X2ZiNjFJIn0.eyJleHAiOjE3MTgxMjYzNTQsImlhdCI6MTcxODEyNjA1NCwiYXV0aF90aW1lIjoxNzE4MTI1MTMxLCJqdGkiOiI4OGQ3YTlkYy05YzI1LTQ0NTEtYjNiNC01NTRiNjVjMDBhYzUiLCJpc3MiOiJodHRwOi8va2MubG9jYWxkb21haW46ODA4MC9yZWFsbXMvZXhhbXBsZSIsImF1ZCI6ImVpdGNoLWV4YW1wbGUiLCJzdWIiOiJkYzI0Mjk0Ny0yODI4LTQ5NDgtOTRmMy0zODExMTZkZjc1ZTUiLCJ0eXAiOiJJRCIsImF6cCI6ImVpdGNoLWV4YW1wbGUiLCJub25jZSI6Ik9MUzhWMEhJU1U3VVo0TkxRR1Y1Iiwic2Vzc2lvbl9zdGF0ZSI6Ijc3NTVkYWFiLWVlYTMtNGRkZC05NTI4LTNlMjFmNDYxN2M2MCIsImF0X2hhc2giOiJ2TjNadzBXZFdqeDNScElEM2tKTU5BIiwiYWNyIjoiMCIsInNpZCI6Ijc3NTVkYWFiLWVlYTMtNGRkZC05NTI4LTNlMjFmNDYxN2M2MCIsImVtYWlsX3ZlcmlmaWVkIjp0cnVlLCJhZGRyZXNzIjp7fSwibmFtZSI6IlVzZXIgT25lIiwicHJlZmVycmVkX3VzZXJuYW1lIjoidXNlcjEiLCJnaXZlbl9uYW1lIjoiVXNlciIsImZhbWlseV9uYW1lIjoiT25lIiwiZW1haWwiOiJ1c2VyMUBleGFtcGxlLmNvbSJ9.NX_v5a3sNYa1Actv6fETHPlTQ4zHWYvKC60V5Al2jYtqZFDDZcLI8E-v-T9gw7xFqt8qM0URd61BNQd06CLldsLWpWIVJeSpvljIeFRVCm_tA7YoyIs-YcxJLznp0fHYPMVnr04UTpCSCAMrdFmjv_niqWugInIu8XzHNWGb0bnQhS96FtrJceYh5cL--sI-WZXib-cXjKzQDyFJb0DwtlOJcN3IL-NbKy58oLnbNheuSKbwbLfTPUINvvpn7I3oLRqdlkRzk2vzZuqe9iswV5l34W6zVtHLkZXsGPtpVt_g4xc6u6b_2KTtgCdCsBoMUAvxQevPEfBL17HSfNBA7w",
      "expires_in": 60,
      [...]
    }
    ```

    With python, you can decode these tokens with this code:

    ```python
    def _b64_decode(data):
        data += '=' * (4 - len(data) % 4)
        return base64.b64decode(data).decode('utf-8')

    def jwt_payload_decode(jwt):
        _, payload, _ = jwt.split('.')
        return json.loads(_b64_decode(payload))

    access_token = 'eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lk...etc...'
    jwt_payload_decode(access_token)
    ```

    Example output:

    ```json
    {
      "header": {
        "typ": "JWT",
        "alg": "RS256",
        "kid": "rzqRnInnAWGTHHGi7jT_1_MVYl8qhdwqshdg5_fb61I"
      },
      "payload": {
        "acr": "0",
        "address": {},
        "allowed-origins": [
          "http://ogs.localdomain:9000"
        ],
        "aud": "account",
        "auth_time": 1718125131,
        "azp": "eitch-example",
        "email": "user1@example.com",
        "email_verified": true,
        "exp": 1718126354,
        "family_name": "One",
        "given_name": "User",
        "iat": 1718126054,
        "iss": "http://kc.localdomain:8080/realms/example",
        "jti": "e6037131-8a2b-4258-9982-81700938c917",
        "name": "User One",
        "nonce": "OLS8V0HISU7UZ4NLQGV5",
        "preferred_username": "user1",
        "realm_access": {
          "roles": [
            "offline_access",
            "default-roles-example",
            "uma_authorization"
          ]
        },
        "resource_access": {
          "account": {
            "roles": [
              "manage-account",
              "manage-account-links",
              "view-profile"
            ]
          }
        },
        "scope": "openid email address phone profile",
        "session_state": "7755daab-eea3-4ddd-9528-3e21f4617c60",
        "sid": "7755daab-eea3-4ddd-9528-3e21f4617c60",
        "sub": "dc242947-2828-4948-94f3-381116df75e5",
        "typ": "Bearer"
      }
    }
    ```

    As the JWT is signed and the application trusts the SSO server, it can use the token to determine what username
    is using, which permissions, its email, etc.

## Summary

What the authorization code grant type does:

1. Client requests the web page with a browser

2. The web page redirects to the SSO server

3. The SSO server authenticates the user locally through forms or other methods (including *MFA*)

4. The SSO server redirects the user to the application along with an authorization code

5. The application internally requests an access token to the SSO server, using the authorization code

6. The SSO server responds with a valid, signed JWT token.

## Conclusion and references

Open Source Single Sign-On server options:

- [Keycloak](https://www.keycloak.org/)
- [Authentik](https://goauthentik.io/)
- [Authelia](https://www.authelia.com/)
- [Gluu](https://gluu.org/)
- [Casdoor](https://casdoor.org/)
- [Aerobase](https://aerobase.io/)

References:

- [OAuth2](https://oauth.net/2/)
- [An online tool to generate code verifier and code challenge for OAuth with PKCE.](https://tonyxu-io.github.io/pkce-generator/)
- [Step by Step OAuth 2.0 Authorization Code Flow with PKCE](https://www.stefaanlippens.net/oauth-code-flow-pkce.html)
