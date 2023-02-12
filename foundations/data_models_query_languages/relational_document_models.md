# Relation Model Versus Document Model
## Relational Model (SQL)
SQL organises data into **relations** (tables) which are unordered collections of **tuples** (rows). SQL has dominated for 30+ years and is the standard choice for **storing** and **querying** data with a **regular structure**.

Relational databases were originally for **business data processing** which involves:
- **Transaction processing**: entering sales, banking transactions, airline reservations, etc.
- **Batch processing**: customer invoicing, payroll, reporting, etc.

Nowadays, relational databases are used much more generally. Much of the **web** today is powered by relational databases.

## NoSQL
NoSQL is a class of **non-relational databases** which does not refer to any particular technology. In modern applications, the need for NoSQL was driven by:
- A need for **greater scalability** including very large datasets or high write throughput.
    - Typically difficult for relational databases to scale.
- A widespread preference for **free** and **open source** software.
- **Specialised query operations** that are not well supported by the relational model.
- A desire for **more dynamic** and **expressive** data models.
    - Relational schemas are typically restrictive.

## Object-Relational Mismatch
Most applications are written in an **OOP language**. This creates an **awkward translation layer** between objects in the application code and the relational database model of tables, rows and columns. 

ORM frameworks **reduce boilerplate** for this translation layer, but it can't completely hide the differences between the models.

## Relational vs Document Databases
### Modelling Relationships
Normalised relational databases often **refer to rows in other tables by ID**, because joins are easy.

Document databases **store data directly in the base table** as a hierarchical tree structure. As a result, document databases often have **poor support for joins**. 

### Schema Flexibility
Relational databases enforce schemas on **write**.
- Incorrectly formatted tuples are rejected on insertion.
- Application code does not do any checks as it can ensure all data it reads from the database is valid.
- Analagous to **static typing** in programming.

Document databases delays schema checking to a **read**.
- No checks on the structure of data inserted into a table.
- Application code is responsible for checking the validity when reading from the database.
- Analagous to **dynamic typing** in programming.

This difference is most apparent in **schema changes**:
- Relational database would require a **migration** on the table.
- Document database would require a **change to application code** to handle both the new and old schema.

### Data Locality
Relational databases often split data across multiple tables.
- Slower to retrieve **all associated data** for one entity, because more disk access required.
- Faster to retrieve or modify **individual parts** for one entity, because it can be access indepedently of the other data for the same entity.

Document databases often store whole documents in a single contiguous string.
- Fast to retrieve **all associated data** for one entity, because less disk access required.
- Slower to retrieve or modify **individual parts** for one entity, because it requires updating the entire document.

### Convergence of Document and Relational Databases
Relational databases have started introducing JSON/XML features. Document databases have started introducing joins. Modern databases have started taking a **hybrid approach** between the two models.
