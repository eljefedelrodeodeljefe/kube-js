# kube-js [![codecov](https://codecov.io/gh/eljefedelrodeodeljefe/kube-js/branch/master/graph/badge.svg)](https://codecov.io/gh/eljefedelrodeodeljefe/kube-js) [![Build Status](https://travis-ci.org/eljefedelrodeodeljefe/kube-js.svg?branch=master)](https://travis-ci.org/eljefedelrodeodeljefe/kube-js)

Simplified [Kubernetes API](http://kubernetes.io/) client for Node.js.

## Installation

Install via npm:

```console
npm i kube-js --save
```

## Examples

`kube-js` provides access to all the Kubernetes objects and
includes some niceties for writing simpler code.

### Basics

`kube-js` maps the URI paths in the Kubernetes API to
sequences of objects chained together via properties and ending in a
function. For example, to GET the `ReplicationController` named
'http-rc' in the `Namespace` 'my-project':

```js
const Api = require('kube-js')
const core = new Api.Core({
  url: 'http://my-k8s-api-server.com',
  version: 'v1',  // Defaults to 'v1'
  namespace: 'my-project' // Defaults to 'default'
})

function print(err, result) {
  console.log(JSON.stringify(err || result, null, 2))
}

core.namespaces.replicationcontrollers('http-rc').get(print)
```

`kube-js `supports the Extensions API group. For example, GET
the `Deployment` named `http-deployment`:

```js
const ext = new Api.Extensions({
  url: 'http://my-k8s-api-server.com',
  version: 'v1beta1',  // Defaults to 'v1beta1'
  namespace: 'my-project' // Defaults to 'default'
})

ext.namespaces.deployments('http-deployment').get(print)
```

`kube-js` provides a helper to get in-cluster config and accessing the API from a Pod:

```js
const Api = require('kube-js')
const core = new Api.Core(Api.config.getInCluster())
```

and a helper to get the current-context config from `~/.kube/config`:

```js
const Api = require('kube-js')
const core = new Api.Core(Api.config.fromKubeconfig())
```

### **Experimental** support for promises and async/await

`kube-js` exposes **experimental** support for promises via
the `promises` option passed to API group constructors. The API is the
same, except for the functions that previously took a callback
(*e.g.*, `.get`). Those functions now return a promise.

```js
// Notice the promises: true
const core = new Api.Core({
  url: 'http://my-k8s-api-server.com',
  version: 'v1',  // Defaults to 'v1'
  promises: true,  // Enable promises
  namespace: 'my-project' // Defaults to 'default'
})
```

and then:

```js
core.namespaces.replicationcontrollers('http-rc').get()
  .then(result => print(null, result))
```

or with `async/await`:

```js
print(null, await core.namespaces.replicationcontrollers('http-rc').get())
```

You can invoke promise-based and callback-based functions explictly:

```js
print(null, await core.namespaces.replicationcontrollers('http-rc').getPromise())
core.namespaces.replicationcontrollers('http-rc').getCb(print)
```

### Creating and updating

kube-js objects expose `.post`, `.patch`, and `.put`
methods. Create the ReplicationController from the example above:

```js
const manifestObject = require('./rc.json')
core.namespaces.replicationcontrollers.post({ body: manifestObject }, print)
```

or update the number of replicas:

```js
const patch = { spec: { replicas: 10 } };
core.namespaces.replicationcontrollers('http-rc').patch({
  body: patch
}, print)
```

### Using the correct API group and version

`kube-js` client includes functionality to help determine the
correct Kubernetes API group and version to use based on manifests:

```js
const Api = require('kube-js')
const api = new Api.Api({
  url: 'http://my-k8s-api-server.com',
  namespace: 'my-project'
})

const manifest0 = {
  kind: 'Deployment',
  apiVersion: 'extensions/v1beta1'
  ...
}
const manifest1 = {
  kind: 'ReplicationController',
  apiVersion: 'v1'
  ...
}

api.group(manifest0).ns.kind(manifest0).post({ body: manifest0 }, print)
api.group(manifest1).ns.kind(manifest1).post({ body: manifest1 }, print)
```

### Object name aliases

`kube-js` supports the same aliases as
[`kubectl`](http://kubernetes.io/docs/user-guide/kubectl-overview/)
(*e.g.*, `ns` for `namespaces`) and the singular versions of the
resource name (*e.g.*, `namespace` for `namespaces`). We can shorten
the example above:

```js
core.ns.rc('http-rc').get(print)
```

### Switching namespaces

You can call the `namespace` object to specify the namespace:

```js
core.ns('other-project').rc('http-rc').get(print)
```

### Query parameters

You can optionally specify query string object `qs` to GET
endpoints. kube-js passes `qs` directly to
[`request`](https://www.npmjs.com/package/request#requestoptions-callback).
For example to filter based on [label
selector](http://kubernetes.io/docs/user-guide/labels/):

```js
core.ns.rc.get({ qs: { labelSelector: 'service=http,component=api' } }, print)
```

### Label selector filtering

kube-js has a shortcut, `matchLabels`, for filtering on label
selector equality:

```js
core.ns.rc.matchLabels({ service: 'http' }).get(print)
```

and a more general `match` method based on Kubernetes Match Expressions:

```js
core.ns.rc.match([{
  key: 'service',
  operator: 'In',
  values: ['http']
}, {
  key: 'deploy',
  operator: 'NotIn',
  values: ['production', 'staging']
}]).get(print)
```

### ThirdPartyResources

You can extend the Kubernetes API using a
[ThirdPartyResource](https://kubernetes.io/docs/user-guide/thirdpartyresources/)
and kube-js:

```js
const newResoure = {
  apiVersion: 'extensions/v1beta1',
  kind: 'ThirdPartyResource',
  metadata: {
    name: 'new-resource.kube-js.io'
  },
  description: 'Example resource',
  versions: [{
    name: 'v1'
  }]
}

ext.thirdpartyresources.post({ body: newResource }, print)
```

and then extend an `ThirdPartyResource` API client with your new resources:

```js
const thirdPartyResources = new Api.ThirdPartyResources({
  url: 'http://my-k8s-api-server.com',
  group: 'kube-js.io',
  resources: ['customresources']  // Notice pluralization!
})

// Access `customresources` as if they were a regular Kubernetes object
thirdPartyResources.ns.customresources.get(print)
thirdPartyResources.addResource('newresources')  // Notice pluralization!
// Now access `newresources`
thirdPartyResources.ns.newresources.get(print)
```

### ReplicationController Pods

`kube-js` provides a shortcut for listing all Pods matching a
ReplicationController selector:

```js
core.ns.rc.po.get(print)
```

`kube-js` deletes all the Pods associated with a
ReplicationController when it deletes the ReplicationController. You
can preserve the Pods:

```js
core.ns.rc.delete({ name: 'http-rc', preservePods: true }, print)
```

### Watching and streaming

You can call `.getStream` to stream results. This is useful for watching:

```js
const JSONStream = require('json-stream')
const jsonStream = new JSONStream()

const stream = core.ns.po.getStream({ qs: { watch: true } })
stream.pipe(jsonStream)
jsonStream.on('data', object => {
  console.log('Pod:', JSON.stringify(object, null, 2))
})
```

You can access logs in a similar fashion:
```js
const stream = core.ns.po('http-123').log.getStream({ qs: { follow: true } })
stream.on('data', chunk => {
  process.stdout.write(chunk.toString())
})
```

**Note:** the kube-apiserver will close watch connections eventually
according to the
[`--min-request-timeout`](http://kubernetes.io/docs/admin/kube-apiserver/
command line argument. `kube-js` does not attempt to reconnect
when the kube-apiserver closes a connection.

### Authenticating

`kube-js` supports Kubernetes [apiserver
authentication](http://kubernetes.io/docs/admin/authentication/).

Basic authentication (with optional certificate authority):

```js
const core = new Api.Core({
  url: 'https://my-k8s-api-server.com',
  ca: fs.readFileSync('cluster-ca.pem'),
  auth: {
    user: 'user',
    pass: 'pass'
  }
})
```

or without a certificate authority:

```js
const core = new Api.Core({
  url: 'https://my-k8s-api-server.com',
  insecureSkipTlsVerify: true,
  auth: {
    user: 'user',
    pass: 'pass'
  }
})
```

token authentication:

```js
const core = new Api.Core({
  url: 'https://my-k8s-api-server.com',
  auth: {
    bearer: 'token'
  }
})
```

and client certificate authentication:

```js
const core = new Api.Core({
  url: 'https://my-k8s-api-server.com',
  ca: fs.readFileSync('cluster-ca.pem'),
  cert: fs.readFileSync('my-user-cert.pem'),
  key: fs.readFileSync('my-user-key.pem')
})
```

### Passing options to `request`

kube-js uses
[`request`](https://github.com/request/request). You can specify
[`request` options](https://github.com/request/request#requestoptions-callback)
for kube-js to pass to `request`:

```js
const core = new Api.Core({
  url: 'https://my-k8s-api-server.com',
  request: {
    timeout: 3000
  }
})
```

## Testing

`kube-js` includes unit tests and integration tests.
[Minikube](https://github.com/kubernetes/minikube) is a tool that
makes it easy to run integration tests locally.

Run the unit tests:

```console
npm test
```

The integration tests use a running Kubernetes server. You specify the
Kubernetes server context via the `CONTEXT` environment variable. For
example, run the integration tests with the `minikube` context:

```console
CONTEXT=minikube npm run test-integration
```

## More Documentation

* [Kubernetes Reference Documentation](http://kubernetes.io/docs/reference/)
* [Kubernetes Swagger Spec](http://kubernetes.io/kubernetes/third_party/swagger-ui/)

## License

[MIT](LICENSE)
