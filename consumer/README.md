# ASN.1 consumer

This independent module resolves `ecosystem::asn1 = "0.1.0"` through the verification registry. Its test encodes and decodes a DER sequence using an explicit positional schema, then rejects a mismatched schema.

Run `(cd ../../verification && just ecosystem-test asn1)` from the repository root.
