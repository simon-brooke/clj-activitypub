# clj-activitypub

> **Warning**
> This is very much a work-in-progress. Only a tiny bit of the ActivityPub spec is implemented, and it definitely does not conform to all of the nuances expected _yet_.

`clj-activitypub` is a set of utilities that can be combined together to create a fully-functional ActivityPub server.

## Libraries
- `activitypub-core` — The base functionality for generating HTTP headers (i.e. `Signature`, `Digest`), building ActivityPub activities and objects, and sending requests to remote servers.
- `activitypub-ring` — A Ring-specific implementation that builds on `activitypub-core`, providing default routes and handlers for making an ActivityPub-compliant server.

## Examples

```clj
;; ActivityPub object handling
(require '[clj-activitypub.core :as activitypub])
;; Translate account handles into ActivityPub URL-based IDs
(require '[clj-activitypub.webfinger :as webfinger])
;; Build and perform network requests
(require '[clj-activitypub.net :as activitypub-net])
```

### Reading remote data

```clj
;; Fetching user account on remote server

(require '[clojure.pprint :refer [pprint]])

;;; Use any ActivityPub account handle you like
(def account-handle "@jahfer@mastodon.social")

;;; Retrieve the account details from its home server
(def account
 (-> account-handle
     (webfinger/parse-handle)
     (webfinger/fetch-user-id!)
     (activitypub-net/fetch-user!)
     (select-keys [:name :preferredUsername :summary])))

;;; Examine what you got back!
(pprint account) ;; => ({:name "Jahfer",
                 ;;      :preferredUsername "jahfer",
                 ;;      :summary "<p>Hello world!</p>"})
```

```clj
;; Fetching a list of users from multiple servers
(-> "https://mastodon.social/users/jahfer/followers"
    (activitypub-net/fetch-users!)
    (count)) ;; => 80
```

### Pushing data to remote servers

Before POSTing data to a remote server, you'll want to create a local key/value pair in the `/keys` directory. You'll also need a corresponding HTTP endpoint to serve the actor data, which is used to verify the signature of the message by the remote server.

```bash
$ openssl genrsa -out keys/private.pem 2048
$ openssl rsa -in keys/private.pem -outform PEM -pubout -out keys/public.pem
```

```clj
;; Submitting a Create activity for an Object to remote server
(require '[clj-http.client :as client])

(def config
  (activitypub/config {:domain "ottawa.place"
                       :username "jahfer"}))

(def my-note
  (activitypub/obj config {:id 1
                           :type :note
                           :content "Hello world!"})

(let [body (activitypub/activity config :create my-note)
      request {:headers {"Host" "mastodon.social"}
               :body (json/write-str body)}]
  ;;; POST our note to the shared inbox for mastodon.social
  (client/post "https://mastodon.social/inbox"
               (assoc request
                      :headers (activitypub-net/auth-headers config request)
                      :throw-exceptions false)))
```

## Running tests

There are two libraries within this package: `activitypub-core` and `activitypub-ring`. In order to run tests, the following command must be run from inside either of those directories:

```bash
# clj_activitypub/activitypub_core
$ clj -X:test
```

## References
- https://www.w3.org/TR/activitypub/
- https://www.w3.org/TR/activitystreams-core/
- https://www.w3.org/ns/activitystreams#Public
- https://www.w3.org/TR/activitystreams-vocabulary/
