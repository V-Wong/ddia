# Formats for Encoding Data
## Overview
Programs usually work with data in 2 (or more) representations:
- **In-memory**: data kept in structures optimised for efficient access and manipulation by the CPU (typically with pointers).
- **Byte sequence**: data is encoded in a self-contained sequence of bytes to be sent over a network or persisted in a file.

The translation from in-memory to a byte sequence is known as **encoding** (aka serialisation or marshalling), and the reverse is known as **decoding** (aka parsing, deserialisation or unmarshalling). As this is a common problem, there are many libraries and encoding formats to choose from.

## Language-Specific Formats
Many programming languages come with built-in support for encoding in-memory objects into byte sequences:
- Java has `java.io.Serializable`.
- Ruby has `Marshal`.
- Python has `Pickle`.

These encoding libraries are convenient, but have many deep problems:
- The encoding is often **tied to a specific language** and reading the data in another language is very difficult.
- Allowing the decoder to **instantiate arbitrary classes** is a source of **security problems**.
- **Versioning** is often an afterthought in these libraries.
- **Efficiency** (in encoding/decoding and size of encoded structures) is often an afterthought.

## JSON, XML, and Binary Variants
JSON, XML and CSV are **standardised encodings** that can be read and written by **many programming languages**. They however all have subtle issues:
- **Ambiguity in encoding numbers**: XML and CSV does not distinguish between a number and a string of digits. JSON distinguishes strings and numbers but not integers and floats.
- **Support for binary strings**: JSON and XML don't support sequences of bytes without a character encoding. 
- **Limited use of schemas**: JSON and XML both have optional schema support, but this seldom used for JSON. CSV does not have any schema support, so it is up to the application to define the meaning of each row and colum.

Despite these flaws, JSON, XML, and CSV are good enough for many purposes. They will likely remain popular as a **data interchange format** as the difficulty of agreement on any other format outweighs most other concerns.

### Binary Encoding
For data that is used only internally, there is less pressure to use a lowest-common-denominator encoding format. Hence it may be worthwhile to choose a format that is **more compact** or **faster to parse**. Once the data reaches terabyte ranges, the choice of data format can have a huge impact.

There are **binary encoding variants** of JSON and XML, but they have seen limited use compared to the textual versions. Since these encodings don't prescribe a schema, they need to include all object field names within the encoded data. The result is that the binary encoding is only marginally smaller than the textual representation. It's not clear whether such a small space reduction is worth the loss of human readabiility.

## Thrift and Protocol Buffers
Apache Thrift and Protocol Buffers are binary encoded libraries that **require a schema** for any data that is encoded. 

These libraries both come with a **code generation tool** that produces classes from a given schema in various programming languages. Our application code can then call this generated code to encode or decode records of the schema.

The binary encoding for these libraries is much like that of JSON and XML. Each field has:
- **Type annotation**: indicates whether the field is a string, integer, list, etc.
- **Field tag**: indicates which field in the schema.
- **Length**: indicates the size of the field, if required.
- **Actual value**.

The main difference is that there are **no field names**. Instead, the encoded data contains **field tags**, which are numbers corresponding to the order the field appears in the schema definition. This fields tags are essentially **aliases** for fields.

### Field Tags and Schema Evolution
Field tags are critical to the meaning of encoded data. Names of fields can be changed in the schema, since the encoded data never refers to field names, but we can not change a field's tag, as that would make all existing encoded data invalid.

When adding a new field, the new field must be given a new tag number. This ensures:
- **Forwards compatibility**: old code will simply ignore the unknown tag number.
- **Backwards compatibility**: new code can read old data, because the tag numbers still have the same meaning.
    - We however can't make the field required.

Removing a field is just like adding a field, with backwards and forward compatibility concerns **reversed**.

### Datatypes and Schema Evolution
Changing the datatype of a field may be possible, but there is a risk that values will lose precision or get truncated. It is best to refer to documentation for details.

## Avro
Avro is another binary encoding format that is interestingly different from protobuff and Thrift. It uses two schemas to specify the structure of the data being encoded:
- **Avro IDL**: intended for human editing.
- **JSON**: for machine readability.

Avro does NOT use tag numbers in its schemas. However it has the **most compact encoding**. Its encoding does not contain any identity fields or data types. Instead the encoding is simply the **values concatenated together**. To parse this binary data, we go through the fields in the order that they appear in the schema and use the schema to tell us the datatype of each field. This means that the binary data can only be decoded correctly if the code reading the data is using the **exact same schema** as the code that wrote the data. Any mismatch in the schema between the reader and the writer would mean incorrectly decoded data.

This naturally makes schema evolution more complex.

### Writer Schema and Reader Schema
With Avro, an application encodes the data using whatever version of the schema it knows about - for example, the schema may be compiled into the application. This is known as the **writer's schema**.

When an application decodes some data, it is expecting the data to be in some schema, known as the **reader's schema**. This is the schema the application code is relying on - code may have been generated from that schema during the application's build process.

The key idea with Avro is that these two schemas **don't have to be the same** but has to be **compatible**. When data is decoded, Avro resolves the differences by translating from the writer's schema into the reader's schema.

### Schema Evolution Rules
To maintain compatibility, Avro only allows adding fields with a **default value**. This ensures:
- **Forwards compatibility**: new field exists in new writer schema but omitted old reader schema, old code simply ignores it.
- **Backwards compatibility**: new field omitted in old writer schema but exists in new reader schema, new code simply uses default value.

Similarly, a field can only be removed if it has a default value. 

### Writer Schema in Depth
There is complexity with how a reader can know the writer's schema with which a particular piece of data is encoded. This is achieved in various ways depending on context:
- **Large file with lots of records (all encoded with same schema)**: writer of the file can include the writer's schema once at the beginning of the file.
- **Database with individually written records**: every record includes a version number at the beginning of every encoded record. The database also keeps a list of schema versions. A reader can fetch a record, extract the version number, and then fetch the writer's schema for that version number from the database.
- **Sending records over network connection**: when two processes are communicating over a bidirectional network connection, they can negotiate the schema version on connection setup and use that schema for the lifetime of the connection.

### Code Generation and Dynamically Typed Languages
Code generation is useful in statically typed languages because it allows efficient in-memory structures to be used to decode data, and it allows type checking and autocompletion in IDEs. In dynamically typed programming languages, there is not much point in generating code. 

Avro provides optional code generation for statically typed languages, but it can be used without any code generation. If we have a object container file which embeds the writer's schema, we can simply open it using the Avro library much like a JSON file. The file is **self-describing** since it includes all necessary metadata.

## Merits of Schemas
Compared to textual data formats such as JSON, XML and CSV, binary encodings based on schemas have multiple nice properties:
- They are much more **compact** since they can omit field names from the encoded data.
- The schema is a form of **documentation**, and because it is required for decoding, we can ensure it is up to date.
- Keeping a database of schemas allows us to check **forward and backward compatibility** of schema changes, before anything is deployed.
- For statically typed languages, code generation from schemas allows **type checking at compile time**.

In summary, schema evolution allows the same kind of flexibility as schemaless databases, while also providing better guarantees about our data and better tooling.
