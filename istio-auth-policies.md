# Istio Authorization Policies

I recently had to debug Istio authorization policies and I learned what a pain it can be understand how and when policies are applied to protect service on your mesh. This post is my attempt to explain the import bits of how to set this up. With that said, checkout the [Istio official docs](https://istio.io/latest/docs/reference/config/security/authorization-policy/) , despite being dense at times, they are pretty good.



## The Tools

### Istio

Istio is one of the more popular service mesh implementations out there. It's used by lots of companies to manage the growing overhead of having services running on Kubernetes. This is especially true when dealing with microservice architectures, which yield dozens (if not more) services which need to talk to each other securely and reliably. 

Using a service mesh, like Istio, handles a lot of this complexity for you:

- mTLS between services (encrypted both ways)
- telemetry (common network metrics exposed for Prometheus)
- rate limiting & fault tolerance
- authentication (are you who you say you are?)
- authorization (can **you** do **that**?)

We're going to focus on the last two in this post. 



## The JWT

(*pronounced 'jot'*)

Authentication is proving who are you. Using Istio, you can request invokers of services on the mesh prove who they are using JWTs ([Javascript Web Token](https://jwt.io/introduction)). This functionality is configured with a [RequestAuthenication](https://istio.io/latest/docs/reference/config/security/request_authentication/) CRD. Once this functionality is configured you can use [AuthorizationPolicies](https://istio.io/latest/docs/reference/config/security/authorization-policy/) to enforce RBAC(role based access control) based on provided JWTs.

But to that, you need to understand what the JWT is, and how it's used in conjunction with Istio to enforce RBAC for services.

![image-20210709173717855](assets/image-20210709173717855.png)

*Above: Anatomy of a JWT*

The JWT is the standard token used in [OpenIDConnect](https://openid.net/connect/) (OIDC) spec, and widely used as result. It has three pieces to it, the header, **payload**, and signature. We are focused on the payload. Within the JSON payload one can put more or less anything (note: keys in the payload are called **claims**). That said, there are certain keys (claims) that are reserved by the JWT spec. Non-reserved claims are called custom claims, of which the OIDC spec [defines a handful of](https://auth0.com/docs/scopes/openid-connect-scopes). These claims are often grouped into scopes. Where a scopes can be thought of as roles with associated set of permissions. Again, unsurprisingly the OIDC specifies certain scopes.

### Now why does this matter?

Because, the JWT can act as proof of who you are, and claims within the JWT capture what you are allowed to do. The JWT is usually provided by the client in the `Authorization: Bearer <token>` header. There's a whole slew of tooling around doing this "handshake", Istio included. However, the neat thing about using a service mesh is that Istio can handle this interaction transparently to services. You only need configure the RequestAuthenication and AuthorizationPolicy objects.

![image-20210711110154319](assets/image-20210711110154319.png)

*Above: JWT 'handshake'*

## The RequestAuthentication Object

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: "jwt-example"
  namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  jwtRules:
  - issuer: "testing@secure.istio.io" # Whose JWTs do you trust?
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release- 1.10/security/tools/jwt/samples/jwks.json" # Where to get certs to verify JWT
```

In the `RequestAuthentication` object you can specify which workloads require a JWT from which trusted issuers for usage. Note you need to provide `jwksUri` so that Istio knows where to grab the certs used in the validation of the tokens.

## The AuthorizationPolicy Object

- [ ] Understand where principle comes from
- [ ] Create the visualization of many bouncers for thinking about auth policies
- [ ] Include link to auth policy be example doc
- [ ] Is there tooling to check?

Now here it gets a little tricky.

❓️ Where is the principle defined

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "reader"
  namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/inventory-sa"]
    to:
    - operation:
        methods: ["GET"]
```
