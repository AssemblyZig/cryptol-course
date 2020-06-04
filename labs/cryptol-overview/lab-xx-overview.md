# Introduction

Cryptol is a domain specific language and tool suite created by Galois, Inc with support from NSA cryptographers. The language has lots of cool programming language features that make it well suited for applications in high assurance systems and cryptographic development, including:

 * strong, static typing
 * type inference
 * parametric size-polymorphism
 * higher-order functions

Cryptol is used to create gold standard algorithm specifications and provides access to tools that facilitate their exploration and evalution.

This lab will provide a quick overview of Cryptol, some motivating applications where the language and technology have been deployed, and to the language features which make Cryptol an excellent choice for these applications.

More information about Cryptol is available at [http://www.cryptol.net](https://cryptol.net).


# Real World Cryptol

Cryptol has been used in the development and evaluation of high assurance cryptographic systems that enjoy wide use. Notable examples where Cryptol has been successfully applied by industry leaders to add assurance to cryptographic implementations include Amazon s2n and Microsoft's ElectionGuard.

Further examples are distributed with the [Cryptol software source](https://github.com/GaloisInc/cryptol) or are available for review on other projects hosted on the [Galois Github Page](https://github.com/GaloisInc/).


## Amazon s2n Continuous Integration

Amazon s2n is "a C99 implementation of the TLS/SSL protocols that is designed to be simple, small, fast, and with security as a priority". TLS/SSL is a suite of cryptographic protocols and algorithms used to provide integrity, confidentiality and other familiar security services. Amazon s2n is an implementation of this suite used to protect communications on Amazon's cloud infrastructure platforms such as Amazon Web Services (AWS) and Amazon Simple Storage Service (S3).

[TODO: A blurb about the application of Cryptol here]

Furthermore, these security property tests are performed as part of a continuous integration pipline using the [Travis Continuous Integration Service](https://travis-ci.com/). Whenever changes are made -- no matter how small -- to the C implementations, Cryptol and SAW evaluations are automatically run to ensure that no security properties of the system have been disrupted by the proposed updates.

A thorough description of the research, design decisions, and application of Cryptol to evaluating cryptographic implementations in Amazon's s2n system can be found in the paper [Contiuous Formal Verificationof Amazon s2n](https://link.springer.com/chapter/10.1007/978-3-319-96142-2_26). This paper was selected by NSA's Science of Security group for honorable mention in the [7th Annual Best Scientific Cybersecurity Paper Competition](https://cps-vo.org/group/sos/papercompetition/pastcompetitions).

You can review the code for yourself on [Amazon's s2n Github Repository](https://github.com/awslabs/s2n). The code relevant to the specification and evaluation of the HMAC routines can be found in the `tests/saw/` directory.

Further exposition on the development of these integration tests can be found in a three part series on the [Galois Inc. Blog](https://galois.com/blog/):
 * [Part 1](https://galois.com/blog/2016/09/verifying-s2n-hmac-with-saw/) - **Verifying s2n HMAC with SAW**
 * [Part 2](https://galois.com/blog/2016/09/specifying-hmac-in-cryptol/) - **Specifying HMAC in Cryptol**
 * [Part 3](https://galois.com/blog/2016/09/proving-program-equivalence-with-saw/) - **Proving Program Equivalence with SAW**

## Microsoft ElectionGuard

ElectionGuard SDK, including key ceremony, ballot encryption, encrypted ballot tally, and partial decryptions for knowledge proofs of trustees.

## Verifying Properties about Algorithms

Cryptol provides an easy interface for using powerful tools such as SAT solvers for verifying properties about algorithms we care about. Throughout this course we will introduce examples and explain how to take advantage of these tools in your own designs and evaluations. Here is an example packaged with the Cryptol source that demonstrates a simple but important property about an encryption algorithm which only uses the (XOR) operation:

```haskell
encrypt : {a}(fin a) => [8] -> [a][8] -> [a][8]
encrypt key plaintext = [pt ^ key | pt <- plaintext ]

decrypt : {a}(fin a) => [8] -> [a][8] -> [a][8]
decrypt key ciphertext = [ct ^ key | ct <- ciphertext ]

property roundtrip k ip = decrypt k (encrypt k ip) == ip
```

This file defines an `encrypt` operation, a `decrypt` operation, and a property called `roundtrip` which checks for all keys `k` and all input plaintexts `ip` that `decrypt k (encrypt k ip) == ip` (*i.e.* that these operations are inverse to one another).

We can see the effect of encrypting the particular input `attack at dawn` with the key `0xff`:

```haskell
Main> encrypt 0xff "attack at dawn"
[0x9e, 0x8b, 0x8b, 0x9e, 0x9c, 0x94, 0xdf, 0x9e, 0x8b, 0xdf, 0x9b,
 0x9e, 0x88, 0x91]
```

Cryptol interprets the string `"attack at dawn"` as a sequence of bytes suitable for the encrypt operations. We will introduce Cryptol types in this lab and discuss them in detail throughout this course.

Furthermore, we can prove this property holds in the interpreter using the `:prove` command and the currently configured SAT solver (Z3 by default):

```haskell
Main> :prove roundtrip : [8] -> [16][8] -> Bit
Q.E.D.
(Total Elapsed Time: 0.010s, using Z3)
```

Cryptol reports `Q.E.D.` indicating that our property is indeed true for all possible inputs. Here we must specify the length of the messages that we want to check this property for. This example checks the property for 16 character messages, but we could check this for any arbitrary length.


# Language Features

So what makes Cryptol special compared to other langauges?

Cryptol is a language designed with Cryptography specifically in mind -- much of the syntax and language was designed with the way that real cryptographers think about and design systems. This allows the Cryptol user to create formal algorithm specifications that closely imitate the style used to describe these algorithms mathematically.

Furthermore, Cryptol provides direct access and easily integrates with powerful tools such as SAT solvers and the Software Analysis Workbench (SAW). These tools allow the user to *prove* facts and demonstrate properties about their code which can provide assurance guarantees that go far beyond simple unit testing.

We will introduce some of these features below and discuss how they support building Cryptographic specifications and evaluations. If you have access to the Cryptol interpreter you can follow along with some of the examples, but a detailed introduction to the Cryptol interpreter will be introduced in future lessons.


## Basic Data Types

Cryptol was designed to provide easy access to the sorts of data and operations that appear in Cryptographic algorithms and specifications. There are five basic data types provided by Cryptol: bits, sequences, integers, tuples, and records. Cryptol also supports the ability to create user defined types built up from the basic types. In this section we present some basic examples deomonstrating these types, note that commands using `:t` are a request to Cryptol to report the type of argument.

 * **Bits** - The simplest data type which can take on two values, `True` and `False`. Common operations like `and`, `or`, and `not` are available.

```haskell
Cryptol> :t True
True : Bit
Cryptol> True && False
False
Cryptol> True /\ False
False
Cryptol> True || False
True
Cryptol> True \/ False
True
Cryptol> ~True
False
```

 * **Sequences** - Finite lists of objects all of the same data type. Common cryptographic algorithms make use of *words* or *registers* which are one dimensional sequences of Bits. Cryptol seemlessly handles operations on words of arbitrary sizes and also allows for multi-dimensional sequences (*i.e.* sequences of words or sequences of sequences of words).

```haskell
Cryptol> let s1 = [True, True, False, True]
Cryptol> :t s1
s1 : [4]
Cryptol> let s2 = [0x1f, 0x11, 0x03, 0xd5]
Cryptol> :t s2
s2 : [4][8]
Cryptol> let s3 = [[0x1, 0x2], [0x3, 0x4], [0x5, 0x6]]
Cryptol> :t s3
s3 : [3][2][4]
```
 
 * **Integers** - An arbitrary precision integer type.

```haskell
Cryptol> 1+1
Showing a specific instance of polymorphic result:
  * Using 'Integer' for type argument 'a' of '(Cryptol::+)'
2
Cryptol> 42*314
Showing a specific instance of polymorphic result:
  * Using 'Integer' for type argument 'a' of '(Cryptol::*)'
13188
Cryptol> 123456789234567890+234567890123456789
Showing a specific instance of polymorphic result:
  * Using 'Integer' for type argument 'a' of '(Cryptol::+)'
358024679358024679
```

 * **Tuples** - Tuples support heterogenous collections. Members are accessed with the dot (`.`) operator and are zero indexed.
```haskell
Cryptol> let tup = (1 : Integer, 0x02, [0x31, 0x32])
Cryptol> :t tup
tup : (Integer, [8], [2][8])
Cryptol> tup.0
1
Cryptol> tup.1
0x02
Cryptol> tup.2
[0x31, 0x32]
```
 * **Records** - Records allow for more complex data structures to be formed, where fields may be accessed by their tnames.
 
```haskell
Cryptol> let point = {x = 10:[32], y = 25:[32]}
Cryptol> point.x
0x0000000a
Cryptol> point.y
0x00000019
```

## Operators

Cryptol provides a collection of built in operators to build expressions and perform computations. Some familiar operators which appear in cryptographic applications include:

* `~` -- bitwise inversion
* `&&`, `||`, `^` -- bitwise logical operations
* `==`, `!=` -- structural comparison
* `>=`, `>`, `<=`, `<` -- nonnegative word comparisons
* `+`, `-`, `*`, `/`, `%`, `**` -- wordwise modular arithmetic
* `>>`, `<<`, `>>>`, `<<<` -- shifts and rotates
* `#` concatenation

There are many more. A list of the currently defined symbols and operators are available by running `:browse` in the interpreter.

An interesting feature of Cryptol's type system is that you can check the type of an operator with the `:t` command in the interpreter just as you can for data:

```haskell
Cryptol> :t (&&)
(&&) : {a} (Logic a) => a -> a -> a
```

In a nutshell, this indicates that the bitwise and operater (`&&`) operates on two elements of type `a` that are in the "`Logic`" typeclass and returns another element of the same type.

## Primitives

## Functions

## Functions and Laziness

## Enumerations

## Sequence Comprehensions

## Control Structures

# References