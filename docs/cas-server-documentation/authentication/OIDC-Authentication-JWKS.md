---
layout: default
title: CAS - OpenID Connect Authentication
category: Protocols
---
{% include variables.html %}

# JWKS - OpenID Connect Authentication

The JWKS (JSON Web Key Set) endpoint and functionality returns a JWKS containing public keys that enable 
clients to validate a JSON Web Token (JWT) issued by CAS as an OpenID Connect Provider.

{% include_cached casproperties.html properties="cas.authn.oidc.jwks" excludes=".rest,.groovy" %}

## Keystores
       
CAS expects a single global keystore to load and use for signing and encryption operations of various tokens. 
In addition to the global keystore, each [registered application in CAS](OIDC-Authentication-Clients.html) 
can *optionally* contain its own keystore as a `jwks` resource.

The following strategies can be used to generate a keystore.

### Default

By default, a global keystore can be expected and defined via CAS properties. The format 
of the keystore file is similar to the following:

```json
{
  "keys": [
    {
      "d": "...",
      "e": "AQAB",
      "n": "...",
      "kty": "RSA",
      "kid": "cas"
    }
  ]
}
```

CAS will attempt to auto-generate a keystore if it can't find one at the location specified in settings. If 
you wish to generate one manually, a JWKS file can be generated using [this tool](https://mkjwk.org/)
or [this tool](http://connect2id.com/products/nimbus-jose-jwt/generator).

<div class="alert alert-info"><strong>Clustered Deployments</strong><p>
When deploying CAS in a cluster, you must make sure all CAS server nodes have access to 
and share an <strong>identical and exact copy</strong> of the keystore file. Keystore differences
will lead to various validation failures and application integration issues.
</p></div>

<div class="alert alert-info"><strong>Key Rotation</strong><p>
Allowing CAS to rotate keys in the keystore is only applicable and relevant with this option,
where CAS is in charge of the keystore location and generation directly. If you outsource the
keystore generation task, you are also taking on the responsibility of rotation.
</p></div>

This keystore is cached, and is then automatically watched and monitored by CAS for changes. As changes are detected, CAS
will invalidate the cache and will reload the keystore once again.

### REST

Keystore generation can be outsourced to an external REST API. Endpoints must be designed to
accept/process `application/json` and generally should return a `2xx` response status code. 

The following requests are made by CAS to the endpoint:

| Operation        | Parameters      | Description      | Result
|------------------|-----------------|------------------|----------------------------------------------------
| `GET`            | N/A             | Retrieve the keystore.  | `2xx` status code; JWKS resource in response body.

{% include_cached casproperties.html properties="cas.authn.oidc.jwks.rest" %}
  
### Groovy

Keystore generation can be outsourced to an external Groovy script whose body should be defined as such: 

```groovy
import org.apereo.cas.oidc.jwks.*
import org.jose4j.jwk.*

def run(Object[] args) {
    def logger = args[0]
    logger.info("Generating JWKS for CAS...")
    def jsonWebKeySet = "{ \"keys\": [...] }"
    return jsonWebKeySet
}
```

{% include_cached casproperties.html properties="cas.authn.oidc.jwks.groovy" %}

### Custom

It is possible to design and inject your own keystore generation strategy into CAS using the following `@Bean`
that would be registered in a `@Configuration` class:

```java
@Bean(initMethod = "generate")
public OidcJsonWebKeystoreGeneratorService oidcJsonWebKeystoreGeneratorService() {
    return new MyJsonWebKeystoreGeneratorService(...);
}
```

Your configuration class needs to be registered 
with CAS. [See this guide](../configuration/Configuration-Management-Extensions.html) for better details.

## Key Rotation

Key rotation is when a key is retired and replaced by generating a 
new cryptographic key. Rotating keys on a regular basis is an industry 
standard and follows cryptographic best practices.

You can manually rotate keys periodically to change the JSON web key (JWK) key, or you can configure the appropriate schedule
in CAS configuration so it would automatically rotate keys for you. 

<div class="alert alert-info"><strong>Rotation Guidance</strong><p>
NIST guidelines seem to recommend a rotation schedule of at least once every two years. 
In practice, modest CAS deployments in size and scale tend to rotate keys once every six months, either 
manually or automatically on a schedule.
</p></div>

CAS always signs with only one signing key at a time, typically the *very first key* listed and loaded from the keystore.
The dynamic discovery endpoint will always include both the current key and the next key, and it may also 
include the previous key if the previous key has not yet been revoked. To provide a seamless experience in 
case of an emergency, client applications should be able to use any of the keys specified in the discovery document. 
