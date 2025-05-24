# Plan for `uint256` Type Support

This document summarizes the existing `rw_int256` implementation and outlines the work required to add an unsigned 256‑bit integer type (`rw_uint256`).

## Current `rw_int256` Implementation

* **Data type definition** – `DataType` includes `Int256` as a variant parsed from `"rw_int256"`【F:src/common/src/types/mod.rs†L175-L177】【F:src/frontend/src/binder/expr/mod.rs†L1046-L1053】.
* **Underlying struct** – `Int256` wraps `ethnum::i256` with a corresponding reference type `Int256Ref` and a macro implementing common traits【F:src/common/src/types/num256.rs†L33-L52】【F:src/common/src/types/num256.rs†L186-L188】.
* **Arrow mapping** – Arrow conversion represents `Int256` as `Decimal256`【F:src/common/src/array/arrow/arrow_impl.rs†L356-L360】.
* **Value encoding** – serialization handles the 32‑byte little-endian format for `Int256` values【F:src/common/src/util/value_encoding/mod.rs†L208-L239】【F:src/common/src/util/value_encoding/mod.rs†L339-L340】.
* **Connectors** – PostgreSQL `NUMERIC` values can be mapped to `Int256` by `pg_numeric_to_rw_int256`【F:src/connector/src/parser/scalar_adapter.rs†L341-L349】.

## Feasibility of `rw_uint256`

RisingWave already depends on the `ethnum` crate which provides the `u256` type. Since the signed `i256` integration is complete across arrays, Arrow, value encoding and connectors, a `u256` based type can reuse much of this infrastructure. The main challenges are:

1. PostgreSQL and Arrow lack a native unsigned 256‑bit type. Encoders must use compatible representations (`NUMERIC` or decimal strings for Postgres; `Decimal256` or string for Arrow).
2. Casting and arithmetic rules must exclude signed operations (e.g., negation).

## Implementation Plan

1. **New Wrapper Types**
   * Create `UInt256` and `UInt256Ref` in `src/common/src/types/num256.rs` mirroring `Int256` but using `ethnum::u256`.
   * Implement methods/macros with `impl_common_for_num256!`.

2. **DataType Variants**
   * Add `Uint256` and `DataTypeName::Uint256` to `DataType` and protobuf mappings.
   * Extend `postgres_type.rs` so `DataType::Uint256` maps to `"rw_uint256"` and has an OID entry in system catalog.

3. **Arrays and Builders**
   * Create `Uint256Array`/`Uint256ArrayBuilder` in `src/common/src/array/num256_array.rs` (similar to existing `Int256Array`).

4. **Arrow Integration**
   * Use `Decimal256` to encode `Uint256` values; adjust `arrow_impl.rs` and `arrow_iceberg.rs` to convert using unsigned semantics.

5. **Value Encoding**
   * Extend serialization/deserialization logic in `value_encoding/mod.rs` for the new type.

6. **Expression Functions and Casting**
   * Provide casting rules and arithmetic functions (add, sub, mul, div, mod) similar to `Int256` but without `Neg` or `CheckedNeg`.
   * Implement a helper `hex_to_uint256` for hexadecimal parsing.

7. **Binder and Type Parsing**
   * Recognize the alias `"rw_uint256"` in the binder when parsing column types and function signatures.
   * Optional: extend literal parsing to support numeric constants that fit into `u256`.

8. **Connectors**
   * Update `ScalarAdapter` to allow PostgreSQL `NUMERIC` values to decode into `Uint256` when requested.
   * JSON/Avro formats should treat `Uint256` as string to avoid overflow.

9. **Aggregates and Functions**
   * Add aggregate signatures (`sum(uint256) -> uint256`, `avg(uint256) -> float8`, etc.) and update macros in `expr/macro/src/types.rs`.

10. **Tests and Documentation**
    * Provide unit tests similar to `hex_to_int256` and array tests.
    * Document the new type in the SQL data types guide.

With the existing `rw_int256` code path as reference, introducing `rw_uint256` is feasible. The main implementation surface area mirrors that of `Int256` with additional handling for external encodings and unsigned semantics.
