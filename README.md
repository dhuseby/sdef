# Serialized Data Expression Format

## Data Expressions

There are only two forms that these expressions may take: regular and type-define. The regular form is an expression surrounded by parenthesis:

```
(<type name> <type parameters>)
```

Expressions of this form are serialized data. The type for the data is specified by the type name and the type parameters are the data for each part of the type.

The type-define form is for defining concrete types. The type-define form starts with a left parenthesis and a single quote followed by a string, a space, and one or more s-expressions or a single regex string. It ends with a right parenthesis: 

```
('<type name> <s-expr>+|<regex>)
```

When defining the new type with `('` the value for the type can either be a single regular expression or it can be one or more s-expressions that bind types to named members of the type. This is all that is needed to define data types and how they are parsed.

Each type definition gets added to the global list of types as aliases for the regular expression in the type regex. The type regex may reference other, previously defined types so that the regex is built up recursively.

Type definitions and regular data expressions can be in the same file. The only restriction is that a type definition must appear before any data expressions referencing the type occur.

## Extensibility

To make this fully extensible, the only other special form needed is an expression to include other files at parsing time. To accomplish that, a special type-define expression with no type name is used to mean "include and process the following files". These are called "include expressions". An include expression defines one or more URI's that refer to files that should be parsed and processed at the point of the include expression. This allows for reuse of type definitions and to also include data from other files.

As an example, I may extend my own local type definitions with the standard one for comments like so:

```
(' https://w3c.org/sdef/comment.sdef)

("A single hexidecimal character")
('hex_char [0-9a-fA-F])

("A hex encoded key is 128 hex chars")
('KeyHex hex_char{128})

("A list of one or more keys")
('Keys KeyHex+)
```

The `('` include tells the parser to download and process the `comment.sdef` file before proceeding. The `comment.sdef` file defines just the comment type. By including the comment type definition, my local type definition file can have comments in it anywhere. For the record, the definition of a comment type is:

```
('" (?:[^"]*)(?:\"))
```

This says that any expression starting with the double quotation mark `"` should match all characters that are not a double quotation mark--including newlines--and then match an ending double quotation mark. They are defined as non-matching groups in the regex because we want to ignore the data in a comment. The result of processing any comment is "nil", or void, which causes the parser to ignore it. Thus comments can be placed anywhere in a serialized data file.

If the type definitions for hexchar and KeyHex are in the file named `keyhex.sdef` stored at `http://foo.se/` then an example of serialized data that defines a Keys list of KeyHex objects looks like the following:

```
(' http://foo.se/keyhex.sdef)

("My public keys")
(Keys
    1a87a6fa65a9663c4192fe74ccb88a51bb1b2251d7041a858512f970bf1230d2454617014478c93a6b319dfc7f9353d9bcd46c5d031d2e2be3339170971a09c3
    454617014478c93a6b319dfc7f9353d9bcd46c5d031d2e2be3339170971a09c31a87a6fa65a9663c4192fe74ccb88a51bb1b2251d7041a858512f970bf1230d2
)
```

## Complex Data Types

Defining complex data types is as simple as nesting named expressions under a newly defined type. Let's assume I want to define a Person type that includes one or more email addresses and one or more phone numbers. First I have to define the email and phone number types:

