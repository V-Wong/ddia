# Reliability
## Overview
A **reliable** system continues to work **correctly** when things go wrong. The things that can go wrong are called **faults**. A system that can anticipate and cope with faults is called **fault-tolerant** or **resilient**.

## Importance of Reliability
Bugs in applications can cause:
- Lost productivity.
- Lost revenue.
- Damage to reputation.

In certain situations, reliability may be sacrified to **reduce** development cost or operation cost. We should be very concious when cutting corners.

## Types of Faults
Formally, a fault is when a component **deviates from its spec**. This is in contrast to a **failure** which is when the system stops providing the required service. Faults often lead to failures.

### Hardware Faults
Hardware faults include:
- Hard disk crash.
- RAM failure.
- Power grid failure.
- Network failure.
- Etc.

These faults are often **random** and **independent** from each other.

#### Hardware Redundancy
The first response to hardware faults is to add **redundancy** to individual hardware components to reduce failure rate. This includes:
- Disks in RAID configuration. 
- Servers with dual power supplies.
- Hot-swappable CPUs.
- Backup power generators.
- Etc. 

When one component dies, the redundant component can take its place while the broken component is replaced. This approach is well understood and can often keep a machine running uninterrupted for years.

**Component-level redundancy** is often sufficient as it makes total failure of a single machine fairly rare. However, **highly available** systems go further with **machine-level redundancy**.

### Software Faults
Software faults are **systematic** failures in the system. These faults are hard to anticipate, and often correlated across nodes which leads to many more system failures. Examples include:
- Software bug that crashes when given bad input.
- Runaway process using up a shared resource.
- Service that system depends on slowing down.
- Cascading failures, where a small fault in one component triggers a fault in another component, which in turn triggers further faults.

### Human Errors
Humans are unreliable. Configuration errors by operators are a leading cause of outage. Systems should be reliable in spite of human error. Such approaches include:
- Design systems to minimise opportunities for error.
    - Design good abstractions, APIs and admin interfaces.
    - Encourage the "right action" and discourage "wrong actions".
- Decouple the places where people make mistake from the place they can cause failures.
    - Provide **sandbox** environments where people can explore and experiment safely.
- Test thoroughly at all levels, from unit tests to whole-system integration tests and manual tests.
- Allow quick and easy recovery from human errors.
    - Make it fast to roll back configuration changes.
    - Roll out new code gradually.
    - Provide tools to recompute data.
- Set up detailed and clear monitoring, such as performance metrics and error rates.

