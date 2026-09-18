# CST8915 Lab 1: Algonquin Pet Store on Azure VM

- **Student Name**: Hye Ran Yoo
- **Student ID**: 041145212
- **Course**: CST8915 Full-stack Cloud-native Development
- **Semester**: Fall 2026

---

## Demo Video

🎥 [Watch Demo Video](https://www.youtube.com/watch?v=YOUR_VIDEO_ID)

---

## Technical Explanations

### Order Service (Node.js)

The Order Service takes the orders from the Store Front and puts them into RabbitMQ. It is written in Node.js 24 with **Express** for the web part and the **amqplib** library to talk to RabbitMQ. It has only one endpoint, `POST /orders`, and it runs on port 3000. Node.js is a good choice here because this service does no heavy calculation. It just receives a small JSON message and waits for the network.

In the architecture, this service is the **producer**. When an order arrives, it connects to `amqp://localhost`, creates a **durable** queue called `order_queue`, and sends the order as a **persistent** message. It answers `Order received` only after RabbitMQ confirms that the message is saved. It talks to the Store Front over HTTP and to RabbitMQ over AMQP, and it never talks to the Product Service. There is no consumer in this lab, so the orders stay in the queue.

### Product Service (Rust)

The Product Service gives the product list. It is written in **Rust** with the **Warp** framework on the **Tokio** runtime. It has one route, `GET /products`, which returns three products (Dog Food $19.99, Cat Food $34.99, Bird Seeds $10.99). The list is written directly in the code, so there is no database. It listens on `0.0.0.0:3030`, which means it answers on every network interface of the VM. Rust is compiled and fast, which fits a small service that only reads data.

This service is **independent**. It does not need RabbitMQ and does not know about the Order Service, so it can run alone. Only the Store Front calls it, with a `GET` request from the browser. The code adds a CORS filter that allows any origin but only the `GET` method, so the page on port 8080 is allowed to read the product list.

### Store Front (Vue.js)

The Store Front is the page that the customer sees. It is a **Vue.js 3** application, started with `npm run serve` on port 8080. The `OrderForm.vue` component loads the products when the page opens, shows them as radio buttons, lets the user type a quantity, and calculates the total price (price × quantity, for example 2 × Dog Food = $39.98).

The Store Front is the **client of both backend services**. It uses `fetch()` to call `GET :3030/products` for the product list and `POST :3000/orders` to send the order. The important point is that `fetch()` runs in the browser on my laptop, not on the VM. So `localhost` would mean my own laptop. That is why I changed both URLs in `OrderForm.vue` to the VM public IP, and why ports 3000 and 3030 must be open in the Network Security Group. The Store Front never talks to RabbitMQ directly. The Order Service hides RabbitMQ behind its REST endpoint.

---

## Challenges and Learnings

- This was my first time creating a **virtual machine in Azure**. I could see how the basic pieces work together: the resource group, the VM, the public IP, and the **Network Security Group** that opens the ports.
- `Standard_B2s` was shown as **"Size not available"** for my Azure for Students subscription, so I used `Standard_B2ls_v2` (2 vCPU, 4 GiB) in Canada Central instead.
- I connected to the VM with **VS Code Remote-SSH** and ran each service in its own terminal. Keeping three terminals open made it clear that **each service is a separate program**: `cargo run` for the products, `node index.js` for the orders, and `npm run serve` for the web page. If I close one terminal, only that part of the application stops.
- I also learned what **RabbitMQ** stores. In the management UI I could see that `order_queue` is **durable** and the messages are **persistent**, so they are written to disk and survive a restart. The message count only goes up, because this lab has no consumer that reads the orders.
- The error `Failed to fetch products` happens when `OrderForm.vue` still points to `localhost`. The browser runs on my laptop, so the URLs must use the VM public IP, and the NSG must allow ports 3000 and 3030.
- Building one small application out of **four different technologies** showed me the point of microservices: each service can use the language that fits its job, and they only need to agree on the messages they exchange.

---

## Acknowledgments

- Lab instructions and application source: [ramymohamed10/26F_Lab1_CST8915](https://github.com/ramymohamed10/26F_Lab1_CST8915)
