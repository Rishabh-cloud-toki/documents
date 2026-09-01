JWT Token - Creation and Use

## ?? JWT Authorization in Spring Boot (Microservices Context)

### ? Basics:
- JWT is a compact token issued by an **Identity Provider (IDP)** (e.g., Okta, Auth0).
- Contains **claims** (like email, roles) in the payload.
- Used in **Authorization headers** for **inter-service communication**.

### ? Flow:
1. User logs in via IDP.
2. IDP issues **JWT** (access token).
3. Angular stores and sends the token in `Authorization: Bearer <token>`.
4. Spring Boot backend receives the token and validates it.

### ? Token Validation:
- Spring Security auto-validates JWT if properly configured.
- Use `spring-boot-starter-oauth2-resource-server`.

### ? Setting Roles for Authorization:
- Roles are in the `roles` or `groups` claim.
- Spring extracts them via `JwtAuthenticationConverter`.

```java
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
   JwtGrantedAuthoritiesConverter converter = new JwtGrantedAuthoritiesConverter();
   converter.setAuthoritiesClaimName("roles");
   converter.setAuthorityPrefix("ROLE_");

   JwtAuthenticationConverter jwtConverter = new JwtAuthenticationConverter();
   jwtConverter.setJwtGrantedAuthoritiesConverter(converter);
   return jwtConverter;
}
```

---

## ? Angular + Okta Authentication (OIDC Flow)

### ? 1. Initial Login Trigger:
- User clicks **Login** ? Angular calls:
 ```ts
 oktaAuth.signInWithRedirect();
 ```
- Browser is redirected to:
 ```
 https://<okta-domain>/oauth2/default/v1/authorize
 ```

### ? 2. User Authenticates on Okta Hosted UI:
- Enters username/password.
- Okta redirects to:
 ```
 https://your-app.com/login/callback?code=...&state=...
 ```

### ? 3. Token Exchange (Handled by Okta SDK):
- Angular parses `code` from URL.
- `okta-auth-js` calls Okta `/token` endpoint:
 ```http
 POST /oauth2/default/v1/token
 grant_type=authorization_code
 code=<code>
 redirect_uri=<callback>
 ```
- Okta responds with:
 ```json
 {
   access_token, id_token, refresh_token (if offline_access)
 }
 ```
- SDK stores tokens in memory or localStorage.

---

## ? Redirect After Login

- Default: Returns to original route user tried to access.
- To customize:
 - Use `CustomLoginCallbackComponent` and redirect manually:
   ```ts
   await oktaAuth.handleLoginRedirect();
   this.router.navigate(['/menu']);
   ```

---

## ? Access Token Refresh in Angular

### ? Automatic Token Refresh:
- Enable in `OktaAuth` config:

```ts
tokenManager: {
 autoRenew: true,
 autoRemove: true
}
```

- Also add `offline_access` to scopes:
 ```ts
 scopes: ['openid', 'profile', 'email', 'offline_access']
 ```

### ? Manual Token Refresh:
```ts
await oktaAuth.tokenManager.renew('accessToken');
```

### ? Handle Token Expiration:
```ts
oktaAuth.tokenManager.on('error', err => {
 console.error('Token renewal failed', err);
 // Optionally logout
});
```

### ? Verify App Settings in Okta:
- App must allow refresh tokens.
- Must use **SPA** or **Native App** type.

---

## ? Summary Diagram of Angular + Okta Flow

```
[User] ? clicks Login
  ?
[Angular] ? Redirects to Okta Auth URL
  ?
[Okta Hosted Login Page] ? User logs in
  ?
Redirect to Angular callback with ?code=
  ?
[Okta SDK] ? Exchanges code for tokens
  ?
Tokens stored, user redirected to app (e.g., /menu)
```


