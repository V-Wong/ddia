# Graph-Like Data Models
## Overview
Recall that:
- **Document Model**: good for 1-m relationships or no relationships between records.
- **Relational model**: can handle simple m-m relationships between records.

As the connections within the data grow more complex, it becomes natural to model the data as a **graph**.

## Examples of Graph Data Models
Examples of data that can be modelled as graphs include:
- **Social Graphs**: vertices are people, and edges indicate which people know each other.
- **Web Graph**: vertices are web pages, and edges indicate which HTML links to other pages.
- **Road Networks**: vertices are junctions, and edges represent the roads between them.

In these examples, all vertices in a graph are of the **same kind**. However, graphs can also provide a consistent way of storing **different types of objects** in a single data store. For example, Facebook maintains a single graph with many different types of vertices and edges.

## Property Graphs
In the property graph model, each vertex consists of:
- A unique identifier.
- A set of outgoing edges.
- A set of incoming edges.
- A collection of properties (key-value pairs).

Each edges consists of:
- A unique identifier.
- The vertex at which the edge starts (tail vertex).
- The vertex at which the edge ends (head vertex).
- A label to describe the kind of relationship between the two vertices.
- A collection of properties (key-value pairs).

Some important aspects of this model are:
- Any vertex can have an edge connecting it with any other vertex.
- Given any vertex, we can efficiently find both its incoming and outgoing edges, and thus **traverse** the graph.
- By using different labels for different kinds of relationships, we can store several different kinds of information in a single graph.

Query languages for property graphs include **Cypher**.

## Triple-Stores
In the triple-store model, all information is stored in the form of **three-parth statements**: `(subject, predicate, object)`. This is conceptually mostly equivalent to the property graph model.

Query languages for triple stores include **SPARQL** and **Datalog**.
