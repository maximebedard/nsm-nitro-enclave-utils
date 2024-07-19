# Overview
`nsm-nitro-enclave-utils` is designed to make the lives of teams building with AWS Nitro Enclaves a bit easier. 
It's primary purpose is to support "bring your own PKI" as an option for attestation documents in development environments, allowing you to swap out the root of trust in clients that are verifying the attestation document's certificate chain.
With the root of trust swapped, your enclave services can dynamically generate attestation documents outside a Nitro Enclave. Clients can have _their_ root of trust swapped to successfully perform attestation against the attestation documents signed by your certificate. 

Replacing the root of trust inherently destroys the security guarantees of AWS Nitro Enclaves: it is up to your team to ensure `nsm-nitro-enclave-utils` is not misconfigured outside a development environment.

The api of `nsm-nitro-enclave-utils` is intentionally designed to mimic `aws-nitro-enclaves-nsm-api` in an effort to make adopting it a painless process.

⚠️ This crate is functional but its API is changing rapidly until further notice.

### Why "bring your own PKI"?
AWS Nitro Enclaves cryptographic attestation is a powerful tool that is accompanied by a less-than-desirable hurdle: ["You can request an enclave's attestation document from inside the enclave only"](https://docs.aws.amazon.com/enclaves/latest/user/set-up-attestation.html). This limitation introduces two big problems for development teams:
1. You need infra to start building with Nitro Enclaves, and [there are currently no free EC2 instances that support the AWS Nitro System](https://docs.aws.amazon.com/ec2/latest/instancetypes/ec2-nitro-instances.html).
2. Working locally means you either have to disable large parts of your system and/or meticulously mock attestation documents by "extracting" them from a deployed service; the latter of which often requiring the extraction of private key material.

Both of these aren't great. While a funded team may be able to afford to provision new infra during the earliest stages of development, this is a barrier for many who wish to play around with the principles of the technology. Furthermore, once you start building something serious with Nitro Enclaves, the need to address your development environments grows into its own little mountain of tech debt.

When you "bring your own PKI", you can tell `NsmBuilder` to use _your_ signing key instead of AWS's signing key. Now your services can dynamically request new attestation documents outside a Nitro Enclave. Any client that needs to perform attestation against these documents just needs its root of trust to be that of your signing key. All of this is supported by `nsm-nitro-enclave-utils`.

## Setup
If you're already using `aws-nitro-enclaves-nsm-api`, you'll need to swap out `aws_nitro_enclaves_nsm_api::driver::nsm_init` with `NsmBuilder`, which allows you to swap our your pki and specify the PCRs of your attestation document.

### Root of trust

#### Creating your own
This crate comes with a script to make generating your own pki easier. These scripts are used to generate the scripts in `data/certs`, which **shouldn't** be used as your root of trust, but you can generate your own with them! You can use`data/cert_chain.sh` to get a root, intermediate, and end cert.

#### AWS Root Certificate
When verifying an attestation document coming from a Nitro Enclave, you'll need to use AWS's root certificate; which can be downloaded from their documentation: https://docs.aws.amazon.com/enclaves/latest/user/verify-root.html#validation-process

### Examples

Coming soon 👷


## Wasm Compatability

`nsm-nitro-enclave-utils` provides WebAssembly support by disabling the `nitro` feature flag. When `nitro` is disabled, you can still sign your own attestation documents, and verify any attestation document (including authentic ones!), but you cannot generate authentic documents due to a lack of wasm support in `aws-nitro-enclaves-nsm-api`.

### Test coverage

There is a `wasm-pack` test harness in place to ensure features remain wasm compatible. This can be run with the following command: `wasm-pack test --node --no-default-features --features seed,rand`

### Feature support

| Feature                | Wasm Support |                                                                                    Description                                                                                     |
|:-----------------------|:------------:|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|
| Verifying              |      ✅       |                                                             Verify self signed, and AWS signed, attestation documents                                                              |
| Signing                |      ✅       |                                                                Sign attestation documents with "Bring your own PKI"                                                                |
| Authentic NSM requests |      ❌       | Due to limitations with `aws-nitro-enclaves-nsm-api`, requests to an authentic Nitro Secure Module don't have WebAssembly support. Disabling the `nitro` feature flag is required. |
| Seeded PCRs            |      ✅       |                                 Use any collection of strings to deterministically seed your PCRs with `Pcrs::seed`. Feature flag `seed` required.                                 |
| Random PCRs            |      ✅       |                     Don't care about the value of your PCRs, but don't want them to be all zeros? `Pcrs::rand` has you covered. Feature flag `rand` required.                      |