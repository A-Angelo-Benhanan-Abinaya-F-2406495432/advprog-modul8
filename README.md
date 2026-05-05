# gRPC Tutorial

### 1. What are the key differences between unary, server streaming, and bi-directional streaming RPC methods, and in what scenarios would each be most suitable?

**Unary** is the simplest. The client sends one request and gets one response back. Like our `process_payment`, it's good for simple operations like submitting a form or making a payment.
**Server streaming** is when the client sends one request but the server sends back multiple responses over time. We used this in `get_transaction_history` where the server streams 30 transactions one by one. It's useful when the server has a lot of data to send back.
**Bidirectional streaming** lets both the client and server send messages to each other at the same time. We used this in the Chat Service where the client types a message and the server replies, all through the same open connection. It's good for real-time apps like chat.


### 2. What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption?

Right now our server runs on plain HTTP with no security at all, which is fine for learning but not for real use.
For a real app, we'd want **TLS** to encrypt the connection so nobody can read the data being sent. We'd also want authentication, like checking a token in the request header to verify who is calling the service. On top of that, authorization would check whether that user is actually allowed to do what they're asking. For example, a user should only be able to see their own transactions.


### 3. What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications?

One challenge is when the server is sending messages faster than the client can receive them, or the other way around. We handled this a little bit by setting a buffer size on our channel, but if the buffer fills up things can get slow or blocked. Another issue is error handling inside `tokio::spawn`. If something goes wrong in that separate task, it doesn't automatically crash the whole program. It just silently stops. So we have to be careful to handle errors properly. Also, when the client disconnects, the server task doesn't immediately know. It only finds out when it tries to send a message and gets an error back.


### 4. What are the advantages and disadvantages of using `tokio_stream::wrappers::ReceiverStream` for streaming responses in Rust gRPC services?

The main advantage is that it makes it really easy to turn a Tokio channel into a stream that Tonic can use. We just create a channel, spawn a task that sends data into it, then wrap the receiver in `ReceiverStream` and return it. It's simple and works well for most cases. The downside is that if the client disconnects, the task that's producing data keeps running until it tries to send something and notices the channel is closed. There's no built-in way to stop the producer early, which could waste resources.


### 5. In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity, promoting maintainability and extensibility over time?

Right now all the service implementations are in one big file (`grpc_server.rs`), which gets messy as the project grows. A better approach would be to put each service in its own file, like `src/services/payment.rs`, `src/services/transaction.rs`, and `src/services/chat.rs`. Shared things like error handling helpers or authentication checks could go in a separate `common` module so we don't repeat the same code in every service. This way each part of the code has one clear job and is easier to update or test independently.


### 6. In the `MyPaymentService` implementation, what additional steps might be necessary to handle more complex payment processing logic?

Our current implementation just returns `success: true` no matter what, which is obviously not how a real payment works.
A real payment service would need to validate the input (like checking the amount isn't negative), connect to a database to record the transaction, and probably call an external payment provider API. It should also handle failures gracefully and return useful error messages instead of just true or false.


### 7. What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms?

Using gRPC means both the server and client have to agree on the `.proto` file, which defines exactly what messages and methods exist. This is good because it makes the API clear and consistent, and if someone changes it you'll get a compile error right away.
The downside is that not everything supports gRPC easily. Browsers, for example, can't use gRPC directly. REST APIs with JSON are much easier to call from anywhere. So gRPC is great for communication between backend services, but less convenient for public-facing APIs.


### 8. What are the advantages and disadvantages of using HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs?

HTTP/2 lets multiple requests happen over the same connection at the same time, which is more efficient than HTTP/1.1 where each request kind of waits in line. This is what makes gRPC streaming possible and fast. The downside is that HTTP/2 is harder to debug. Raw traffic can't be read directly in your terminal like with plain HTTP/1.1. Also, not all tools and proxies support it perfectly. WebSockets are similar to bidirectional streaming but they don't come with structure or code generation like gRPC does. You'd have to define your own message format, which is more work.


### 9. How does the request-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness?

With REST, the client always has to ask first and wait for a reply. If you want real-time updates, you'd have to keep asking over and over (polling), which is wasteful. With gRPC bidirectional streaming like our Chat Service, the connection stays open and both sides can send messages whenever they want. This is much better for real-time apps because there's no delay from repeatedly opening new connections, and the server can push data to the client without being asked.


### 10. What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads?

With Protocol Buffers, you define exactly what your data looks like in the `.proto` file and the code gets generated from that. This means if you send the wrong type or forget a field, you'll catch it at compile time. It also makes the binary data smaller and faster to send compared to JSON text. With JSON, you can send pretty much anything and the structure is flexible. This is nice when you're prototyping or building a public API, since anyone can read and write JSON easily. But it also means mistakes are only caught at runtime, which can be harder to debug. For internal services that need to be fast and reliable, Protocol Buffers are the better choice.
