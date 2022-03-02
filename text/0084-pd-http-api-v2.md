# PD HTTP API V2

## Motivation

The V2 API aims to provide a standard RESTful API for PD. The current HTTP API doesn't follow the RESTful API conventions. We have summarized some problems of the current status:

- mixed-use of `_` and `-` in the path
- some methods are not accurate, e.g., we do not distinguish between PUT and POST
- the resource should use a noun    
- query parameter should not occur in the path
- mixed-use of singular and plural nouns

Besides the above problems, we are going to refine some internal structures and reduce some unnecessary APIs which are never been used before.

## Detailed Design

We can use gin as our HTTP web framework in the V2 API, which is more popular and has a better ecosystem.

Here is a basic example for stores:

```go
router := gin.New()
router.Use(middlewares.Redirector())
root := router.Group(apiV2Prefix)
meta := root.Group("meta")
meta.Use(middlewares.BootstrapChecker())
meta.GET("/stores", handlers.GetStores())
meta.GET("/stores/:id", handlers.GetStoreByID())
meta.DELETE("/stores/:id", handlers.DeleteStoreByID())
meta.PATCH("/stores/:id", handlers.UpdateStoreByID())
```

### Middleware

We can easily add the middleware through the `Use` function and the middleware we defined only needs to return [gin.HandlerFunc](https://github.com/gin-gonic/gin/blob/v1.7.7/gin.go#L34). This also can be applied in a specified API group. Gin has provided many [ready-made middleware](https://github.com/gin-contrib) which is convenient.

### Path definition

#### Resource

Since the key abstraction of information in RESTful API is a resource, it can be any non-virtual object. We use plural form to represent a set of objects, e.g., `/pd/api/v2/stores` and use the plural form with a unique field to represent a singular object, e.g., `/pd/api/v2/stores/:id`. There should not be a verb or any query parameter existing in the path. The previous path like `/pd/api/v1/store/{id}/state` should be rewritten to `/pd/api/v2/stores/:id` with `state` data in JSON format as an input.

#### Method

Here is a simple guide about how to use choose the correct method:

- GET: retrieve resources only
- POST: create resources or do a custom action. We should avoid using it on a single resource for a creating purpose.
- PUT: replace resources or collections
- PATCH: make a partial update on a resource
- DELETE: delete the resources

#### Word delimiter

The hyphen will be recommended to use as the word delimiter.

#### Group

For those middlewares that only need to be applied in a set of APIs, we can use API group. API group also can be used to divide our APIs into different sets according to their purpose and share the same middleware. The previous `/pd/api/v1/admin/*` can be put into `admin` group. For stores itself, we can put them into `meta` group, so the previous `/pd/api/v1/stores` or `/pd/api/v1/store` become to `/pd/api/v2/meta/stores`. Currently, the V1 API can be divided into the following groups:

- member: something about etcd itself
- meta: those resources related to PD meta info, the entity in PD core package, like regions, stores, cluster
- scheduling: use to control the scheduling behaviors, e.g., schedulers, checkers, operators
- admin: control the system behavior, e.g., log, ping
- debug: debug information through go pprof
- extension: some extra features required by other components

There are some other APIs that don't belong to any group mentioned above. We can decide to use an individual group according to middleware usage.

#### Custom action

We have many custom actions in API V1, such as `/pd/api/v1/regions/split` or `/pd/api/v1/regions/scatter`, etc. In V2, We recommend using `/pd/api/v2/regions/:action` with `POST` method. There may be [segment conflicts with existing wildcard](https://github.com/gin-gonic/gin/issues/1301) when using gin as the web framework. But fortunately, we don't have this conflict problem after we change existed V1 API to V2.
