# ORIGINAL.zip File Specification

Last modified at: 2024-1-19

# Introduction
ORIGINAL.zip is designed to encapsulate data, signatures, and timestamps into a single file. It enables multiple signatories to sign a set of data with provisions for timestamping those signatures.

# Enconding Rules

* __base64url__ - This encoding is similar to base64, but it's URL- and filename-safe and removes any trailing padding '='s. Check RFC 7515, Appendix C for more information. base64url is used for most part of this spec.

* __base64__ - Base64 standard encoding with trailing padding '='. This is used only for encoding x509 certificates in DER format.

* __hex__ - The digest value must be hex-encoded for inclusion in a message imprint, which facilitates easy concatenation with other message imprints in string format. However, the digest data to be signed must be a decoded binary data during the signing process.

# File Structure
By unzipping ORIGINAL.zip, you get the following two files:

- `data.zip`: A zip archive containing all the files that the signatories wish to sign.
- `seal.json`: A JSON file containing signatures from all signatories and timestamps.

## data.zip
This archive holds the files designated for signing. For instance, if signatories intend to sign `sample.txt` and `another.mp4`, these are zipped into `data.zip` which they sign.

## seal.json
`seal.json` follows this structure:

```json
{
	"version": "v1",
	"signatures": [
		"<base64url-encoded signature 1>",
		"<base64url-encoded signature 2>",
		"<base64url-encoded signature N>"
	],
	"timestamps": [
		"<base64url-encoded timestamp 1>",
		"<base64url-encoded timestamp 2>",
		"<base64url-encoded timestamp N>"
	]
}
```

# Version

`v1` is the latest version at the time of writing this spec.

---

# Signatures

Each signature in the `signatures` list is a base64url-encoded string. When decoded, it reveals the following JSON object:

```json
{
	"protected": "<base64url-encoded protected field>",
	"signature": "<base64url-encoded signature>"
}
```

The `protected` field is a base64url-encoded string that, when decoded, yields a JSON object containing metadata about the signature. The structure of the decoded `protected` field is as follows:

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

- `sigT`: The signing time claimed by the signatory in RFC3339 format. The time is computed on the signatory's computer, thus other signatories are advised to verify its accuracy when receiving the signature.
- `x5c`: An array or chain of base64-encoded (not base64url) certificates. The first certificate in the list is used to verify the signature, followed by its parent certificates. Further infromation is available in RFC7515 [2], clause 4.1.6.
- `metadata`: A JSON object containing additional arbitrary data provided by the signatory. If no data is provided, place an empty JSON object `{}` instead of `null`.
- `alg`: This field specifies the cryptographic algorithm used to secure `data.zip' and the protected header, taken from RFC 7518, clause 3.1, known as JSON Web Algorithms (JWA). Supported algorithms are listed below: 

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


## Sign & Verify Signature
The following steps show the process of signing & verifying the signature value in `signature` field.

### Step 1. Check what algorithm to use
The algorithm specified in `alg` field is used throught the process of signing & verifying the signature. For example, if `alg` is ES512, SHA-512 is used to generate a digest of data, and ECDSA using P-521 curve is used to sign & verify the signature.

### Step 2. Construct the message imprint
Concatenate the value in `protected` field and the __hex-encoded__ digest of `data.zip` using `.` as a separator, as shown below:

```shell
<base64url-encoded protected>.<hex-encoded digest of data.zip>

# Actual message imprint would look like this
eyJzaWdUIj···S5hYSJdfX0.c1527cd893···7a57f79421
```
<br/>

### Step 3. Compute the digest
Compute the digest over the concatenated string above. The digest that is generated in this way __MUST BE in binary format__ (not hex-encoded).

### Step 4. Sign or verify the signature
* (Signatory) Sign over the digest computed in Step 3 using the signatory's private key. The generated signature must be base64url-encoded and saved in `signature` field. 

* (Verifier) Decode the base64url-encoded signature in the `signature` field. Then, use the decoded signature, the digest computed in Step 3, and the first certificate in `x5c` to verify the signature.

---

# Timestamps
The structure and its handling of each timestamp in the `timestamps` field of `seal.json` is identical to that of `signatures`. The only difference is the sign & verify process, especially `step 2`, where the message imprint is created.


### Step 2. Construct the message imprint (for timestamp)

Concatenate the __hex-encoded__ digest of `data.zip`, all signatures and current timestamps in `seal.json` using `.` as a separator, as shown below. Note that the signatures and timestamps must be placed according to the order in the list in `seal.json`.

```shell
<hex-encoded digest of data.zip>.<signature-1>.<signature-N>.<timestamp-1>.<timestamp-N>
```

Let's call this concatenated string as 'X'. Then, compute the digest over X. The digest of X __MUST BE hex-encoded__.

Now, create another imprint by concatenating the value in `protected` field and the digest of X as follows:

```shell
<protected>.<hex-encoded digest of X>
```

That's all for step 2. The rest of the steps are the same as in the signature section.

Once you have completed all the steps, you will get a timestamp that covers all the signatures and previous timestamps. Their validity period is extended and guaranteed to be valid until the expiry date of the new timestamp's certificate. Add the new timestamp to the `timestamps` list in `seal.json`.

---

# Additional Consideration

## Certificate Revocation
Recipients should verify the status of certificates upon receipt. If a certificate is found to be revoked, the corresponding signature should be rejected. ORIGINAL.zip does not require to include OCSP responses or CRLs, making it the recipient's responsibility to check the current status of certificates.
