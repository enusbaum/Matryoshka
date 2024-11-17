# Matryoshka JWT Pattern

An innovative approach to enhancing authentication, authorization, and tracing in microservices architectures using nested JWTs.

---

## Table of Contents

- [Introduction](#introduction)
- [Motivation](#motivation)
- [Problems Addressed](#problems-addressed)
- [Solution Overview](#solution-overview)
- [Key Concepts](#key-concepts)
  - [Auth Stack](#auth-stack)
  - [Token Structure](#token-structure)
  - [Preventing Recursion](#preventing-recursion)
  - [Compression and Encryption](#compression-and-encryption)
- [Implementation Guidelines](#implementation-guidelines)
  - [Prerequisites](#prerequisites)
  - [Token Generation](#token-generation)
  - [Token Processing](#token-processing)
  - [Middleware Extension](#middleware-extension)
- [Security Considerations](#security-considerations)
- [Performance Considerations](#performance-considerations)
- [Limitations](#limitations)
- [Future Work](#future-work)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)

---

## Introduction

The **Matryoshka JWT Pattern** is a novel approach to managing authentication, authorization, and tracing in microservices architectures. It involves nesting JSON Web Tokens (JWTs) within JWTs, much like Russian Matryoshka dolls, to create an authentication stack that can be passed along service calls.

This pattern enhances visibility into the call chain, improves security by verifying the entire chain of calls, aids in debugging by providing a built-in call stack within the tokens themselves, and helps prevent recursive or circular service calls.

---

## Motivation

In complex microservices architectures, services often call other services, leading to deep and sometimes circular call chains. Traditional methods may lack the ability to:

- Trace these calls effectively.
- Enforce security policies based on the entire call chain.
- Prevent recursion and circular dependencies.
- Provide services with context about their position in the call chain.

The Matryoshka JWT Pattern addresses these challenges by embedding the call history within the tokens themselves.

---

## Problems Addressed

1. **Lack of Call Chain Visibility:**

   - **Problem:** Services are unaware of how deep they are in a call chain or which services have been called before them.
   - **Solution:** The pattern embeds an authentication stack within the JWT, allowing services to see the entire call chain up to that point.

2. **Difficulty in Debugging and Monitoring:**

   - **Problem:** Tracing issues in microservices can be challenging without clear visibility into service interactions.
   - **Solution:** By providing a built-in call stack, the pattern facilitates easier debugging and monitoring of service calls.

3. **Unauthorized Service Access:**

   - **Problem:** Services may inadvertently call or be called by unauthorized services.
   - **Solution:** The authentication stack allows services to verify the entire chain of callers, ensuring that only authorized services are involved.

4. **Circular Dependencies and Recursion:**

   - **Problem:** Circular service calls can lead to infinite loops, stack overflows, or performance degradation.
   - **Solution:** The pattern enables detection of recursion by inspecting the call chain for repeats and limits the call depth to prevent excessive recursion.

5. **Inefficient Handling of Deep Call Chains:**

   - **Problem:** As call chains deepen, tokens may become large, and performance may suffer.
   - **Solution:** Compression and encryption mechanisms reduce token sizes and protect sensitive information.

---

## Solution Overview

The Matryoshka JWT Pattern proposes that each service, upon receiving a request, generates a new JWT that includes a custom claim called `auth_token`. This claim contains the previous token (from the caller), forming a chain of tokens representing the call stack.

Key features of the solution include:

- **Embedding the Call Chain:** Nesting tokens within tokens to build an authentication stack.
- **Enhanced Security:** Verifying the entire call chain to enforce security policies.
- **Recursion Prevention:** Detecting and preventing circular calls by analyzing the call chain.
- **Compression and Encryption:** Using compression to reduce token size and encryption to protect sensitive data.

---

## Key Concepts

### Auth Stack

The **Auth Stack** is the chain of nested JWTs representing the sequence of service calls. Each token in the stack contains metadata about the previous call, allowing services to trace back through the call chain and make informed decisions based on the entire sequence of calls.

### Token Structure

The `auth_stack` claim contains a JSON object with the following fields:

#### Required Fields

- **`fmt`** (Format):
  - Indicates the format of the token contained within the `container` field.
  - Possible values:
    - `"JWT"`: Standard JSON Web Token.
    - `"JWE"`: JSON Web Encryption token (encrypted JWT).
  - **Purpose:**
    - Allows services to know how to process the nested token.
    - Enables the use of encrypted tokens (JWEs) for added security, especially across organizational boundaries.
    - If a service cannot decrypt the JWE, it processes as much of the auth stack as it can, maintaining security boundaries.

- **`cmp`** (Compression):
  - Specifies the compression algorithm used on the token container.
  - Possible values:
    - `"g"`: GZip compression has been used on the `container`
    - `null` or omitted if no compression is used.
  - **Purpose:**
    - Reduces the size of the nested tokens.
    - Addresses concerns about token size and performance impact.

- **`hash`**:
  - A SHA256 checksum of the `container` value.
  - **Purpose:**
    - Ensures the integrity of the nested token.
    - Allows the receiving service to verify that the token has not been tampered with or corrupted.

- **`container`**:
  - Contains the actual nested token (JWT or JWE), which itself may have an `auth_stack` claim.
  - If `cmp` is specified, contents will be a base64-encoded representation of the actual nested token (JWT or JWE), and will need to be decompressed first.
  - **Purpose:**
    - Holds the previous JWT/JWE in the call chain.
    - By recursively including the `auth_stack` in each token, the call stack is built.

#### Optional Fields
- **`sid`** (Service ID):
  - Specifies the Service ID of the service that signed the current parent JWT, and is the top of the stack for the `auth_stack`
  - This can be an internal organizational identifier or a human readable namespace
  - **Purpose**:
    - Allows identification of services that are part of the `auth_stack`
    - Can be used to restrict access to services if a service in the chain is not permitted (crossing permissions boundaries)

- **`depth`**:
  - Indicates the current `auth_stack` depth at the current point.
  - **Purpose:**:
    - Explicitly states the `auth_stack` depth at the given point
    - This is an optional, informational field for Services to provide to downstream consumers who lack access to decrypt the `auth_stack` token secured with JWE
    - Because this can potentially reveal information about your organizations service architecture and topography, it is not recommended for use beyond debugging

### Preventing Recursion

To prevent recursion and circular dependencies:

- **Call Chain Inspection:**
  - Each service inspects the `auth_stack` chain to identify if it has already been called in the current request flow.
  - If the service's identifier appears in the call chain, it can abort the call to prevent recursion.

- **Maximum Call Depth:**
  - Services can enforce a maximum call depth by checking the length of the `auth_stack` chain.
  - If the call depth exceeds a predefined limit, the service can deny the request to prevent excessive recursion.

- **Unique Identifiers:**
  - Including unique service identifiers in tokens helps in detecting repeats in the call chain.

- **Policy Enforcement:**
  - Organizations can define policies to handle recursion detection, such as logging occurrences, sending alerts, or blocking the request.

### Compression and Encryption

- **Compression:**

  - **Purpose:** Reduce token size to improve performance.
  - **Methods:** Use algorithms like gzip.
  - **Implementation:** Compressed tokens are base64-encoded and indicated by the `cmp` field.

- **Encryption (JWE):**

  - **Purpose:** Protect sensitive data within tokens, especially when crossing organizational boundaries.
  - **Methods:** Use JSON Web Encryption standards.
  - **Implementation:** Encrypted tokens are indicated by the `fmt` field set to `JWE`.

---

## Implementation Guidelines

### Prerequisites

- **Technology Stack:**
  - Modern Web Frameworks capable of processing JWT and JWE Tokens
  - .NET 8 or later (for example prototype implementation).
  - Familiarity with JWT, JWE, and microservices architectures.

- **Infrastructure:**
  - Secure key management system for handling encryption keys.
  - Shared trust framework among services for token validation.

### Token Generation

- **Create Initial Token:**

  - Service generates a JWT with standard claims and any necessary custom claims.
  - No `auth_stack` claim is included if it's the initial token.

- **Nesting Tokens:**

  - When a service calls another service, it generates a new JWT.
  - The previous token is embedded in the `auth_token` claim of the new token.
  - Apply compression and/or encryption as needed.

- **Metadata Inclusion:**

  - Include necessary metadata in the `auth_stack` claim fields (`fmt`, `cmp`, `hash`).

### Token Processing

- **Token Validation:**

  - Upon receiving a token, a service validates it according to standard JWT validation procedures.
  - Verify signatures, issuer, audience, expiration, and other claims.

- **Extracting the Auth Stack:**

  - Parse the `auth_stack` claim to retrieve the nested token.
  - Decompress and decrypt the `container` as indicated by the `cmp` and `fmt` fields.
  - Recursively process nested tokens to build the full call chain.

- **Recursion Detection:**

  - Inspect the call chain for repeats of service identifiers.
  - Compare the current service's identifier with those in the call chain.
  - If recursion is detected, handle according to policy (e.g., deny the request).

- **Call Depth Limitation:**

  - Determine the depth of the call chain by counting the nested tokens.
  - If the call depth exceeds the maximum allowed, deny the request.

### Middleware Extension

- **Automated Processing:**

  - Implement middleware components to automate token processing.
  - Middleware handles extraction, validation, and parsing of the `auth_stack` claim.

- **Integration with Frameworks:**

  - Extend existing authentication and authorization frameworks to support the pattern.
  - Ensure compatibility with existing services by ignoring the `auth_stack` claim if not implemented.

---

## Security Considerations

- **Key Management:**

  - Securely store and manage encryption keys.
  - Use key management services (e.g., AWS KMS, Azure Key Vault).
  - Implement key rotation policies.

- **Token Integrity:**

  - Verify the `hash` of the `container` to ensure data integrity.
  - Use strong hashing algorithms like SHA256.

- **Encryption Algorithms:**

  - Use robust encryption algorithms for JWEs (e.g., RSA-OAEP, AES256-GCM).
  - Ensure that only intended recipients can decrypt the tokens.

- **Access Control:**

  - Limit the information included in tokens to what is necessary.
  - Avoid including sensitive data unless absolutely required.

- **Cross-Boundary Security:**

  - When crossing organizational boundaries, use JWEs to protect internal tokens.
  - Ensure that external services cannot access internal call chain details unless authorized.

---

## Performance Considerations

- **Token Size:**

  - Monitor token sizes to prevent exceeding header limits.
  - Use compression to mitigate token size growth.

- **Processing Overhead:**

  - Be aware of the CPU overhead introduced by compression and encryption.
  - Optimize performance by caching public keys and using efficient algorithms.

- **Network Latency:**

  - Larger tokens may impact network transmission times.
  - Compression helps reduce latency caused by increased token sizes.

- **Scalability:**

  - Test the pattern under load to assess scalability.
  - Ensure that services can handle the additional processing without significant degradation.

---

## Limitations

- **Complexity:**

  - Increased complexity in token generation and processing.
  - Requires careful implementation and thorough testing.

- **Compatibility:**

  - May not be compatible with all JWT libraries or tools.
  - Custom parsing logic may be necessary.

- **Performance Impact:**

  - Potential performance impacts due to processing overhead.
  - Needs optimization to minimize impact.

- **Key Distribution:**

  - Requires secure distribution and management of public keys across services.
  - Key compromise can have widespread effects.

---

## Future Work

- **Tooling Support:**

  - Develop libraries or extensions to simplify implementation.
  - Create middleware packages for popular frameworks.

- **Standardization:**

  - Propose the pattern as a standard or contribute to existing standards.
  - Engage with the developer community for feedback and adoption.

- **Integration:**

  - Explore integration with service meshes or other architectural patterns.
  - Leverage existing infrastructure for authentication and tracing.

- **Security Enhancements:**

  - Implement additional security features and conduct regular audits.
  - Stay updated with the latest security best practices.

---

## Contributing

Contributions are welcome! Please open issues or pull requests for enhancements, bug fixes, or suggestions.

1. **Fork the Repository:**

   - Click the "Fork" button at the top right of the repository page.

2. **Create Your Feature Branch:**

   - `git checkout -b feature/YourFeature`

3. **Commit Your Changes:**

   - `git commit -m 'Add some feature'`

4. **Push to the Branch:**

   - `git push origin feature/YourFeature`

5. **Open a Pull Request:**

   - Navigate to your forked repository and click "New Pull Request".

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## Acknowledgments

- **Inspiration:**

  - Derived from challenges faced in microservices architectures regarding authentication and tracing.

- **Community Support:**

  - Contributions and feedback from the developer community.

- **Technologies:**

  - Built upon the capabilities of JWT, JWE, and modern cryptographic practices.

---

**Disclaimer**: This pattern is experimental and should be thoroughly tested and reviewed before use in production environments. Security considerations are critical when handling authentication tokens.

---
