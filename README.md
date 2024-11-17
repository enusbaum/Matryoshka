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
    - `"g"`: GZip compression (Good)
    - `"b"`: Brotli compression (Better)
    - `null` or omitted if no compression is used.
  - **Purpose:**
    - Reduces the size of the nested tokens.
    - Addresses concerns about token size and performance impact.
    - Compression is only recommended to help alleviate any compatibility issues with applications handling large HTTP header values
    - Because the compressed results are Base64 encoded, only `auth_stack` containers with 3 or more entries should be compressed as shorter entries could result in a longer result than the uncompressed value
   
- **`hash`**:
  - A HMAC-SHA256 hash of the `container` value.
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
 
Implementation of compression is only recommended in order to maintain compatibility with applications that run into issues with large HTTP headers. While the compression ratio increases as the 
size of the uncompressed token increases, the added compute overhead will negate any performance offsets in transmission times. The below table shows simulated examples of compression ratios for 
JWT tokens implementing the Matryoshka pattern at given depths:

|Depth|Uncompressed Size|GZip+Base64|Brotli+Base64
|--|--|--|--|
|1|529|572 (1.08x)|556 (1.05x)
|2|1163|1132 (0.97x)|1120 (0.96x)
|3|1903|1884 (0.99x)|1868 (0.98x)
|4|2965|2768 (0.93x)|2748 (0.92x)
|5|4288|3944 (0.91x)|3957 (0.92x)
|6|6105|5496 (0.90x)|5468 (0.89x)
|7|8528|7452 (0.87x)|7256 (0.85x)
|8|11759|10076 (0.85x)|9604 (0.81x)

Note: Enabling HTTP Compression on your applciation endpoint will not compress the JWT Token, as HTTP compression only appplies to the payload body and not the headers.

- **Encryption (JWE):**

  - **Purpose:** Protect sensitive data within tokens, especially when crossing organizational boundaries.
  - **Methods:** Use JSON Web Encryption standards.
  - **Implementation:** Encrypted tokens are indicated by the `fmt` field set to `JWE`.

---

## Examples

(All examples use the shared secret `example` for their HMAC-SHA256 signatures and hash)

### Simple JWT with One nested Token

JWT:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyLCJhdXRoX3N0YWNrIjp7ImZtdCI6Imp3dCIsImhhc2giOiI0MGJhMTk4ODg3YzIyM2I4YmIzYzY1MTg3ZDYxMDgwMjFiMDQ4MjAyNWYxOTMyMzVhMjk3Yjk3MzVhMGNmMDE5IiwiZGVwdGgiOjEsImNvbnRhaW5lciI6ImV5SmhiR2NpT2lKSVV6STFOaUlzSW5SNWNDSTZJa3BYVkNKOS5leUp6ZFdJaU9pSXhNak0wTlRZM09Ea3dJaXdpYm1GdFpTSTZJa3B2YUc0Z1JHOWxJaXdpYVdGMElqb3hOVEUyTWpNNU1ESXlmUS5pVUNST0h0NkpIQU5kdHpUNmFPdVVnT3FWRlJhbE9XMjBTYnpSc241U2tJIn19.ud_wwYeKCxj5AW5sRSopMBT2h-oZCnqT-BfLlgjHlmQ
```

JSON:
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022,
  "auth_stack": {
    "fmt": "jwt",
    "hash": "40ba198887c223b8bb3c65187d6108021b0482025f193235a297b9735a0cf019",
    "sid": "Organization.Services.ServiceA"
    "depth": 1,
    "container": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.iUCROHt6JHANdtzT6aOuUgOqVFRalOW20SbzRsn5SkI" 
   }
}
```

In the above example, a client (`Client1`) has called `ServiceA` which has then generated a JWT token to send to a downstream service (`ServiceB`), implementing the Matryoshka JWT Pattern by embedding the original JWT token from `Client1` in the `auth_stack` claim.

