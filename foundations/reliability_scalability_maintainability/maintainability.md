# Maintainability
## Overview
Ongoing **maintenance** involves:
- Fixing bugs.
- Keeping systems operational.
- Investigating failures.
- Adapting to new platforms.
- Modifying for new use cases.
- Repaying technical debt.
- Adding new features.
- Etc.

## Design Principles
### Operability
Good systems make it easy for operations teams to **keep the system running**.

#### Roles of Operations Teams
Operations teams are responsible for:
- Monitoring **system health** and quickly restoring failing services.
- Tracking down **causes of problems**, such as system failures or degraded performance.
- Keeping software and platforms **up to date**.
- Keeping tabs on how different systems **affect each other**.
- Anticipating and solving **future problems**.
- Establishing **good practices** and tools for deployment and configuration management.
- Performing **complex maintenance tasks**.
  - E.g. moving application from one platform to another.
- Defining **processes** that makes operations **predictable** and keep the production environment **stable**.
- Preserving the **organisation's knowledge** about the system.

#### Best Practices
Data systems can do various things to make routine tasks easy, including:
- Providing **good monitoring** into the runtime behaviour and internals of the system.
- Providing good support for **automation** and **integration** with standard tools.
- Avoiding dependency on **individual machines**.
- Providing **good documentation**.
- Providing **good default behaviour**, but also giving administrators the freedom to override defaults.
- **Self-healing** where appropriate, but also giving adminstrators ability for manual control.
- Exhibiting **predictable behaviour**.

### Simplicity
Good systems make it easy for new engineers to **understand the system**, by **minimising complexity**. 

The primary goal is to remove **accidental complexity** which is **not inherent to the problem domain**, but arises **from the implementation**. The main tool to solve accidental complexity is **abstraction** which hides implementation details behind a simple-to-understand **facade**. A good abstraction can be **reused across applications**, which leads to higher-quality software.

### Evolvability
Good systems make it easy for engineers to **make changes** to the system in the future, adapating it for unanticipated use cases as **requirements change**.

The evolvability of a system is heavily linked to its **simplicty** and its **abstractions**. In addition, **agile working patterns** provide a framework for adapting to change. This includes technical tools and and techniques such as TDD and refactoring which help developing software in frequently changing environments.
