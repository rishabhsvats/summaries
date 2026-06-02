# Kubernetes Federated Client Authentication in Keycloak

## End-to-End Flow

### Realm/client setup

You add a kubernetes identity provider with an issuer.

You configure confidential clients to use authenticator `federated-jwt`, and set:
- `jwt.credential.issuer` = IDP alias
- `jwt.credential.sub` = expected SA subject (`system:serviceaccount:<ns>:<sa>`)

**DefaultClientAssertionStrategy.java** (Lines 21-42):
```java
String issuer = clientAssertionState.getToken().getIssuer();
String federatedClientId =  clientAssertionState.getToken().getSubject();
IdentityProviderModel identityProvider = lookupProvider.lookupIdentityProviderFromIssuer(context.getSession(), IdentityProviderType.CLIENT_ASSERTION, issuer);
// ...
ClientModel client = lookupProvider.lookupClientFromClientAttributes(
        context.getSession(),
        Map.of(
                FederatedJWTClientAuthenticator.JWT_CREDENTIAL_SUBJECT_KEY, federatedClientId,
                FederatedJWTClientAuthenticator.JWT_CREDENTIAL_ISSUER_KEY, identityProvider.getAlias()
        )
);
```

### Client calls token endpoint

- Sends `grant_type=client_credentials`
- Sends SA token as `client_assertion`
- Uses `client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer`

### Federated authenticator kicks in

`FederatedJWTClientAuthenticator` handles third-party-signed JWT assertions (issuer != subject), finds the right lookup strategy, resolves IDP + internal client, and calls the IDP-specific verifier.

### Kubernetes verifier validates assertion

`KubernetesIdentityProvider` uses `FederatedJWTClientValidator` with:
- expected issuer = configured Kubernetes issuer
- allowed clock skew = IDP config
- max assertion lifetime = configured value, else 3600s default

**KubernetesIdentityProvider.java** (Lines 31-40):
```java
public boolean verifyClientAssertion(ClientAuthenticationFlowContext context) throws Exception {
    FederatedJWTClientValidator validator = new FederatedJWTClientValidator(context, this::verifySignature, config.getIssuer(), config.getAllowedClockSkew(), true);
    if (config.getFederatedClientAssertionMaxExpiration() != 0) {
        validator.setMaximumExpirationTime(config.getFederatedClientAssertionMaxExpiration());
    } else {
        validator.setMaximumExpirationTime(3600); // Kubernetes defaults to 1 hour
    }
    return validator.validate();
}
```

### Signature verification against Kubernetes JWKS

Reads `kid` + `alg` from JWT header.

Loads key from key cache; if missing, fetches:
- `<issuer>/.well-known/openid-configuration`
- then the `jwks_uri`

If Keycloak is running in Kubernetes and `/var/run/secrets/kubernetes.io/serviceaccount/token` exists, it uses that token to auth these metadata/JWKS calls only when that token issuer matches configured issuer.

**KubernetesIdentityProvider.java** (Lines 42-59):
```java
String kid = header.getKeyId();
String alg = header.getRawAlgorithm();
String modelKey = PublicKeyStorageUtils.getIdpModelCacheKey(validator.getContext().getRealm().getId(), config.getInternalId());
KeyWrapper publicKey = keyStorage.getPublicKey(modelKey, kid, alg, new KubernetesJwksEndpointLoader(session, config.getIssuer()));
SignatureProvider signatureProvider = session.getProvider(SignatureProvider.class, alg);
// ...
return signatureProvider.verifier(publicKey).verify(jws.getEncodedSignatureInput().getBytes(StandardCharsets.UTF_8), jws.getSignature());
```

**KubernetesJwksEndpointLoader.java** (Lines 40-56):
```java
String token = getToken(issuer);
String wellKnownEndpoint = issuer + "/.well-known/openid-configuration";
// ...
if (token != null) {
    wellKnownReqest.auth(token);
}
String jwksUri = wellKnownReqest.asJson(OIDCConfigurationRepresentation.class).getJwksUri();
SimpleHttpRequest jwksRequest = simpleHttp.doGet(jwksUri).header(HttpHeaders.ACCEPT, "application/jwk-set+json");
if (token != null) {
    jwksRequest.auth(token);
}
```

### On success

Keycloak authenticates the internal client and issues the access token (for that internal client/service account user).

## Most Important Things Keycloak Validates

### Issuer trust/mapping

- JWT `iss` must match configured Kubernetes IDP issuer.
- Issuer must uniquely map to one enabled IDP (enforced at config time).

**IssuerValidation.java** (Lines 25-42):
```java
String issuer = getConfig().get(ISSUER);
if (Strings.isEmpty(issuer)) {
    throw new IllegalArgumentException("Issuer is required");
}
checkUrl(realm.getSslRequired(), issuer, "Issuer");
// ...
IdentityProviderModel existingIdp = lookupProvider.lookupIdentityProviderFromIssuer(session, type, getConfig().get(ISSUER));
if (existingIdp != null && (getInternalId() == null || !Objects.equals(existingIdp.getInternalId(), getInternalId()))) {
    throw new IllegalArgumentException("Issuer URL already used for IDP '" + existingIdp.getAlias() + "', Issuer must be unique ...");
}
```

### Subject binding to client

JWT `sub` must match `jwt.credential.sub` on exactly one Keycloak client for that IDP alias.

### Signature integrity

JWT signature must verify using issuer JWKS key (kid/alg path).

### Audience

`aud` must match realm issuer URL; multiple audiences are rejected for federated JWT.

### Time validity

- `exp` required
- token active (`nbf`/`exp`) with allowed skew
- `iat` not in future
- age bounded by max-exp (default 3600s for Kubernetes path)

### Client binding safety

if `client_id` form param is present, it must match resolved internal client.
