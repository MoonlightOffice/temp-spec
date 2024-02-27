# Digital Signature and Timestamp with JSON

Last modified at: 2024-2-27

Table of Contents
* [1. Introduction](#1) 
* [2. Enconding Rules](#2) 
* [3. Overview](#3) 
* [4. Version](#4) 
* [5. Payload](#5) 
* [6. Signatures](#6) 
	* [6.1 protected](#6.1) 
	* [6.2 signature](#6.2) 
* [7. Timestamp](#7) 
	* [7.1 Compute digest covering payload, signature, and timestamps](#7.1) 
	* [7.2 Send Digest to TSA](#7.2) 
		* [7.2.1 Default](#7.2.1) 
		* [7.2.2 RFC3161](#7.2.2) 

# 1. Introduction <a id="1"></a>
This specification lays out the guidelines for dealing with digital signatures and timestamps in JSON. While it draws on the key ideas from [JWS (RFC 7515)](https://datatracker.ietf.org/doc/html/rfc7515) and [JAdES (ETSI TS 119 182-1 V1.1.1)](https://www.etsi.org/deliver/etsi_ts/119100_119199/11918201/01.01.01_60/ts_11918201v010101p.pdf), this specification is designed to be more straightforward, with the goal of simplifying implementation in applications.

# 2. Enconding Rules <a id="2"></a>

* __Base64url__ - This encoding is similar to base64, but it's URL- and filename-safe and removes any trailing padding '='s. Check RFC 7515, Appendix C for more information. Base64url is used for most part of this spec.

* __Base64__ - Base64 standard encoding with trailing padding '='. This is used for encoding x509 certificates in DER format and a timestamp token in RFC 3161.

# 3. Overview <a id="3"></a>
The whole properties in JSON format follows this structure:

```json
{
	"version": "v1",
	"payload": "<base64url-encoded data to be signed>", // Present only when data is attached
	"signatures": [
		{
			"protected": "<signature 1's protected header>",
			"signature": "<signature 1>",
			"timestamps": [
				{
					"type": "Default",
					"timestamp": "<timestamp 1>"
				},
				// ...
				{
					"type": "RFC3161",
					"timestamp": "<timestamp N>"
				}
			]
		},
		// ...
		{
			"protected": "<signature N's protected header>",
			"signature": "<signature N>",
			"timestamps": [
				{
					"type": "RFC3161",
					"timestamp": "<timestamp 1>"
				},
				// ...
				{
					"type": "Default",
					"timestamp": "<timestamp N>"
				}
			]
		}
	]
}
```

# 4. Version <a id="4"></a>

The version of the specification. `v1` is the latest version at the time of writing. You can extend or customize this specification, for example by adding your own properties or including certificate revocation information, by changing the version to something like `example.com/v1`.

# 5. Payload <a id="5"></a>

The data to be signed in base64url. For instance, if the data you want to sign is `Hello, world!`, the payload field should be `{"payload": "SGVsbG8gd29ybGQ"}`. When the data is quite large, it is recommended that you detach the payload as a separate file. In that case, you should delete the field `payload` from the JSON file.

# 6. Signatures <a id="6"></a>

Each entry in the `signatures` list is an object consisting of a signature value and the parameters used to compute the signature value.

```json
{
	"protected": "<signature's protected header>",
	"signature": "<signature>",
	"timestamps": [
		// ...
	]
}
```

## 6.1 protected <a id="6.1"></a>

The field `protected` is a base64url-encoded string that, when decoded, yields a JSON object containing parameters of the signature as shown below:

```json
{
	"sigT": "<RFC3339-formatted signing time>",
	"x5c": [ 
		"<base64-encoded certificate 1>",
		"<base64-encoded certificate 2>"
	],
	"alg": "<sign and hash algorithms>",
	"metadata": {
		// <Arbitrary data in JSON format>
	}
}
```

- `sigT`: The signing time claimed by the signatory. The contents of the string shall meet the following requirements:
	* It Shall be formatted as specified in IETF RFC 3339.
	* It Shall be the UTC time for date and time.
	* It Shall not contain the part corresponding to fraction of seconds.
- `x5c`: An array or chain of base64-encoded (not base64url) certificates. The first certificate in the list is used to verify the signature, followed by its parent certificates. Further infromation is available in RFC7515 [2], clause 4.1.6.
- `metadata`: A JSON object containing additional arbitrary data provided by the signatory. If no data is provided, place an empty JSON object `{}` instead of `null`.
- `alg`: This field specifies the cryptographic algorithm used to secure the payload and protected header, taken from RFC 7518, clause 3.1, known as JSON Web Algorithms (JWA). Supported algorithms are listed below: 

| Algorithm | Description |
| ---- | ---- |
| RS256 | RSASSA-PKCS1-v1_5 using SHA-256 |
| RS384 | RSASSA-PKCS1-v1_5 using SHA-384 |
| RS512 | RSASSA-PKCS1-v1_5 using SHA-512 |
| PS256 | RSASSA-PSS using SHA-256 and MGF1 with SHA-256 |
| PS384 | RSASSA-PSS using SHA-384 and MGF1 with SHA-384 |
| PS512 | RSASSA-PSS using SHA-512 and MGF1 with SHA-512 |
| ES256 | ECDSA using P-256 and SHA-256 |
| ES384 | ECDSA using P-384 and SHA-384 |
| ES512 | ECDSA using P-521 and SHA-512 |

Example of protected header would look like this:
```json
{
	"sigT": "2019-11-19T17:28:15Z",
	"x5c":[
		"MIIE3j...(cert1)...WVU+4=",
		"MIIE+z...(cert2)...9VZw==",
		"MIIC5z...(cert3)...ViH0Pd"
	],
	"alg": "ES256",
	"metadata": {
		"example.com": {
			"signatory": "S.John",
			"email": "sample@example.com"
		}
	}
}
```

## 6.2 signature <a id="6.2"></a>

The field `signature` is a base64url-encoded signature value. Computation of the signature is done according to the following steps.

### Step 1. Check what algorithm to use
The algorithm specified in field `alg` is used throught the process of signing & verifying the signature. For example, if `alg` is ES512, SHA-512 is used to generate a digest of data, and ECDSA using P-521 curve is used to sign & verify the signature.

### Step 2. Construct the message imprint

Concatenate `protected` with `payload` using `.` as a separator. If the payload is detached (available as a raw data in separate file), encode it in base64url before concatenating it with the message imprint.

```plain
<protected>.<payload>

# Example of message imprint
eyJzaWdUIj···S5hYSJdfX0.MV9b23bQeM···Gk0W_yUx1iU7dM
```
<br/>

### Step 3. Compute the digest
Compute the digest over the concatenated string above. The digest that is generated in this way __MUST BE IN BINARY FORMAT__ (not base64url-encoded).

### Step 4. Compute or verify the signature
* (Signatory) Sign over the digest computed in Step 3 using the signatory's private key. The generated signature must be base64url-encoded and saved in the field `signature`. 

* (Verifier) Decode the base64url-encoded signature in the field `signature`. Then, use the decoded signature, the digest computed in Step 3, and the first certificate in `x5c` to verify the signature.


# 7. Timestamps <a id="7"></a>

Each entry in the `timestamps` list is an object consisting of a timestamp token and its type. Different types of timestamps from different TSAs are supported. The order of entries in the list is chronological, meaning that a new timestamp is added to the end of the list. Each timestamp proves the existence of the payload, signatures, and all other previous timestamps at the time of timestamping, guaranteed by the issuer of the timestamp.

```json
{
	"protected": "<signature K's protected header>",
	"signature": "<signature K>",
	"timestamps": [
		{
			"type": "Default",
			"timestamp": "<timestamp 1>"
		},
		// ...
		{
			"type": "RFC3161",
			"timestamp": "<timestamp N>"
		}
	]
}
```

The signing & verification of each timestamp is done in two parts. First is to construct a message imprint and compute its digest. Second is to compute a timestamp over the digest.

## 7.1 Compute digest covering payload, signature, and timestamps <a id="7.1"></a>

This part computes a digest of the message imprint covering the payload, signature, and all previous timestamps. The digest computation is done according to the following steps.

### Step 1. Choose which hash algorithm to use

Choose the hash algorithm from the list below to compute the digest.

* SHA-256
* SHA-384
* SHA-512

### Step 2. Construct the message imprint

Concatenate the following values in the following order using `.` as a separator. If the payload is detached (available as a raw data in separate file), encode it in base64url before concatenating it with the message imprint.

* payload
* signature K's protected header
* signature K
* timestamp 1
...
* timestamp N


```plain
<payload>.<protected>.<signature K>.<timestamp 1>.<...>.<timestamp N>
```

<br/>

### Step 3. Compute the digest

Compute the digest over the concatenated string above. The digest that is generated in this way __MUST BE IN BINARY FORMAT__ (not base64url-encoded).

That's all for computing the digest.

## 7.2 Send Digest to TSA <a id="7.2"></a>

The digest and hash algorithm computed in 7.1 is sent to the TSA to obtain a timestamp token. Two types of TSA are currently supported.

* Default
* RFC3161

## 7.2.1 Default <a id="7.2.1"></a>

The type `Default` defines timestamping rules which is designed to be simple. Its main purpose is for internal use, but it is also expected to be used as a public protocol in the future.

The following steps demonstrate how to create a timestamp token for a digest given by a client using a specified hash algorithm.

### Step 1. Send digest and hash algorithm to TSA

The client sends the hash algorithm and the digest computed in 7.1 to the TSA. The identifier of the hash algorithm must be specified in the following way:

| Identifier | Hash Algorithm |
| --- | --- |
| S256 | SHA-256 |
| S384 | SHA-384 |
| S512 | SHA-512 |

The digest computed in the previous step is in binary format. Encode it in base64url to get the string representation of it before sending to a TSA. Note that the client must use the hash algorithm the TSA supports.

### Step 2. Construct protected header

Here's the structure of the protected header. It is the same as that of the signature you have seen in this specification.

```json
{
	"sigT": "<RFC3339-formatted signing time>",
	"x5c": [ 
		"<base64-encoded tsa certificate 1>",
		"<base64-encoded tsa certificate 2>"
	],
	"alg": "<sign and hash algorithms>",
	"metadata": {
		// <Arbitrary data in JSON format>
	}
}
```

The TSA records the time, its certificate chain, and the algorithms used for digest computation at the client side and the timestamp computation at the TSA side.  The signing algorithm must follow the hash algorithm used at the client side. For example, if the digest and its algorithm sent by a client is `S384`, the TSA must use either `RS384`, `PS384`, or `ES384`.

Encode the protected header in base64url.

### Step 3. Construct the message imprint

Concatenate the following values in the following order using `.` as a separator.

* `base64url-encoded protected header`
* `base64url-encoded digest sent by client`

```plain
<base64url-encoded protected header>.<base64url-encoded digest sent by client>
```

<br/>

### Step 4. Compute the digest

Compute the digest over the concatenated string above. The digest that is generated in this way __MUST BE IN BINARY FORMAT__ (not base64url-encoded).

### Step 5. Compute or verify the timestamp

* (TSA) Sign over the digest computed in Step 4 using the TSA's private key. The generated timestamp must be base64url-encoded. Then, return the timestamp token, which is the concatenation of its protected header and the timestamp as shown below, to the client: 

```plain
<base64url-encoded protected header>.<base64url-encoded timestamp>
```

The client is expected to add this timestamp token to the `timestamps` list.

* (Verifier) Split the timestamp token into `base64url-encoded protected header` and `base64url-encoded timestamp` with `.` as a separator. Decode `base64url-encoded timestamp` to extract its binary representation. Then, use the timestamp in binary format, the digest computed in the previous step, and the first certificate in `x5c` to verify the timestamp.


## 7.2.2 RFC3161 <a id="7.2.2"></a>

Refer to RFC 3161 for more details.
