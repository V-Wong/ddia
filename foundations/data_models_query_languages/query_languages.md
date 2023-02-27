# Query Languages for Data
## Declarative vs Imperative Query Languages
**Declarative languages**:
- Programmer specifies the **pattern of data** they want.
    - Includes: conditions and how the data is to be trasnformed.
- Programmer does NOT specify how to achieve the goal.
    - Database **query optimiser** decides the exact steps to achieve the goal.
    - Database can introduce **performance improvements** without any changes to queries.
- Are attractive because it is typically more **concise** and **easier** to work with.
- SQL is the most common example of a declarative language.

**Imperative languages**:
- Programmer specifies the **operations** to obtain the data they want.
- Most programming languages are imperative.

## Declarative Queries on the Web
Query languages are also used on web frontends to **target specific elements**. This includes languages such as:
- **CSS selectors**: declaratively state the pattern of elements to target.
- **Javascript**: imperatively use the DOM to search for the elements to target.

In this example, declarative languages are significantly easier to use and more robust compared to imperative languages.

## MapReduce Querying
**MapReduce** is a programming model for processing **large amounts of data in bulk across many machines**. A MapReduce query is based on two functions:
- ``map``: apply transformation to each element while **emitting a key**.
- ``reduce``: aggregate result together by **grouping on keys**.

MapReduce is a **hybrid** between an imperative and declarative query API. The two functions it provides act a as a declarative entry point for the processing framework. The code inside these functions can however be imperative.
