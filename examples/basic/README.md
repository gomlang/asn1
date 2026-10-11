# ASN.1 example

This example uses `ecosystem::asn1` from the library root. Its test encodes and decodes a DER sequence using an explicit positional schema, then rejects a mismatched schema.

Run `(cd ../verification && just ecosystem-test asn1)` from the library root.

This example shares the library root manifest and its dependencies. From the library root, run `goml test` to build and test the library and its examples.
