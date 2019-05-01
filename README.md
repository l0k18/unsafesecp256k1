# secp256k1-go

~~This package provides bindings (using cgo) to the upstream [https://github.com/bitcoin-core/secp256k1](libsecp256k1) C library.~~

This package provides bindings to the 'unsafe' secp256k1 C library found here: [https://github.com/llamasoft/secp256k1_fast_unsafe](https://github.com/llamasoft/secp256k1_fast_unsafe).

You might be wondering, if you are not well versed in the important security characteristics of encryption libraries, why one would want an 'unsafe' version:

- Constant Time implementations of cryptographic primitives is very important for security in the use case of interactive sessions with encryption - the timing of various operations that can be provoked by an attacker can reveal information about the internal state and thus potentially lead to finding a way to breach the security and gain unauthorised access.

- Memory utilisation - most elliptic curve signature algorithms use some amount of pre-computation to accelerate scalar multiplication. Mostly the libraries are targeted to work adequately on a very low resource machine, very nearly embedded grade, ARM RISC processors and similar. But for running servers, if latency is an issue it is worth paying for more memory and using apps that make more use of more memory. So for performance critical EC signing performance, it makes a lot of sense to allow the programmer to enable a much larger precomp to reduce the time per operation and increase throughput and maybe lower latency a little.

- Most good encryption libraries have functions that sanitise secret data before freeing it. This is important for reducing the chances of malicious processes gaining access by searching newly freed memory.

### Use cases of ECDSA with relaxed security requirements

Obviously, removing the overhead of memory wiping, and constant time operations is out of the question if one is using the EC library to auth data for encrypted interactive sessions.

**However**, the case of the use of elliptic curves in cryptocurrency transaction authentication does not have several of the requirements for HMAC use of ECDSA:

- The crypto server is not running interactive services with untrusted peers when it is verifying transactions, and it produces very little useful information for an attacker aside from that maybe it can fingerprint the algorithm by sending crafted transactions. But one failure in a cryptocurrency network does not breach its security, and none of the data is secret anyway, except for the keys required to generate the signatures, which happens offline and not in interactive sessions.

- The crypto server, when providing data for miner workers, really should not be doing even one thing that is unnecessary. Constant time signature verifications is exactly NOT what you want for a cryptocurrency server. You want to rip through verification as quickly as possible so you are able to generate new block templates and/or relay the received transactions/blocks.

- Cryptocurrency servers often provide data to other applications like web wallets, explorers, payment processing, and the like. The operators of such services will also prefer to minimise the costs and processing delays whenever possible.

Because security relating to memory containing secrets is irrelevant for this case, and very often a cryptocurrency server is running alone inside an isolated virtualization environment where there is nearly no possibility of any other process seeing the memory, either inside or outside the container.

Further, when one wants optimal response time and throughput from a cryptocurrency server, every nanosecond you can shave from verification of signatures reduces the delay for mining and improves the network's latency in general.

So constant time, bigger memory use, and memory wiping are not important for verification of public transaction data, except maybe in as far as it may create a way to breach a cryptocurrency wallet connected to the server, but this is external to the transaction verification and witness signature verification. Faster is better, and there is no data in a full node that is secret anyway. Only wallets are sensitive.

It exposes several high level functions for elliptic curve operations over the secp256k1 curve, namely ECDSA, point & scalar operations, ECDH, and recoverable signatures. 

## Warning

It should be mentioned that the upstream library is still experimental
and has yet to be formally released. As such, you should think twice
before installing this package. 

The currently targeted version of libsecp256k1 is the latest master commit. 

Currently two experimental libraries are also included and supported: ECDH and 
signature recovery. These are included with the default installation, and
may eventually be discontinued by the same (as has happened with Schnorr). 

## Contributing

To start developing, clone the package from github, and from the
source directory, run the following to install the package.

    git submodule update --init
    make install
    
Tests can be run by calling `make test`
Coverage can be build by calling `make coverage`
To display a HTML code coverage report, call `make coverage-html`

Please make sure to include tests for new features.  

## Rationale behind API

There have been some slight changes to the API exposed by libsecp256k1. 
This section will document conventions adopted in the design. 

#### Always return error code from libsecp256k1
There are some functions which return more than one error code, indicating
the specific failure which occurred. With this in mind, the raw error
code is always returned as the first return value. 

To help provide some meaning to the error codes, the last parameter will
be used to return reasonable error messages.

#### Use write-by-reference where upstream uses it
In functions like EcPrivkeyTweakAdd, libsecp256k1 will take a pointer
to the private key, tweaking the value in place (overwriting the original value)

To avoid making copies of secrets in memory, we allow upstream to
overwrite the original values. If the to-be-written value is a new object,
it is returned with the other return values (example: EcdsaSign)
  