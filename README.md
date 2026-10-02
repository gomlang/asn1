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

For explicit ASN.1 structures that contain additional tags, `Tlv`, `parse_tlv`, `split_tlvs`, and `encode_tlv` expose bounded single-byte-tag DER framing. `validate_der` walks constructed values, enforces depth and element limits, validates the primitive types supported above, and checks DER SET ordering.
It also checks minimal nonempty ENUMERATED integer contents and the repertoires
of NumericString (digits and space), PrintableString (ASN.1 letters, digits and
its restricted punctuation), IA5String (7-bit bytes), and VisibleString (ASCII
space through tilde). These checks apply recursively inside constructed values;
raw TLV framing APIs preserve content without these semantic checks. See
[ITU-T X.680](https://www.itu.int/rec/T-REC-X.680-202102-I/en) for character
repertoires and [ITU-T X.690](https://www.itu.int/rec/T-REC-X.690-202102-I/en)
for encoding rules. Unknown primitive tags retain their content bytes; their type-specific rules require an application schema. High-tag-number form and end-of-contents markers are rejected.

`Limits::standard()` allows at most 1 MiB of input or output, 32 nested sequence levels, 10,000 elements, and 64 OID arcs. Callers may supply stricter limits. All four limits are enforced during decoding and encoding. OID subidentifiers that do not fit `u64` and INTEGERs that do not fit `i64` return errors. Encoded output is canonical for the supported subset.

Run `(cd ../verification && just ecosystem-test asn1)` from this library repository to test the library, example, downstream verification, and cached build.

## Development and examples

Requires GoML 0.1.56 or newer. The `examples/basic/` example shares the root manifest and its dependencies. From the library root, run:

```sh
goml run --example basic
goml test
goml verify --timeout 300s
```

`goml test` builds the example and runs its tests. `goml verify` repeats the example checks as an independent module against an isolated registry snapshot. `(cd ../verification && just ecosystem-test asn1)` also retains the library-specific smoke and compatibility checks.