```
(' http://w3c.org/sdef/comment.sdef)

("An email requires a crazy regex")
('email (?:[a-z0-9!#$%&'*+/=?^_`{|}~-]+(?:\.[a-z0-9!#$%&'*+/=?^_`{|}~-]+)*|"(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21\x23-\x5b\x5d-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])*")@(?:(?:[a-z0-9](?:[a-z0-9-]*[a-z0-9])?\.)+[a-z0-9](?:[a-z0-9-]*[a-z0-9])?|\[(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?|[a-z0-9-]*[a-z0-9]:(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21-\x5a\x53-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])+)\]))

("A phone number is 10 digits, no spaces")
('phone [0-9]{10})
```

Assume that the phone and email types are stored in a file called `pfields.sdef`. Then the definition of a `Person` type could look like:

```
(' http://foo.se/pfields.sdef)

("A person has one or more emails followed by one or more phones")
('Person
    (emails email+)
    (phones phone+)
)
```

With an example of a serialized Person object looks like this:

```
(' http://foo.se/person.sdef)
(Person
    (emails
        joe@schmoe.com
        joe@foo.com
    )
    (phones
        9119119111
        8005551212
    )
)
```

It is important to note a few things about the definition of complex data objects. First of all in the `('Person` defition, instead of giving a regex directly, the definition includes two s-expressions that bind names to types. This is how named fields of a type are defined. So in the case above, each `Person` has two fields, one named `emails` and the other named `phones`. Furthermore, `emails` is defined as a list of one or more `email` type data units and `phones` is defined as a list of one or more `phone` type data units.

When defining lists of objects, the data type for each object can be inferred from the type definition and do not require explicit definition. It is important to note that all data units in a list are of the same type to simplify parsing. The follow is also a valid encoding of the above `Person` object but is pedantic and not necessary:

```
(' http://foo.se/person.sdef)
(Person
    (emails
        (email joe@schmoe.com)
        (email joe@foo.com)
    )
    (phones
        (phone 9119119111)
        (phone 8005551212)
    )
)
```

The explicit encoding of each email and phone number object is unnecessary because the parser generated from the type definition knows that all objects in `emails` are `email` typed. Same goes for the list of phone numbers. The extra encoding is implied and redundant.

#### Field Ordering

One other aspect of the type definition process is that the order of the fields is significant. Unlike JSON, the field ordering in the type definitions elliminates the need for an externally defined and arbitrary canonicalization algorithm. All `Person` objects have a list of `emails` and then a list of `phones`.

If you defined another type, say `AntiPerson` that had a list of `phones` before a list of `emails`, the two types would be considered distinct and different because the order of their fields are different.

This design detail is important to facilitate automated parser generation and completes the self-describing attribute of this encoding scheme.

#### Type Names

By convention, top-level or first-order objects are named with camel case names. In the examples above, objects like `Person` and `HexKey` are top-level types. Names for basic types should be short and are lower case with underbars separating words. In the examples above, types such as `email` and `phone` and `hex_char` are basic types. Also by convention, field names in complex types are also all lower case with underbars separating words.

### Canonicalization

To canonicalize any SDEF encoded object, the only thing we need to worry about is normalizing whitespace. The rules for normalizing whitespace are simple. We need to preserve one space between expression parameters unless they are themselves an s-expression. The following is also a valid serialization of the `Person` object above:

```
(' http://foo.se/person.sdef)(Person (emails joe@schmoe.com joe@foo.com)(phones 9119119111 8005551212))
```

Note that there is no space between s-expressions but there is a space between the email addresses and the phone numbers. Technically the space between an expression type identifier and the expression parameters is not necessary but we add one for readability. The only exception is if the regex for a given type parameter is a non-matching group such as the definition for comments. The regex is non-matching so we don't put a space after `("` which also increases readability. 

The above normalized and serialized form of the Person object is used when sending data over a wire or when doing some cryptography such as calculating a digital signature over a piece of data.

## Digital Signatures

One feature that many serialization formats need to tackle is how to include digital signatures over data. JSON has to use a combination of application-specific rules (e.g. signature is included last--see Secure ScuttleButt format) plus a complicated canonicalization algorithm to get JSON objects into some normalized form for the digital signature creation and verification to work. 

With data in serialized data expression format (SDEF), signatures are just another data type and included somewhere in a data object. By convention they should probably always be last in an object just to make both creating and verifying digital signatures easier for implementors.

Any digital signature needs four things:

0. A standardized name for the signature algorithm used. This defines the type of public key to expect and which digest and encryption algorithm was used to create the digital signature.
1. A public key to validate the signature with. This can either be an inline key or a reference to a key somewhere else.
2. The data object that is signed.
3. The value of the digital signature.

First of all, you would think we would need to define a list of signature formats like so:

```
('SignatureFormat Ed25519Signature2019|RsaSignature2019)
```

However since there will only be one of these values in each of the types of signature objects, it makes more sense to specify the signature format when we define the signature types.

Next let's define what a Key and KeyRef look like:

```
('hex [0-1a-fA-F])
('Ed25519Key hex{128})
('RsaKey4096 hex{1024})
('Key Ed25519Key|RsaKey4096)
('uri \w+:(\/?\/?)[^\s]+)
('KeyRef uri)
```

Lastly, let's bring it all together into one SDEF file that defines what a digital signature looks like:

```
(' http://w3c.org/sdef/comment.sdef)

("Define basic value types:")
    ('hex [0-1a-fA-F])
    ('uri \w+:(\/?\/?)[^\s]+)

("Define the primitive types:")
    ('Ed25519Key hex{128})
    ('RsaKey4096 hex{1024})
    ('Key Ed25519Key|RsaKey4096)
    ('KeyRef uri)
    
    ("Ed25519Sigs are 128 bytes in length.")
    ('Ed25519Sig hex{128})
    
    ("PKCS#1 says the sigs are the same size as the modulus")
    ('RsaSig4096 hex{1024})

("Define the two different types of signatures:")
    ('Ed25519Signature
        (type Ed25519Signature2019)
        (key Ed25519Key)
        (sig Ed25519Sig)
    )
    ('RsaSignature
        (type RsaSignature2019)
        (key RsaKey4096)
        (sig RsaSig4096)
    )
    
("Define a generic signature:")
    ('Signature Ed25519Signature|RsaSignature)
```

Now if we take our example of a `Person` from above and add a signature to it the following is the result (**NOTE:** the hex values are random, invalid values used just as an example):

```
(' pfields.sdef signature.sdef)

("Re-define Person as having a signature")
('Person
    (emails email+)
    (phones phone+)
    (signature Signature)
)

(Person
    (emails
        joe@schmoe.com
        joe@foo.com
    )
    (phones
        9119119111
        8005551212
    )
    (signature
        (type Ed25519Signature2019)
        (key 454617014478c93a6b319dfc7f9353d9bcd46c5d031d2e2be3339170971a09c31a87a6fa65a9663c4192fe74ccb88a51bb1b2251d7041a858512f970bf1230d2)
        (sig 1a87a6fa65a9663c4192fe74ccb88a51bb1b2251d7041a858512f970bf1230d2454617014478c93a6b319dfc7f9353d9bcd46c5d031d2e2be3339170971a09c3)
    )
)

(Person
    (emails
        jane@doe.com
        jane@bar.com
    )
    (phones
        2112112111
    )
    (signature
        (type RsaSignature2019)
        (key 621cfb78c12a99a6528cae2fdab472236234ef1a6df7d14e9783ade7f171d11665363d420bf25d37b5a82df816aa2a2db228e896805c2869d94bd7fb3271899dbdf697cf476d61fcf76d1932792f6bdaf6f94234596891b43da0840eda739bcc72bfe5b012b732c4dd2f7eece64f6f8c13f9c0871670e9b6706251895a8e504f12b8b010c4854ea8fce46facc59aa95472d21580ca63d774ce4e5107c50b6258740d7f84456b40b0adf4f8ef8e17e885c77a85052f5b7581a92e5d7779e8018392697f02dfb5e5ad7732b36cae8f0c0a21c65f277644a72929064c59935fe48e68d6141ef8f0cfd85e9ed1908e3e340310b39cbf229eeab1518f2ace406372c5f55c5fc205c3c37356ae56861fcf28d1bb19e44fa39bf27ebe58f56141c96a30f4cc05f7dea3d4ca99fca20195b47117a36722675d2940fac8ea58fd9f75a8705932ae4ffca3c70c418a15dd9fa2701805cbabff79b2a0bd78c8a802784cd2a2971e52e1b9d63e6fddbeb459771b3c8fb3504a1da64f55a66ba6be010e8c00d0505b92c9717616eedfddb4a2cb18c4eff0917e8f56e67a0de47711db5faea611e8f9df11a418fc54af7c022dcf707e6933dbc3db40414881a843192a8bc9a3115cc930310f402dd3602dbc9c473cb5dfbfa8d47da86360488e86a87adfd02639b64ecc0f64c55d251465a41b59378407e4d8d3d284124b27a634fa42221d8d3f)
        (sig edce3a83d8b04868dce12303ff0149c8037fa20ad10f628d9801eb05216556a6162162e7a0e2a37b3c4d987683027b02b5391b547cc07a6f743637a42816d0da21dd96e4b75ca89f6073df3a2570b19b35d6bdf32d42f97b2b126e6a3cf4ab3469ab319c0b2dca492e3d774d38d344f284e311f0fc107be84cd400c5ff9b906824219585f5e47049167d87968f5df17fe25b2cdc322fba06bb40e0cee0e3d8154d368814c478f9dde1fd3cb32cebd925646aad52954176bf3407809a42071918b11da802564b202638334a5cbadbf344b0a2c0e559844e536a58cb573dac3c5ab79f1278dba67ed290d02306f64eee35227a1f8a76ffcd30df304c0067992326f6122efb54150d87e1235a26e9f052f4554bfa2eefb4f9d0b088ad2bfb2b8d45f0008fa4053e64bff781470062ed3f6750ea0f25a476916d8d2d99ce5c796966bbbc3f23b89f92fad520f9aa4ea325dd2bed2e387dce0c6adc320afae4ba1e8d22bc5a4ec78a269927fb3cad8ff27cd623b6494724fd2887e922c153c52a81eb0fcd33ae9f4fd9e92d08999214bd153584ffac0bed1f235d282c607d9debc3a9d33151d9c4c678147ca2f19124b1e6a69c5d66535231f157620b85dd0ec7ede392aaa846cfe7defbcebace7de5e788f1787dd5513ddb6d27acdf5e4ed5934524f1f9e6d9695eeb2d618cbf8ce2e185cc235696c3e3539305fa53fc5f9b4dd4a6)
    )
)
```

The above example shows a way for having inline signatures in data objects however it probably makes more sense to define a new object called `SignedPerson` that contains both the `Person` object and the `Signature` for the object. This actually makes creating and validating digital signatures much easier for implementors. Here's how things change:

```
(' http://w3c.org/sdef/comment.sdef)

("Define basic value types:")
    ('hex [0-1a-fA-F])
    ('uri \w+:(\/?\/?)[^\s]+)

("Define the primitive types:")
    ('Ed25519Key hex{128})
    ('RsaKey4096 hex{1024})
    ('Key Ed25519Key|RsaKey4096)
    ('KeyRef uri)
    
    ("Ed25519Sigs are 128 bytes in length.")
    ('Ed25519Sig hex{128})
    
    ("PKCS#1 says the sigs are the same size as the modulus")
    ('RsaSig4096 hex{1024})

("Define the two different types of signatures:")
    ('Ed25519Signature
        (type Ed25519Signature2019)
        (key Ed25519Key)
        (sig Ed25519Sig)
    )
    ('RsaSignature
        (type RsaSignature2019)
        (key RsaKey4096)
        (sig RsaSig4096)
    )
```

Note that there is no definition of a generic signature because we only want to allow onle Ed25519 digital signatures on our `SignedPerson`. Next let's define the `SignedPerson` assuming that the `Person` type is defined in the file `person.sdef` and the `Ed25519Signature` type is defined in the file `signature.sdef`:

```
(' person.sdef signature.sdef)

('SignedPerson
    (person Person)
    (signature Ed25519Signature)
)
```

Assuming the `SignedPerson` object is defined in the file named `signed_person.sdef`, an example use of `SignedPerson` looks like:

```
(' signed_person.sdef)

(SignedPerson
    (person
        (emails
            joe@schmoe.com
            joe@foo.com
        )
        (phones
            9119119111
            8005551212
        )
    )
    (signature
        (type Ed25519Signature2019)
        (key 454617014478c93a6b319dfc7f9353d9bcd46c5d031d2e2be3339170971a09c31a87a6fa65a9663c4192fe74ccb88a51bb1b2251d7041a858512f970bf1230d2)
        (sig 1a87a6fa65a9663c4192fe74ccb88a51bb1b2251d7041a858512f970bf1230d2454617014478c93a6b319dfc7f9353d9bcd46c5d031d2e2be3339170971a09c3)
    )
)
```

The reason this is much easier for implementors is that they will receive a `SignedPerson` object with two fields--person and signature. To verify the digital signature, the implementor extracts the key and sig values from the signature, then grabs the Person object from the person field, canoncicalizes it and then passes all three to the signature verification routine.

With inline signatures, implementors have to remove the signature field from the data object before canonicalizing and verifying the signature. Depending on the implementation this is not always trivial nor memory/cpu friendly. By wrapping a data object in a signature envelope type the operations are likely to be easier to implement however this format doesn't prescribe one way over the other. Inline signatures or enveloped signatures are both valid and it is up to the implentor to decide.

One last example showing how a `SignedPerson` can be defined to contain a generic signature that can be either a `Ed25519Signature` or `RsaSignature4096`:

```
(' person.sdef signature.sdef)

('SignedPerson
    (person Person)
    (signature Ed25519Signature|RsaSignature4096)
)
```

This makes it possible to automatically parse a `SignedPerson` with either of the two signature types.

## Standard Types

To further simplify and standardize the use of SDEF, a set of standard basic types shoudl be defined. Types such as hexidecimal strings, Base64/Base58/etc strings, URI strings, Email strings, etc are probably all worth defining so that others don't. The hope is that we can settle on some standard regular expressions for basic types that everybody uses to increase reuse and simplicity.

```
('" (?:[^"]*)(?:\"))

("hexidecimal characters for hex strings")
    ('hex [0-1a-fA-F])
    
("A URI string")
    ('uri \w+:(\/?\/?)[^\s]+)

("An email address")
    ('email (?:[a-z0-9!#$%&'*+/=?^_`{|}~-]+(?:\.[a-z0-9!#$%&'*+/=?^_`{|}~-]+)*|"(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21\x23-\x5b\x5d-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])*")@(?:(?:[a-z0-9](?:[a-z0-9-]*[a-z0-9])?\.)+[a-z0-9](?:[a-z0-9-]*[a-z0-9])?|\[(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?|[a-z0-9-]*[a-z0-9]:(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21-\x5a\x53-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])+)\]))

("Base64 encoded data")
    (?:[A-Za-z0-9+/]{4})*(?:[A-Za-z0-9+/]{2}==|[A-Za-z0-9+/]{3}=)?
```
