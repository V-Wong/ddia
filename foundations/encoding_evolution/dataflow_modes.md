# Modes of Dataflow
## Dataflow Through Databases
In a database, the process that writes to the database encodes the data, and the process that reads from the database decodes it. This may be a single process, where the reader is simply a later version of the same process.

It's common for **several different processes** to be accessing a database at the same time. These processes could be different services, or simply several instances of the same service. Either way, it is likely that some processes will be running **newer code** while others will be running **older code**. Thus, both forwards and backwards compatibility is required.

Furthermore, consider the scenario:
1. New field is added to a table.
2. New code writes a value for that new field to the database.
3. Old code reads the record, updates it, and writes it back.

The desirable behaviour is usually for the old code to keep the new field intact, even though it couldn't be interpreted. The encoding formats discussed previously supports such preservation of unknown fields, but additional care needs to be taken at the application level. 

### Different Values Written at Different Times
When a new version of an application is deployed, it may entirely replace the old version within minutes. However, database contents will **remain in their original encoding indefinitely** unless we have explictly rewritten it since then. This observation is known as **data outlives code**.

Migrating data into a new schema is possible, but is expensive and seldomly done. Most relational database allow simple schema changes, such as adding columns with a `null` default value, without rewriting existing data. When an old row is read, the database fills in `null`s for any columns that are missing from the encoded data on disk. Some databases use Avro for storage, exploiting its schema evolution rules.

Schema evolution allows the entire database to appear as if it was **encoded with a single schema**, even if the underlying storage may contain records **encoded with various historical versions of the schema**.

### Archival Storage
A snapshot of the database may be periodically taken for backup or data warehousing purposes. In these cases, it is often beneficial to **encode the data dump consistently** using the **latest schema version**.

## Dataflow Through Services: REST and RPC
### Client Server Architecture
The most common arrangement for communication of processes over a network is to have two roles: **clients** and **servers**. The **servers expose an API** over the network, and the **clients connect to the servers** to **make requests to that API**. The API exposed by the server is known as a **service**.

Moreover, a server can itself by a **client to another service**. This is often used to **decompose a large application** into **smaller services by area of functionality**, such that one service makes a request to another when it requires some functionality or data from that other service. This is known as **microservice architecture**.

A key design goal of microservice architecture is to make the application **easier to change and maintain** by making services **independently deployable and evolvable**. For example, each service should be owned by one team, and that team should be able to release new versions of the service frequently, without having to coordinate with other teams.

### Web Services
When **HTTP** is used as the underlying protocol for talking to a service, it is called a **web service**. Within web services, there are two popular approaches: **REST** and **SOAP**.

#### REST
REST is a **design philosophy** that builds upon the principles of HTTP. It emphasises:
- Simple data formats.
- Using URLs for identifying resources.
- Using HTTP features for cache control, authentication, and content type negotiation.

An API designed according to the principles of REST is called **RESTful**. A definition format such as **OpenAPI** (aka **Swagger**), can **describe RESTful APIs** and **produce documentation**.

#### SOAP
SOAP is an **XML-based protocol** for making network API requests. It is most commonly used over HTTP, but aims to be **independent from HTTP** and avoids using most HTTP features.

Nowadays, SOAP is seldomly used, so notes are kept brief here.

### Remote Procedure Calls (RPCs)
The RPC model tries to make a request to a remote network **look the same as calling a local function**, within the same process. Although this looks convenient, is fundamentally flawed. A network request is very different from a local function call:
- A local function call is predictable and either succeeds or fails. A network request is **unpredictable**, and can fail for many reasons outside our control. So we have to anticipate for them, for example by retring a failed request.
- A local function either returns a result, throws an exception, or never returns (infinite loop or crash). A network request can also **timeout**, where if we don't get a response from the remote service, we have no way of knowing whether the request got through or not.
- If we retry a failed network request, it could happen that the requests are actually getting through, and only the responses are lost. This is problematic if we don't have **idempotency**. Local function calls don't have this problem.
- Network requests have much greater **variability in latency** compared to local function calls.
- We can pass references (pointers) to objects in local memory to a local function. For network requests, all parameters must be **encoded into a sequence of bytes** that can be sent over the network. This can be a costly process for large objects.
- The client and service may be **implemented in different langauges**, so the RPC framework must translate datatypes from one language into another. This is not a problem for a single process written in a single language.

#### Current Directions for RPC
Despite all these problems, RPC isn't going away. Various RPC frameworks have been built on top of the previously described encodings:
- Thrift and Avro come with RPC support included.
- gRPC is implemented using Protocol Buffers.
- Finagle is implemented using Thrift.
- Rest.li uses JSON over HTTP.