### JWT with three nested JWT tokens, implementing compression

JWT:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIzLCJhdXRoX3N0YWNrIjp7ImZtdCI6Imp3dCIsImNtcCI6ImciLCJoYXNoIjoiZWU4OGJiNjc1M2QyMjViY2U5NWRiYmNmNjMzN2NhZDNmNzgyMThjZmI1NjMxMzY0OGNhMDQ0M2NiYjZjMjYyOSIsInNpZCI6Ik9yZ2FuaXphdGlvbi5TZXJ2aWNlcy5TZXJ2aWNlQyIsImRlcHRoIjozLCJjb250YWluZXIiOiJINHNJQUFBQUFBQUFBMVZTeVphYk9CVDlvdVFBTnBYMnNsMGdESTdrc3RDRWRtSklBWkpzWE1iRjhQVXRWM0s2VDYra045OTczMnVXckMyVHFqdDFXVXJYMUVkZGVrOHZPS3hlMDVkVUQ0SzlacnZ2elpLdE5VOWRVanJESG5xSUZKdFRwS2UwbTdyU2dsSG1YOG1mS3RtKzQyUm5ubjdGZ1pmMjF4bVJPSEExSVl6UzllZHIxdFlDWDhVR2VRVkhIMmsvL0VpdEhPdm5NRHRzM0h0UExScS9odHVxYy9uWFFxQ3I2OU5KRGowWXdma1VJWXZzMlplUnRDZ3BKdFMvYjArY0xvZ3pMUWxjQ3lKYjJOTVpCZWxjckhwR050NGdJZzFhWlNjVDZxR0l6akQ2ZXlsNHVwWFBlUmMweUMreXUwVUc0S0hFME5hSitTeTdrRWlSQllvalV5My8vYzlQZ1N3MjFRRS9jUzBPWTE4R29lTmpIbEk4TmNwTVE0ZHJvZXRlMGQyUWE4TWFyV2RpVFZaNTVrRjcxcDg5SGVSOG5EaVFVZDVyVjJlT2pjVWk1K1pOVWJNbGZFQXdEaWtuNkEwbjQ4YjVONHBuSTc1Z1JmL1UxaFljVUZMbmVEWDMzM0VnY0wvUGxOMXRpY0VBYXY5R0NFTTROaThrUmtlVjRJTGFYUUc5N2NSWmZhcTB1VlZSblpWczhHU01NcVROV0VWSW56Mno1dHkvcWdRNlBaNTQ5RVJvZlZTeFQ5V0tIWjVxNVdMSUd1b3RPY0d5cE9iRjJSSlNuOG9WS1F3Y2Zvb3Z0ZWZmc0RVSWc5aHpmUFlORFFVWDdSdUwvWkNJZ2J0K043VWlxZXg5L1dNZmlQVWRYdXIvNWdPWE1obWZmQVlXb1dOcGpFQmliMHBhVEFXcndwck9kMkxudmpURHhFbkdHLzEvKzZ6TlVBZEdZVjl1WkZ4ZjNDNXVXS0NzcEdGUTJEQlhDUkNJejI1SFR6MTlqaGdhVldKeXFNT0pSZGs5OTkxRjlaS3dXTDRWak4wYmc0VExPVEV2ZEFpeHltMjhubzNoeWd0L0VoLzZaY0pZRlVtRkxUNFdqSVlWUUlXS3cxYXQrdzFMNXB4cmJ6cFIxc0xnR1c5MUJjd0lhYTN4WlUvLzdkZG5rbkU0OFI0YzgzZ25tbmhvZVlMZHZxVEduZy9LQStoZzREVFVqRFVNaXlvT1Q0d0FRUDMyWGpKR25EN2NqU1lzMmwvZC9YeFVzY0dNc0E4WnQ1UHpqOWpzTlRQeTRmUk1tTzg0MnJiRHhCUnVQNjhNbUlmRDNNUCtQRE9TZWU1dUg1RG9SOE5aU25QUE8ydDBsbjdzRllibEpXQjNUa0hHb2pweG5BQTNBNnBqZkNWSnV5Mk52TEZBYjNtZTdYNmR2KzgranAvTllxdXNlZTlQY042a2EzNVozci85QlE2aVNOOVoxRGF2MTdSc3Y4MHZ4VC9NWDNVSWl3UUFBQT09In19.DQRSDDZG2CUBNDwQxpaz79SFf2dDeGaUyw8HN9Aaqw4
```

JSON:
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239023,
  "auth_stack": {
    "fmt": "jwt",
    "cmp": "g",
    "hash": "ee88bb6753d225bce95dbbcf6337cad3f78218cfb56313648ca0443cbb6c2629",
    "sid": "Organization.Services.ServiceC",
    "depth": 3,
    "container": "H4sIAAAAAAAAA1VSyZabOBT9ouQANpX2sl0gDI7kstCEdmJIAZJsXMbF8PUtV3K6T6+kN99732uWrC2Tqjt1WUrX1Eddek8vOKxe05dUD4K9ZrvvzZKtNU9dUjrDHnqIFJtTpKe0m7rSglHmX8mfKtm+42Rnnn7FgZf21xmROHA1IYzS9edr1tYCX8UGeQVHH2k//EitHOvnMDts3HtPLRq/htuqc/nXQqCr69NJDj0YwfkUIYvs2ZeRtCgpJtS/b0+cLogzLQlcCyJb2NMZBelcrHpGNt4gIg1aZScT6qGIzjD6eyl4upXPeRc0yC+yu0UG4KHE0NaJ+Sy7kEiRBYojUy3//c9PgSw21QE/cS0OY18GoeNjHlI8NcpMQ4droete0d2Qa8MarWdiTVZ55kF71p89HeR8nDiQUd5rV2eOjcUi5+ZNUbMlfEAwDikn6A0n48b5N4pnI75gRf/U1hYcUFLneDX333EgcL/PlN1ticEAav9GCEM4Ni8kRkeV4ILaXQG97cRZfaq0uVVRnZVs8GSMMqTNWEVInz2z5ty/qgQ6PZ549ERofVSxT9WKHZ5q5WLIGuotOcGypObF2RJSn8oVKQwcfoovteffsDUIg9hzfPYNDQUX7RuL/ZCIgbt+N7Uiqex9/WMfiPUdXur/5gOXMhmffAYWoWNpjEBib0paTAWrwprOd2LnvjTDxEnGG/1/+6zNUAdGYV9uZFxf3C5uWKCspGFQ2DBXCRCIz25HTz19jhgaVWJyqMOJRdk9991F9ZKwWL4VjN0bg4TLOTEvdAixym28no3hygt/Eh/6ZcJYFUmFLT4WjIYVQIWKw1at+w1L5pxrbzpR1sLgGW91BcwIaa3xZU//7ddnknE48R4c83gnmnhoeYLdvqTGng/KA+hg4DTUjDUMiyoOT4wAQP32XjJGnD7cjSYs2l/d/XxUscGMsA8Zt5Pzj9jsNTPy4fRMmO842rbDxBRuP68MmIfD3MP+PDOSee5uH5DoR8NZSnPPO2t0ln7sFYblJWB3TkHGojpxnAA3A6pjfCVJuy2NvLFAb3me7X6dv+8+jp/NYqusee9PcN6ka35Z3r/9BQ6iSN9Z1Dav17Rsv80vxT/MX3UIiwQAAA==" 
   }
}
```

In the above example, we're now three layers deep in our call stack (`Client` -> `ServiceA` -> `ServiceB` -> `ServiceC`), and the above token has been generated by `ServiceC` to make calls further down the stack. The `container` property in the `auth_stack` payload has been compressed using the specified GZip compression and Base64 encoded.
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
  - Default Maximum Header Sizes within Nginx is 8k

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
