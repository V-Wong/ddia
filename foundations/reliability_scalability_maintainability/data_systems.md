# Data Systems
## Overview
A **data system** provides functionality for an application to:
- **Store** data to be used later (database).
- **Remember** the result of an expensive operation (caches).
- **Search** or **filter** data (search indexes).
- **Send** a message to another process (stream processing).
- **Periodically** crunch accumulated data (batch processing).

Modern applications typically combine many of these systems together to form a **composite data system**. This involves breaking work down into smaller tasks that can be performed efficiently on a single tool, and the tools are then stitched together using **application code**.

## Concerns in Data Systems
There are three main concerns in most software systems:
- **Reliability**: system should continue to work **correctly** in the face of adversity.
- **Scalability**: system should be handle growth in data volume, traffic volume and complexity.
- **Maintainability**: engineers should be productive in maintaining current behaviour and adapting to new use cases.
