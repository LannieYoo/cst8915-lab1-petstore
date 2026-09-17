# CST8915 Lab 1: Algonquin Pet Store on Azure VM

**Student Name**: Hye Ran Yoo
**Student ID**: 041145212
**Course**: CST8915 Full-stack Cloud-native Development
**Semester**: Fall 2026

---

## Demo Video

🎥 [Watch Demo Video](https://www.youtube.com/watch?v=YOUR_VIDEO_ID)

---

## Technical Explanations

### Order Service (Node.js)

The Order Service is the backend API that accepts customer orders from the Store Front and hands them off to RabbitMQ. It is written in Node.js 24 using **Express** for the HTTP layer, **amqplib** (callback API) as the AMQP client, and the **cors** middleware so that the browser-based Store Front, served from a different origin (port 8080), is allowed to call it. It exposes a single endpoint, `POST /orders`, and listens on port 3000 on all interfaces. Node.js is a good fit here because the service is I/O-bound: it does almost no computation, it just receives a small JSON payload and waits on the network, which the Node event loop handles efficiently.

In the microservices architecture this service plays the role of a **producer**. For every request it opens a connection to `amqp://localhost` (RabbitMQ on the same VM), creates a *confirm channel*, asserts a **durable** queue named `order_queue`, and publishes the request body as a **persistent** message with the `mandatory` flag. It only replies `Order received` (HTTP 200) after the broker confirms the message was stored; on any error or an unroutable message it replies HTTP 500, and in both cases it closes the connection. Because the Order Service never talks to the Product Service directly and never processes the order itself, order intake is decoupled from fulfilment: any future consumer service can read `order_queue` at its own pace. In this lab there is no consumer, so orders simply accumulate in the queue, which can be verified with `rabbitmqctl list_queues name durable messages`.

### Product Service (Rust)

The Product Service is responsible for serving the product catalog. It is written in **Rust** using the **Warp** web framework on top of the **Tokio** asynchronous runtime, with `serde_json` to build JSON responses. It exposes one route, `GET /products`, which returns a hard-coded array of three products with `id`, `name` and `price` (Dog Food 19.99, Cat Food 34.99, Bird Seeds 10.99). The server binds to `0.0.0.0:3030`, so it is reachable both from inside the VM (`localhost:3030`) and, once port 3030 is opened in the NSG, from the public IP. Rust was chosen because it is compiled, memory-safe without a garbage collector and very fast, which makes it well suited to a small, stateless, high-throughput read API.

Architecturally this is a fully **independent** service: it has no database, no dependency on RabbitMQ and no knowledge of the Order Service, so it can be started, stopped or scaled on its own. Its only inter-service communication is inbound: the Store Front running in the customer's browser fetches `/products` over HTTP. To make that cross-origin call possible, the route is wrapped in a CORS filter that allows any origin but only the `GET` method, which is exactly what a read-only catalog needs.

### Store Front (Vue.js)

The Store Front is the customer-facing web interface. It is a **Vue.js 3** single-page application built with **Vue CLI** (`@vue/cli-service`), and in this lab it is run with `npm run serve`, a webpack development server on port 8080. The main component, `OrderForm.vue`, loads the product list when the component is created, renders the products as radio buttons, lets the user enter a quantity, computes the total price with a Vue `computed` property (`price × quantity`, e.g. 2 × Dog Food = $39.98), and submits the order with a **Place Order** button. Vue was chosen because its reactive data binding keeps the UI in sync with the state (selected product, quantity, total) with very little code.

In the architecture the Store Front is the only piece the customer sees, and it is the **client of both backend services**. It communicates with them using the browser's `fetch()` API: `GET http://<host>:3030/products` to the Product Service to fill the catalog, and `POST http://<host>:3000/orders` with a JSON body (`product`, `quantity`, `totalPrice`) to the Order Service. An important consequence is that these requests are executed **by the browser on the user's laptop, not on the VM**, so the URLs cannot stay as `localhost`: `localhost` would point at the laptop itself. Before running on Azure, both URLs in `OrderForm.vue` must be changed to the VM's public IP, and ports 3000 and 3030 must be open in the Network Security Group. The Store Front never talks to RabbitMQ; the Order Service hides the message broker behind its REST endpoint.

---

## Challenges and Learnings (Optional)

- **VM size restriction**: `Standard_B2s` was shown as "Size not available" for the Azure for Students subscription regardless of region, so I used `Standard_B2ls_v2` (2 vCPU, 4 GiB) in Canada Central, which meets the lab requirement.
- **`Failed to fetch products`**: the Store Front's `fetch()` calls run in the laptop browser, so `localhost` must be replaced with the VM public IP and the NSG must allow ports 3000 and 3030.

---

## Acknowledgments

- Lab instructions and application source: [ramymohamed10/26F_Lab1_CST8915](https://github.com/ramymohamed10/26F_Lab1_CST8915)
