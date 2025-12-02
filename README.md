# node-red-contrib-mongodb4

A MongoDB driver node for Node-RED that exposes the native MongoDB Node.js driver features without limitations.

[![npm version](https://img.shields.io/npm/v/node-red-contrib-mongodb4.svg?style=flat-square)](https://www.npmjs.org/package/node-red-contrib-mongodb4) [![install size](https://img.shields.io/badge/dynamic/json?url=https://packagephobia.com/v2/api.json?p=node-red-contrib-mongodb4&query=$.install.pretty&label=install%20size&style=flat-square)](https://packagephobia.now.sh/result?p=node-red-contrib-mongodb4) [![npm downloads](https://img.shields.io/npm/dm/node-red-contrib-mongodb4.svg?style=flat-square)](https://npm-stat.com/charts.html?package=node-red-contrib-mongodb4)

This package provides two Node-RED nodes:

- **Config Node** — manage MongoDB connections and connection pools.
- **Flow Node** — perform database and collection operations using the native MongoDB driver API.

Table of Contents
- Installation
- Quick Start
- Compatibility
- Configuration Node
- Flow Node
- BSON Types
- Development
- Links

## Installation

From your `~/.node-red` directory run:

```bash
npm install --save --omit=dev node-red-contrib-mongodb4
```

## Quick Start

1. Add a **MongoDB Config Node** and configure the connection (URI or host/port, database, auth).
2. Add a **MongoDB Flow Node** and select the config node, operation and collection.
3. Pass operation arguments into `msg.payload` as an array — the array items are passed as function arguments to the driver.

Examples

Insert one document:

```js
msg.payload = [{ name: 'Anna', age: 1 }];
return msg;
```

Find with query and options:

```js
const query = { age: 22 };
const options = { sort: { name: 1 }, projection: { name: 1 }, limit: 10, skip: 2 };
msg.payload = [query, options];
return msg;
```

Aggregate (pipeline + options):

```js
const pipeline = [ { $sort: { age: 1 } }, { $project: { name: 1 } }, { $limit: 10 } ];
const options = { allowDiskUse: true };
msg.payload = [pipeline, options];
return msg;
```

> Note: for aggregation you often end up with an array inside the payload array: `msg.payload = [pipeline]`.

## Compatibility

- Node-RED >= v3.0.0
- Node.js >= v16.20.1
- Compatible with MongoDB server versions: 4.0, 4.2, 4.4, 5.0, 6.0, 7.0, 8.0 (depending on driver)

Version 3.x of this package uses the MongoDB driver v6.12 and contains breaking changes from earlier driver major versions — see the official MongoDB driver upgrade notes: https://www.mongodb.com/docs/drivers/node/current/upgrade/#version-6.0-breaking-changes

## Configuration Node

The config node creates a `MongoClient` and manages a connection pool. You can configure connection details in two ways:

- Simple fields: protocol (`mongodb` / `mongodb+srv`), hostname, port, database, username/password, auth source/mechanism.
- Full URI: provide a complete connection string.

Connection-related options available in the node include `connectTimeoutMS`, `socketTimeoutMS`, `minPoolSize`, `maxPoolSize`, `maxIdleTimeMS` and other `MongoClientOptions` (JSON).

TLS options are supported: CA file, client certificate/key file, and an option to disable certificate verification (not recommended).

The config node will automatically generate an application name if you don't provide one (e.g. `nodered-xxxxx`). The application name is logged and can be used to inspect connections on the server.

Example server-side check:

```js
db.currentOp(true).inprog.reduce((acc, conn) => {
    const appName = conn.appName || 'unknown';
    acc[appName] = (acc[appName] || 0) + 1;
    acc.totalCount = (acc.totalCount || 0) + 1;
    return acc;
}, { totalCount: 0 });
```

## Flow Node

Use the flow node to execute collection or database operations.

Key options
- **Mode / msg.mode**: `collection` or `db` (selects whether you call collection APIs or database APIs)
- **Collection / msg.collection**: target collection name
- **Operation / msg.operation**: driver operation name (e.g. `find`, `findOne`, `insertOne`, `updateMany`, `aggregate`, `command`, `stats`)
- **msg.payload**: an array of arguments passed to the driver method
- **Output**: for `find` and `aggregate`, choose `toArray` or `forEach`
- **maxTimeMS**: server-side operation timeout

Common collection operations: `find`, `findOne`, `insertOne`, `insertMany`, `updateOne`, `updateMany`, `deleteOne`, `deleteMany`, `aggregate`.

Deprecated: the legacy `insert`, `update`, `delete` helper methods are not supported by modern drivers — use the newer specific methods instead.

The node returns the driver's response in `msg.payload`.

## BSON Types

If you need BSON types (e.g. `ObjectId`, `Double`, `Timestamp`) from the driver inside function nodes, add the driver to `functionGlobalContext` in your Node-RED `settings.js`:

```js
functionGlobalContext: {
    mongodb: require('node-red-contrib-mongodb4/node_modules/mongodb')
}
```

Then in a function node:

```js
const { ObjectId } = global.get('mongodb');
msg.payload = [{ _id: new ObjectId('624b527d08e23628e99eb963') }];
return msg;
```

The flow node can optionally convert string `_id` fields that look like valid `ObjectId`s to real `ObjectId` instances (deprecated feature — prefer explicit BSON types).

## Development

Start a local development environment (Node-RED + MongoDB) using the included `docker-compose.yml`:

```bash
docker compose up -d
```

Stop and remove containers:

```bash
docker compose down
```

Node-RED will be available at `http://localhost:1880` by default. Replace environment variables in `docker-compose.yml` with secure values for CI or production.

## Links

- Example flow: https://raw.githubusercontent.com/csteinba/node-red-contrib-mongodb4/master/examples/example-1.json
- Collection API docs (driver): https://mongodb.github.io/node-mongodb-native/6.21/classes/Collection.html
- MongoDB Node.js driver docs: https://www.mongodb.com/docs/drivers/node/current/

[Collection-API v6.21](https://mongodb.github.io/node-mongodb-native/6.21/classes/Collection.html)

### Payload Output

The node will output the database driver response as message payload.
The operations `aggregate` and `find` can output with `toArray` or `forEach`.

### How to use BSON data types with this Node

You can use BSON types with this node.

First enable "mongodb" in your function global context. Add this to your `settings.js` file - typically this file located in `~/.node-red`:
```js
functionGlobalContext: {
    mongodb: require("node-red-contrib-mongodb4/node_modules/mongodb")
},
```
This kind of require statement ensures that we use the BSON types from the mongodb driver used in this node. Otherwise we could run into compatibilty issues.

You can now use BSON types in your function node like so:
```js
// get BSON types
const {ObjectId, Double, Timestamp} = global.get("mongodb");
// write your query
msg.payload = [{
    _id: new ObjectId() , 
    value: new Double(1.4), 
    ts: new Timestamp()
}];
// send them to the mongodb node
return msg;
```

### Node Status 

Node status information is displayed below the node:

#### Tags
- **s** : Number of successful executions
- **err** : Number of failed executions 
- **rt** : Last execution runtime in ms 

#### Colors
- **green** : Last execution was successful 
- **blue** : Node execution in progress 
- **red** : Last execution failed

### More general driver information

[Visit the MongoDB Driver Docs](https://www.mongodb.com/docs/drivers/node/current/)

## Development: Run with Docker Compose

You can start a local development environment with Node-RED and MongoDB using the included `docker-compose.yml` file. This starts two containers on a shared network and uses named Docker volumes for persistence.

Start the stack:
```bash
docker compose up -d
```

Stop and remove containers:
```bash
docker compose down
```

Environment variables are set inside `docker-compose.yml` with example values. For production or CI, replace them with secure values or set them in a `.env` file.

Node-RED will be available on `http://localhost:1880`.

