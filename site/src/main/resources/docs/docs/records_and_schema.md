---
layout: docs
title: Records, types and schema
permalink: docs/records_and_schema/
---
# Records

A data entry in Parquet is called a `record`. The record can represent a `row` of data, or it can be a nested complex field in another `row`. Other types of record are a `map` and a `list`. Stored data must be organised in `row`s. Neither primitive type nor `map` or `list` are allowed as a root data type.
In Parquet4s those concepts are represented by types that extend `ParquetRecord`: `RowParquetRecord`, `MapParquetRecord` and `ListParquetRecord`. `ParquetRecord` extends Scala's immutable `Iterable` and allows iteration (next to many other operations) over its content: fields of a `row`, key-value entries of a `map` and elements of a `list`. When using the library you have the option to use those data structures directly, or you can use regular Scala classes that are encoded/decoded by instances of `ValueCodec` to/from `ParqutRecord`.

Parquet organizes row records into `pages` and pages into `row groups`. Row groups and pages are data blocks accompanied by metadata such as `statistics` and `dictionaries`. Metadata is leveraged during [filtering]({% link docs/filtering.md %}) - so that some data blocks are skipped during reading if the metadata proves that the related data does not match provided filter predicate.

For more information about data structures in Parquet please refer to the [official documentation](https://parquet.apache.org/documentation/latest/).

# Schema

Each Parquet file contains the schema of the data it stores. The schema defines the structure of records, names and types of fields, optionality, etc. Schema is required for writing Parquet files and can be optionally used during reading for [projection]({% link docs/projection.md %}).

The official Parquet library, that Parquet4s is based on, defines the schema in Java type called `MessageType`. As it is quite tedious to define the schema and map it to your data types, Parquet4s comes with a handy mechanism that derives it automatically from Scala case classes. Please follow this documentation to learn which Scala types are supported out of the box and how to define custom encoders and decoders.

If you do not wish to map the schema of your data to Scala case classes, then Parquet4s allows you to stick to generic records, that is, to the aforementioned subtypes of `ParquetRecord`. Still, you must provide `MessageType` during writing. If you do not provide it during reading, then Parquet4s uses the schema stored in a file, and all its content is read. 

## Supported types

### Primitive types

| Type                                    | Reading and Writing | Filtering |
| :-------------------------------------- | :-----------------: | :-------: |
| Int                                     |      &#x2611;       | &#x2611;  |
| Long                                    |      &#x2611;       | &#x2611;  |
| Byte                                    |      &#x2611;       | &#x2611;  |
| Short                                   |      &#x2611;       | &#x2611;  |
| Boolean                                 |      &#x2611;       | &#x2611;  |
| Char                                    |      &#x2611;       | &#x2611;  |
| Float                                   |      &#x2611;       | &#x2611;  |
| Double                                  |      &#x2611;       | &#x2611;  |
| BigDecimal with INT96 [^1]              |      &#x2611;       | &#x2611;  |
| BigDecimal with INT64 [^1]              |      &#x2611;       | &#x2611;  |
| BigDecimal with INT32 [^1]              |      &#x2611;       | &#x2611;  |
| java.time.LocalDateTime with INT96 [^2] |      &#x2611;       | &#x2612;  |
| java.time.LocalDateTime with INT64 [^2] |      &#x2611;       | &#x2611;  |
| java.time.Instant with INT96 [^2]       |      &#x2611;       | &#x2612;  |
| java.time.Instant with INT64 [^2]       |      &#x2611;       | &#x2611;  |
| java.time.LocalDate                     |      &#x2611;       | &#x2611;  |
| java.sql.Timestamp with INT96 [^2]      |      &#x2611;       | &#x2612;  |
| java.sql.Timestamp with INT64 [^2]      |      &#x2611;       | &#x2611;  |
| java.sql.Date                           |      &#x2611;       | &#x2611;  |
| Array[Byte]                             |      &#x2611;       | &#x2611;  |

[^1] You can change de default binary (INT96) format of decimal, as well as its scale and precision, with help of utilities available in `com.github.mjakubowski84.parquet4s.DecimalFormat`. For example, to create a decimal expressed as INT32 number with scale of 2 and precision 10:

```scala mdoc:compile-only
import com.github.mjakubowski84.parquet4s.DecimalFormat

val MyDecimalFormat = DecimalFormat.intFormat(scale = 2, precision = 10, rescaleOnRead = false)
// imported implicits override default type classes required for reading, filterig and writing data containing decimal values
import MyDecimalFormat.Implicits._
```

Take note of `rescaleOnRead` flag. By default, during reading, Parquet4s rescales decimal values from original source format to one matching Parquet4s format. You can use the flag to change this behaviour and keep the decimals as they are stored in the source Parquet files.

[^2] You can change the default format of the timestamp column from INT96 to INT64 by importing type classes:

- INT64 micros format: `import com.github.mjakubowski84.parquet4s.TimestampFormat.Implicits.Micros._`
- INT64 mills format: `import com.github.mjakubowski84.parquet4s.TimestampFormat.Implicits.Millis._`
- INT64 nanos format: `import com.github.mjakubowski84.parquet4s.TimestampFormat.Implicits.Nanos._`

The imports contain type classes supporting projection, filtering and writing.

### Complex Types

Complex types can be arbitrarily nested.

- Option
- List
- Seq
- Vector
- Set
- Array - An array of bytes is treated as primitive binary
- Map - **Key must be of primitive type**, only the **immutable** version.
- Any Scala collection that has Scala collection `Factory` (in 2.12 it is derived from `CanBuildFrom`). Refers to both mutable and immutable collections. Collection must be bounded only by one type of element - because of that Map is supported only in the immutable version.
- *Any case class*

### Custom Types

Parquet4s is built using Scala's type class system. That allows you to extend Parquet4s by defining your own implementations of type classes.

For example, you may define a codec for your own type so that it can be read from or written to Parquet. Assuming that you have your own type:

```scala
case class CustomType(i: Int)
```

You want to save it as optional `Int`. In order to achieve that you have to define a codec:

```scala mdoc:compile-only
import com.github.mjakubowski84.parquet4s.{OptionalValueCodec, IntValue, Value, ValueCodecConfiguration}

case class CustomType(i: Int)

implicit val customTypeCodec: OptionalValueCodec[CustomType] = 
  new OptionalValueCodec[CustomType] {
    override protected def decodeNonNull(value: Value, configuration: ValueCodecConfiguration): CustomType =
      value match {
        case IntValue(i) => CustomType(i)
      }
    override protected def encodeNonNull(data: CustomType, configuration: ValueCodecConfiguration): Value =
      IntValue(data.i)
}
```

`ValueCodec` composes `ValueEncoder` and `ValueDecoder`, so if you need only to read or only to write your type, then it is enough if you implement only one of them.

Additionally, if you want to write your custom type, you have to define the schema for it:

```scala mdoc:compile-only
import org.apache.parquet.schema.{LogicalTypeAnnotation, PrimitiveType}
import com.github.mjakubowski84.parquet4s.TypedSchemaDef
import com.github.mjakubowski84.parquet4s.{LogicalTypes, SchemaDef}

case class CustomType(i: Int)

implicit val customTypeSchema: TypedSchemaDef[CustomType] =
  SchemaDef.primitive(
    primitiveType = PrimitiveType.PrimitiveTypeName.INT32,
    required = false,
    logicalTypeAnnotation = Option(LogicalTypes.Int32Type)
  ).typed[CustomType]
```

In order to filter by a field of a custom type `T` you have to implement `FilterEncoder[T]` type class.

```scala mdoc:compile-only
import com.github.mjakubowski84.parquet4s.FilterEncoder
import org.apache.parquet.filter2.predicate.Operators.IntColumn

case class CustomType(i: Int)

implicit val customFilterEncoder: FilterEncoder[CustomType, java.lang.Integer, IntColumn] =
  FilterEncoder[CustomType, java.lang.Integer, IntColumn](
    encode = (customType, _) => customType.i
  )
```

# Using generic records directly

Parquet4s allows you to choose to use generic records explicitly from the level of API in each module of the library. But you can also use typed API and define `RowParquetRecord` as your data type. Parquet4s contains type classes for encoding, decoding and schema provisioning for `RowParquetRecord`.

```scala mdoc:compile-only
import com.github.mjakubowski84.parquet4s.{ParquetReader, ParquetWriter, Path, RowParquetRecord}
import org.apache.parquet.schema.MessageType

// both reads are equivalent
ParquetReader.generic.read(Path("file.parquet"))
ParquetReader.as[RowParquetRecord].read(Path("file.parquet"))

val data: Iterable[RowParquetRecord] = ???
// when using generic record you need to define the schema on your own
implicit val schema: MessageType = ???

// both writes are equivalent
ParquetWriter
  .generic(schema) // schema is passed explicitly
  .writeAndClose(Path("file.parquet"), data)
ParquetWriter
  .of[RowParquetRecord] // schema is passed implicitly
  .writeAndClose(Path("file.parquet"), data)
```