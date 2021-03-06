# Fahrscheine, bitte!

[![Build Status](https://travis-ci.org/dryewo/fahrscheine-bitte.svg?branch=master)](https://travis-ci.org/dryewo/fahrscheine-bitte)
[![codecov](https://codecov.io/gh/dryewo/fahrscheine-bitte/branch/master/graph/badge.svg)](https://codecov.io/gh/dryewo/fahrscheine-bitte)
[![Clojars Project](https://img.shields.io/clojars/v/fahrscheine-bitte.svg)](https://clojars.org/fahrscheine-bitte)

A Clojure library for verifying [OAuth2 access tokens].
For using in [Compojure] routes or in [Swagger1st] security handlers.

> "Die Fahrscheine, bitte!" is what *Fahrkartenkontrolleur* says when entering a bus. *Schwarzfahrer* are fearing this.

Access tokens are verified against [Introspection Endpoint]. In the examples in this document the address of
the endpoint is configured through `TOKENINFO_URL` environment variable.

Responses of this endpoint are cached for 2 minutes. Creation of the cached token resolver function has to be done by the user 
to enable testability and follow best practices: make state explicit. Please see examples below.

## Usage

Examples assume the following:

```clj
(require '[fahrscheine-bitte.core :as oauth2]
         '[mount.core :as m])

(defn log-access-denied-reason [reason]
  (log/info "Access denied: %s" reason))
```

1. Example with [mount] and [Swagger1st]:

```clj
(m/defstate oauth2-s1st-security-handler
  :start (if-let [tokeninfo-url (System/getenv "TOKENINFO_URL")]
           (let [access-token-resolver-fn (oauth2/make-cached-access-token-resolver tokeninfo-url {})]
             (log/info "Checking OAuth2 access tokens against %s." tokeninfo-url)
             (oauth2/make-oauth2-s1st-security-handler access-token-resolver-fn oauth2/check-corresponding-attributes))
           (do
             (log/warn "No TOKENINFO_URL set; NOT ENFORCING SECURITY!")
             (fn [request definition requirements]
               request))))

(m/defstate handler
  :start (-> (s1st/context :yaml-cp "api.yaml")
             (s1st/discoverer)
             (s1st/mapper)
             (s1st/ring oauth2/wrap-reason-logger log-access-denied-reason)
             (s1st/protector {"oauth2" oauth2-s1st-security-handler})
             (s1st/parser)
             (s1st/executor)))
```

In this example we create a security handler that is given to `s1st/protector` to verify tokens on all endpoints that have
`oauth2` security definition in place.
Additionally, we insert a middleware `oauth2/wrap-reason-logger` that will log all rejected access attempts.

2. Example with [mount] and [Compojure]:

```clj
(m/defstate wrap-oauth2-token-verifier
  :start (if-let [tokeninfo-url (System/getenv "TOKENINFO_URL")]
           (let [access-token-resolver-fn (oauth2/make-cached-access-token-resolver tokeninfo-url {})]
             (log/info "Checking OAuth2 access tokens against %s." tokeninfo-url)
             (oauth2/make-wrap-oauth2-token-verifier access-token-resolver-fn ["/health"]))
             ; second argument maintains a whitelist of URI to bypass authentication ("/health" is unprotected in this scenario)
           (do
             (log/warn "No TOKENINFO_URL set; NOT ENFORCING SECURITY!")
             identity)))

(defn make-handler2 []
  (-> (routes
        (GET "/hello" req {:status 200}))
      (wrap-oauth2-token-verifier)
      (oauth2/wrap-log-auth-error log-access-denied-reason)))
```

`(oauth2/make-wrap-oauth2-token-verifier access-token-resolver-fn)` returns a Ring middleware that can be used to
check access tokens against given token introspection endpoint.

## License

Copyright © 2017 Dmitrii Balakhonskii

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.

[mount]: https://github.com/tolitius/mount
[swagger1st]: https://github.com/zalando-stups/swagger1st
[Compojure]: https://github.com/weavejester/compojure
[Introspection Endpoint]: https://tools.ietf.org/html/rfc7662#section-2
[OAuth2 access tokens]: https://tools.ietf.org/html/rfc6749#section-1.4
