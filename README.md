# ASN.1 DER

`ecosystem::asn1` implements a bounded, canonical subset of ASN.1 DER in GoML. It supports BOOLEAN, signed `i64` INTEGER, NULL, OCTET STRING, UTF8String, OBJECT IDENTIFIER, and SEQUENCE. OID arcs are unsigned `u64` values. No runtime reflection, implicit struct mapping, BER indefinite lengths, high-tag-number form, SET sorting, optional fields, arbitrary precision integers, or certificate validation is included.

`Value` is the explicit typed tree. `Schema` is a positional description of expected types and sequence fields. `encode_as` validates a value before encoding; `decode_as` parses exactly one DER value and validates the resulting tree. The schema must be supplied by the caller; it does not infer a GoML type from bytes.

```goml
use ecosystem::asn1;
use ecosystem::asn1::{Schema, Value};

fn encode_identifier() -> Result[Vec[u8], asn1::Error] {
    let schema = Schema::Sequence(Vec::from_array([Schema::Oid, Schema::Integer]));
    let value = Value::Sequence(
        Vec::from_array([Value::Oid(Vec::from_array([1, 2, 840, 113549])), Value::Integer(7)]),
    );
    asn1::encode_as(value, schema, asn1::Limits::standard())
}
```

`encode` and `decode` operate on the same typed tree without a schema. `encode_oid` and `decode_oid` handle OID content octets without a DER tag and length. DER decoding rejects nonminimal lengths and integers, invalid BOOLEAN encodings, malformed UTF-8, truncated or overlong OID arcs, unsupported tags, and trailing bytes. Diagnostics include an error kind and byte offset; schema mismatches use offset zero because validation happens on the decoded tree.

`encode_bit_string(data, unused_bits, limits)` emits a complete primitive DER
BIT STRING; `decode_bit_string(input, limits)` returns `(data, unused_bits)` in
independent storage. Bits run from most to least significant within each octet.
The unused-bit count must be 0–7, unused low bits must be zero, and empty data
requires zero unused bits. Decoding rejects another tag, trailing data,
constructed encodings and noncanonical padding. Byte and element limits apply
to the complete DER value. These helpers preserve the supplied bit length;
ASN.1 named-bit-list schemas require the caller to remove trailing zero bits
according to X.690 section 11.2.2. They do not add `Value` or `Schema` variants.

For explicit ASN.1 structures that contain additional tags, `Tlv`, `parse_tlv`, `split_tlvs`, and `encode_tlv` expose bounded single-byte-tag DER framing. `validate_der` walks constructed values, enforces depth and element limits, validates the primitive types supported above, and applies lexicographic
SET OF ordering to universal tag 17. General ASN.1 SET components use a different
schema-dependent tag order; this generic check is not sufficient to establish
canonical DER for arbitrary SET schemas.

Universal tags must use their required primitive or constructed form, including
types outside `Value`/`Schema`: REAL, RELATIVE-OID, TIME, ObjectDescriptor and
restricted character strings require primitive DER encodings; EXTERNAL,
EMBEDDED PDV and unrestricted CHARACTER STRING require constructed encodings.
Their additional type-specific content rules still require an application
schema. Application, context-specific and private tags retain either form.

Raw framing counts each outer TLV against `max_elements`: parsing or encoding
one requires a budget of at least one, while splitting empty input requires none.
Constructed contents remain opaque to these framing APIs; use `validate_der`
for recursive depth, element and content validation. `encode_tlv` rejects
obviously oversized content before copying it, and the byte budget includes
the tag and complete DER length header.

`validate_der` also checks minimal nonempty ENUMERATED integer contents and the repertoires
of NumericString (digits and space), PrintableString (ASN.1 letters, digits and
its restricted punctuation), IA5String (7-bit bytes), and VisibleString (ASCII
space through tilde). These checks apply recursively inside constructed values;
raw TLV framing APIs preserve content without these semantic checks.

Universal UTCTime and GeneralizedTime are also validated: complete seconds and
uppercase `Z` are required. GeneralizedTime permits a period followed by a
nonempty fractional part with no trailing zero; UTCTime has no fractional part.
Calendar components use Gregorian month lengths and leap years, hours 00–23,
and minutes 00–59. UTCTime seconds are 00–59 under X.680; its two-digit year does
not imply a century window (year `00` may represent a leap century).
GeneralizedTime accepts second `60` only at month-end 23:59 as a possible ISO 8601
leap-second position; historical/future leap-second announcements are not checked.
Four-digit years, including `0000`, are interpreted proleptically. These are TLV
validation rules, not new `Value`/`Schema` variants or certificate time policies.

See
[ITU-T X.680](https://www.itu.int/rec/T-REC-X.680-202102-I/en) for character
repertoires and [ITU-T X.690](https://www.itu.int/rec/T-REC-X.690-202102-I/en)
for encoding rules. Unknown primitive tags retain their content bytes; their type-specific rules require an application schema. High-tag-number form and universal tag zero are rejected, including both primitive (`0x00`) and constructed (`0x20`) forms. Universal zero is reserved for encoding rules and is not a DER value; tag zero remains available in the application, context-specific and private classes.

`Limits::standard()` allows at most 1 MiB of input or output, 32 nested sequence levels, 10,000 elements, and 64 OID arcs. Callers may supply stricter limits. All four limits are enforced during decoding and encoding. OID subidentifiers that do not fit `u64` and INTEGERs that do not fit `i64` return errors. Encoded output is canonical for the supported subset.

Run `(cd ../verification && just ecosystem-test asn1)` from this library repository to test the library, example, downstream verification, and cached build.

## Development and examples

Requires the [current GoML toolchain](https://github.com/gomlang/verification/blob/main/ci/toolchain.json) with unversioned registry support. The `examples/basic/` example shares the root manifest and its dependencies. From the library root, run:

```sh
goml run --example basic
goml test
(cd ../verification && just ecosystem-test asn1)
```

`goml test` builds the example and runs its tests. `(cd ../verification && just ecosystem-test asn1)` runs the library-specific smoke and compatibility checks.

### Arbitrary-width INTEGER bytes

`encode_integer_bytes(payload, limits)` and `decode_integer_bytes(der, limits)`
encode and decode signed INTEGER values as big-endian two's-complement octets.
Their payload is the shortest nonempty signed representation: zero is `[0]`,
positive 128 is `[0, 128]`, and negative 129 is `[255, 127]`. Redundant sign
extension, empty values, noncanonical DER, trailing data and non-INTEGER tags
are rejected. These APIs preserve integers wider than `i64`; `Value::Integer`
and `Schema::Integer` retain their existing `i64` behavior. Neither API performs
arithmetic or unsigned-magnitude conversion. Returned vectors own their storage.

`max_bytes` bounds the entire DER encoding, including long-form length octets;
`max_elements` must admit one primitive and depth zero is sufficient. Inputs must
remain stable during each call. Encoding validates the supplied representation
rather than silently removing sign octets.
