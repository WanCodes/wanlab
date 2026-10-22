# wanlab

Lab repository for [Wan Yau](https://github.com/WanCodes) ([WanCodes](https://github.com/WanCodes) on GitHub), a London-based senior full-stack developer. The public GitHub bio is React, React Native, and TypeScript. LinkedIn: [wan-yau-83550538](https://www.linkedin.com/in/wan-yau-83550538).

This repository holds small backend experiments. The only project in the tree is a Node.js shop API added in October 2018. Other public work is indexed from [WanCodes/portfolio](https://github.com/WanCodes/portfolio).

## Contents

| Path | What it is |
| --- | --- |
| [`nodejs/restapi/node-rest-shop/`](nodejs/restapi/node-rest-shop) | Express and MongoDB API for products and orders (`rest-shop` 1.0.0) |

## node-rest-shop

A tutorial-style shop API. Clients create products with an image, list and update those products, and place orders that point at a product. `package.json` describes it as "A Node.js RESTful API Tutorial Project (Build a simple shop API)". The `author` field is `Wan`.

The server speaks JSON over HTTP, stores documents in MongoDB, and writes product images under `uploads/`. The API and its Postman collection were committed on 25 October 2018. The previous README edit, on 7 November 2018, changed only the repository one-liner.

### Stack

Versions below are the ones locked in `package-lock.json` (lockfileVersion 1).

| Piece | Role | Version |
| --- | --- | --- |
| Node.js | Runtime. The source is CommonJS (`require`). `package.json` has no `engines` field. | — |
| Express | HTTP application | 4.16.4 |
| Mongoose | Models and queries | 5.3.5 |
| MongoDB | Database named in the connection string (`test`), via MongoDB Atlas | — |
| body-parser | JSON and URL-encoded bodies | 1.18.3 |
| Morgan | Request logging in `dev` format | 1.9.1 |
| Multer | Multipart product images, stored on disk | 1.4.1 |
| nodemon | Development process used by `npm start` | 1.18.4 |

`package.json` sets `main` to `index.js`. The file that listens is `server.js`, which loads `app.js`.

### Layout

```
nodejs/restapi/node-rest-shop/
├── server.js                 HTTP server (PORT, or 3000)
├── app.js                    Express app, MongoDB, CORS, routes
├── nodemon.json              Sets MONGO_ATLAS_PW for npm start
├── api/models/products.js    Product schema
├── api/models/orders.js      Order schema
├── api/routes/products.js    /products
├── api/routes/orders.js      /orders
├── uploads/                  Image files, served at /uploads
└── node-rest-shop.postman_collection.json
```

### Run

Install and start from the package directory. Multer writes to `./uploads/`, and Express serves that same relative path, so the process working directory has to be `nodejs/restapi/node-rest-shop`.

```bash
cd nodejs/restapi/node-rest-shop
npm install
npm start
```

`npm start` runs `nodemon server.js`. The process listens on `process.env.PORT`, and on port **3000** when `PORT` is unset:

`http://localhost:3000`

MongoDB has to be reachable. On load, `app.js` calls `mongoose.connect` with `useNewUrlParser: true` and no error handler. The URI points at a MongoDB Atlas replica set from 2018:

- Hosts: `node-rest-shop-shard-00-00-ueprp.mongodb.net`, `...-01-...`, `...-02-...` on port `27017`
- Replica set: `node-rest-shop-shard-0`
- Database: `test`
- User: `main-user`
- Options: `ssl=true`, `authSource=admin`, `retryWrites=true`

The password is interpolated from `process.env.MONGO_ATLAS_PW`. `nodemon.json` sets that variable for `npm start`, and the value is stored in that file in plain text. To use another database, replace the URI in `app.js` and supply your own password. Keep new passwords out of git.

```bash
PORT=3000 MONGO_ATLAS_PW=your-password npm start
```

`server.js` calls `listen` without waiting on Mongoose, so the port can open while Atlas is still connecting or unreachable. Routes that query the database then fail.

`npm test` prints `Error: no test specified` and exits 1. The repository has no test files.

### HTTP API

Responses are JSON. Morgan writes a `dev` log line for each request.

A path with no route returns **404**:

```json
{ "error": { "message": "Not found" } }
```

The error middleware uses `error.status` when it is set, and **500** otherwise, with the same `error.message` shape.

CORS sets `Access-Control-Allow-Origin` to `*`. Allowed request headers are `Content-Type`, `Accept`, and `Authorization`. An `OPTIONS` request returns **200** with body `{}` and allows `PUT`, `POST`, `PATCH`, `DELETE`, and `GET`. No route reads an `Authorization` header.

Many success bodies include a `request` object (`type` and `url`) pointing at a follow-up call. Those URLs are hardcoded to `http://localhost:3000`. The product update and product delete bodies write `http:localhost:3000` (the slashes after the scheme are missing in `api/routes/products.js`).

#### Products

| Method | Path | Request | Success |
| --- | --- | --- | --- |
| `GET` | `/products` | — | **200** `{ count, products }` |
| `POST` | `/products` | Multipart fields `name`, `price`, and file field `productImage` | **201** `{ message, createProduct }` |
| `GET` | `/products/:productId` | — | **200** `{ product, request }` |
| `PATCH` | `/products/:productId` | JSON array of `{ propName, value }` | **200** `{ message, request }` |
| `DELETE` | `/products/:productId` | — | **200** `{ message, request }` |

`GET /products` selects `name`, `price`, `_id`, and `productImage`. Each item includes a `request` link to `GET /products/:id`.

`POST /products` stores one file with Multer. Accepted types are `image/jpeg` and `image/png`. Anything else is filtered out. The size limit is 5 MB (`1024 * 1024 * 5`). The stored filename is `Date.now()` concatenated with the original filename, under `uploads/`. The handler reads `req.file.path` immediately, so a request without an accepted file fails before a document is saved. The **201** body (`createProduct`) returns `name`, `price`, `_id`, and a `request` link. It does not echo `productImage`.

Example patch body, as exported in the Postman collection:

```json
[{ "propName": "price", "value": "11.99" }]
```

The route copies each `propName` / `value` pair into a Mongoose `$set` update. A cast error or a driver error returns **500** `{ error }`. A valid id that matches no document still follows the update result path and returns **200**.

`GET /products/:productId` returns **404** `{ message: "No valid entry found for product" }` when the id is a valid ObjectId and no document exists. An id that cannot be cast returns **500** `{ error }`.

#### Orders

| Method | Path | Request | Success |
| --- | --- | --- | --- |
| `GET` | `/orders` | — | **200** `{ count, orders }` |
| `POST` | `/orders` | JSON `{ productId, quantity }` | **201** `{ message: "Order stored", createdOrder, request }` |
| `GET` | `/orders/:orderId` | — | **200** `{ order, request }` |
| `DELETE` | `/orders/:orderId` | — | **200** `{ message: "Order deleted", request }` |

`POST /orders` loads `productId` first. A missing product returns **404** `{ message: "Product not found" }`. On success the saved order stores `product` as an ObjectId. `quantity` defaults to `1` in the schema when the field is omitted. `createdOrder` returns `_id`, `product`, and `quantity`.

Example body (id taken from the Postman collection; it is not seeded in this repo):

```json
{ "productId": "5bd0a322a711420c1c26718f", "quantity": 2 }
```

The collection sends `quantity` as a string. The schema type is `Number`, and Mongoose casts it.

If `Product.findById` rejects, or if `order.save` fails, the handler responds **500** with `message: "Product not found"` and an `error` field. That same message is used for a missing product (**404**, no `error` field) and for a failed save.

`GET /orders` selects `product`, `quantity`, and `_id`, and populates `product` with `name`. `GET /orders/:orderId` populates `product` with `_id`, `name`, and `price`. A valid id with no document returns **404** `{ message: "Order not found" }`. An id that cannot be cast returns **500** `{ error }`.

`DELETE` calls `Order.remove` and returns **200** when that call resolves, including when no document matched.

#### Uploaded files

`GET /uploads/<filename>` serves the `uploads/` directory.

Two JPEGs are already in the tree:

- `uploads/1540421272687interactive_map.jpg`
- `uploads/1540421849614interactive_map.jpg`

They are leftover upload files. The API does not seed product documents for them.

### Models

Product (`Product`):

| Field | Schema |
| --- | --- |
| `_id` | `ObjectId`, assigned in the route with `mongoose.Types.ObjectId()` |
| `name` | String, `required: true` |
| `price` | Number, `required: true` |
| `productImage` | String. The option in source is `require: true` |

Order (`Order`):

| Field | Schema |
| --- | --- |
| `_id` | `ObjectId`, assigned in the route |
| `product` | `ObjectId` ref `Product`. The option in source is `require: true` |
| `quantity` | Number, default `1` |

Mongoose enforces presence with `required`. The `require` keys on `productImage` and `product` are a different option, so the schema does not enforce those two fields.

### Postman

Import [`nodejs/restapi/node-rest-shop/node-rest-shop.postman_collection.json`](nodejs/restapi/node-rest-shop/node-rest-shop.postman_collection.json).

The collection includes:

- Products: list, JSON create, multipart create, get, patch, delete
- Orders: list, create, delete

It has no request for `GET /orders/:orderId`.

Sample ids in the export (`5bcf46a8e03d580d94ca3af8`, `5bd0a322a711420c1c26718f`, `5bd0e41729ed233c20519ac8`) come from a 2018 session. Replace them with ids returned by the database you are connected to.

The multipart request is named "POST Product file". Its file part has an empty key. Set that key to `productImage` before sending. The separate "POST Products" request sends JSON and no file, which does not satisfy the route.

### Notes

- Product update uses `Product.update`, and both deletes use `Model.remove`. Those are Mongoose 5 methods.
- `.gitignore` lists `node_modules/` only. `nodemon.json`, the lockfile, and `uploads/` are tracked.

### License

`package.json` sets `"license": "ISC"`. This repository has no `LICENSE` file.