New RPC frameworks are **more explicit** about the differences between remote requests and a local function call. They introduce features such as:
- Using **futures** (promises) to encapsulate asynchronous actions that may fail.
- Providing **service discrovery** that allows a client to find out at which IP address and port number it can find a particular service.

Custom RPC protocols with a binary encoding format can achieve **better performance** than something generic like JSON over REST. However, a RESTful API has other significant advantages:
- Good for **experimentation** and **debugging** without any code generation.
- Well supported with a **vast ecosystem** of tools available.

For these reasons, REST remains dominant for public APIs. The main focus of RPC frameworks is on requests between services owned by the same organisation, typically within the same datacenter.

#### Data Encoding and Evolution for RPC
For evolvability, it is important that RPC clients and servers can be **changed and deployed independently**. A reasonable simplifying assumption is that **all servers will be updated before any clients**. This means we only need backwards compatibility on requests, and forwards compatibility on responses. These compatibility properties are inherited from whatever encoding the RPC scheme uses.

Service compatibility is made difficult when RPC is used for **communication across organisational boundaries**, as the provider has no control over its clients. Thus, compatibility needs to be maintained for a long time, perhaps indefinitely. If a compatibility-breaking change is required, the service provider often maintains multiple versions of the service API side by side.

## Message-Passing Dataflow
We've so far discussed:
- REST and RPC where one process sends a request over the network to another process and expects a response as quickly as possible.
- Databases where one process writes encoded data, and another process reads it again sometime in the future.

Asynchronous message passing lies between the two:
- Similar to RPC in that a client's request (called a message) is delivered to another process with **low latency**.
- Similar to databases in that the message is sent via an intermediary called a **message broker**, which **stores the message temporarily**.

A message broker has several advantages compared to direct RPC:
- It can act as a **buffer** if the recipient is unavailable or overloaded, and thus **improve system reliability**.
- It can **automatically redeliver messages** to a process that has crashed, and thus prevent messages from being lost.
- It avoids the sender **needing to know the IP address and port** of the recipient.
- It allows one message to be sent to **several recipients**.
- It logically **decouples** the sender from the recipient (the sender just publishes messages and doesn't care who consumes them).

A difference to RPC is that message-passing communication is **usually one-way**: a sender normally doesn't expect to receive a reply to its messages. It is possible for a process to send a response, but usually over a **separate channel**. This communication pattern is **asynchronous**: the sender doesn't wait for the messsage to be delivered, but simply sends and forgets.

### Message Brokers
The detailed delivery semantics vary by implementation and configuration, but in general, message brokers are used as follows:
1. One process sends a message to a named **queue** or **topic**.
2. Broker ensures that the message is delivered to one or more **consumers** or **subscribers** to that queue or topic.

A topic provides only **one-way dataflow**. However, a consumer may itself publish messages to another topic, or to a reply queue that is consumed by the sender of the original message (allowing a request/response dataflow similar to RPC).

Message brokers typically **don't enforce any particular data model**. If the encoding is backwards and forwards compatible, we have the greatest flexibility to change publishers and consumers independently and deploy them in any order.

If a consumer republishes messages to another topic, we need to be careful to preserve unknown fields.

### Distributed Actor Frameworks
The **actor model** is a programming model for concurrency in a single process. Rather than dealing directly with threads, logic is encapsulated as **actors**. Each actor typically represents one client or entity, may have some local state, and communicates with other actors by sending and receiving asynchronous messages. Message delivery is not guaranteed.

In **distributed actor frameworks**, this model is scaled across **multiple nodes**. The same message-passing mechanism is used, regardless of whether the sender and recipient are on the same or different nodes. If they are on different nodes, the message is transparently encoded into a byte sequence, sent over the network, and decoded on the other side.

Location transparency works better in the actor model than in RPC because:
- Messages are already assumed to be loseable, even within a single process.
- Latency has less of a mismatch between local and remote communications.

A dsitributed actor framework typically integrates a **message broker** and the **actor model** into a **single framework**. However, if we want rolling upgrades, we still have to worry about forwards and backwards compatibility, as messages may be sent from a node running the new version to a node running the old version, and vice versa.

Three popular distributed actor frameworks handle message encoding as follows:
- **Akka**: uses Java's built-in serialisation, which does not provide forwards or backwards compatibility.
- **Orleans**: uses custom data encoding format that does not support rolling upgrade deployments; deploying a new version requires setting up a new cluster, and shifting the traffic from the old clusters to the new ones.
- **Erlang OTP**: supports rolling upgrade but requires careful planning; it is difficult to make record schema changes.
