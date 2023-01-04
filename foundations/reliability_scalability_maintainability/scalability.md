# Scalability
## Overview
Scalability is a system's ability to **cope with increased load**.

## Describing Load
**Load parameters** are numbers that describe the load of the system. Examples include:
- Requests per second to a web server.
- Ratio of reads to writes in a database.
- Number of simultaneously active users in a chat room.
- Hit rate on cache.
- Followers per user in a Twitter-like application.

## Describing Performance
There are two ways to investigate what happens to a system when the load increases:
- When you increase a load parameter and keep the system resources **unchanged**, how is performance affected?
- When you increase a load parameter, how much do you need to **increase** the resources to keep performance unchanged.

### Response Times
**Response time** is the most important performance number for online systems. It measures the time between a client sending a **request** and **receiving** a response. This differs from **latency** which is the duration that aa request is waiting to be handled; it does not include the handling time itself.

#### Distributions
Response times can **vary** a lot due to:
- An expensive request with more data.
- Context switch to a background process.
- Network packet loss with TCP retransmission.
- Garbage collection pause.
- Page fault forcing a read from disk.
- Etc.

We therefore think of response times in terms of **distributions**. From the distribution we can form numerical summaries such as:
- **Average**: arithmetic mean of all values. Not typically a good metric for typical user experience since it doesn't tell how many users actually experienced that delay.
- **Percentiles**: value at the pth percentile indicates p% of users experience a delay lower than that amount.
    - High percentiles of response times are known as **tail latencies**. Users with these response times is rare, but they are often valuable to the business due to having lots of data or otherwise. Optimising for these percentiles is often very difficult however. 

#### Measuring Response Times
**Queueing delays** often account for a large part of the response time at high percentiles. A small number of slow requests can hold up the processing of subsequent requests - an effect called **head-of-line blocking**. Even if subsequent requests are fast to process, the client will see a slow overall response time due to the time waiting for prior requests to finish. It is hence important to measure response times **on the client side**.

## Coping with Load
### Horizontal vs Vertical Scaling
Systems can be scaled in two ways:
- **Horizontal scaling**: distributing load across multiple machines. 
- **Vertical scaling**: upgrading existing machines.

Single machine systems are often simpler, but high-end machines can become very expensive, so very intensive workloads can't avoid horizontal scaling. Good architectures often use a pragmatic mixture of both approaches.

### Elastic Systems
A system is **elastic** if it can automatically add computing resources when a load increase is detected. This is useful when load is highly unpredictable, but involves more complexity over manually scaled systems.

### Stateless vs Stateful Systems
**Stateless** services are easy to distribute across multiple machines. This is much more difficult for **stateful** systems. Conventional wisdom is to keep the database on a single node until scaling cost or availability requirements make scaling out a neccessity.

### Application-Specific Scaling
Different applications are built around different **assumptions** which may be suited to different architectures. Such assumptions include:
- Ratio of reads to writes.
- Volume of data to store.
- Complexity of data.
- Response time requirements.
- Access patterns.
- Etc.

An architecture that works well for one application might not work well for another application. Despite this specificity, scalable architectures are built from general-purpose building blocks, arranged in familiar patterns.
