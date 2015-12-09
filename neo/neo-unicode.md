# Neo-Unicode

Neo Unicode is a repository with experimental, proof of concept code, that aims to improve
[Pharo](http://www.pharo.org)'s [Unicode](https://en.wikipedia.org/wiki/Unicode) support. 
This is a work in progress that is not yet suitable for public consumption.

The project consists of 2 packages, Neo-Unicode-Core and Neo-Unicode-Tests
in the http://mc.stfx.eu/Neo repository. There is no Metacello configuration.


## The current situation

In Pharo we have Unicode strings as a sequence of Unicode characters, 
each defined/identified by a code point (out of 10.000s covering all languages).

To encode Unicode for external representation as bytes, we use UTF-8 like the rest of the modern world. 

You can read more about the current situation in the first part of the [Character Encoding and Resource Meta Description](http://files.pharo.org/books/enterprisepharo/book/Zinc-Encoding-Meta/Zinc-Encoding-Meta.html) chapter of the Enterprise Pharo book.

All in all, the current situation is OK for day to day practical use.


## What is the problem ? What is missing ?

The world is a complex place and the [Unicode](http://www.unicode.org) standard 
tries to cover all possible languages, situations, usages and interpretations, combined with backward compatibility. 
For 1000s of pages, all kinds of exceptions and special rules are described.
Obviously we do not have enough support for all of this.

We do not need all of Unicode, nor do we need to implement all edge cases, but we need more. 
In particular, we need to support normalization, casing and collation/sorting.


## The Unicode Character Database

A first step for better Unicode support is to work with the actual data defined by the standard.
The primary element is the Unicode Character Database, which lists details about each of about 30.000 code points.

NeoUnicodeCharacterData holds this information per code point, 
loading the database in Pharo from an external URL.


## Normalization

Some characters or letters can be written in more than one way.

The simplest example in a common language is (the French letter é) is

    LATIN SMALL LETTER E WITH ACUTE [U+00E9]

which can also be written as

    LATIN SMALL LETTER E [U+0065] followed by COMBINING ACUTE ACCENT [U+0301]

The former being a composed normal form, the latter a decomposed normal form. 
There are different normalization forms, NFC, NFD, NFKC and NFKD.


## Casing

Recognizing lower, upper and title case and converting between them is defined by Unicode as well.
The Unicode Character Database can be used for these operations.

NeoUnicodeCaser is a simple tool to do these conversions.




## Collation/Sorting

Nothing yet.
