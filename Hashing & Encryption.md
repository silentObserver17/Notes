A Hashing Algorithm is a function that takes input(like a password) and converts it into fixed length string of characters, called hash value or digest.

It is a one way transformation:
- Input: `"myPassword123"`
- Output: `"5f4dcc3b5aa765d61d8327deb882cf99"` (example)

### Key properties of hashing algorithms
A good hashing algorithm has these characteristics:
- **Deterministic:** Same input -> Same output every time.
- **Fixed Output length:** No matter how big the input is.
- **Fast Computation:** Efficient to compute.
- **Pre-image Resistance:** Hard to reverse (We should not easily get the output from input hash)
- **Collision Resistance:** Hard to find two inputs with same hash.

### Why hashing is important in a password manager
We never store raw passwords. Instead we store hashes:
- User enters password -> hash it -> compare it with stored hash.
- Even if our database leaks, attacker won't see actual passwords.

## Types of hashing algorithms
---
#### 1. Cryptographic hash functions
These are secure and used in password storage and security systems.
**Common ones:**

- **MD5**
	- Very fast
	- **Not secure anymore** (easy to crack via collisions)
	- Avoid using it for passwords

- **SHA-1**
	- Better than MD5 but now **broken**
	- Deprecated for security use

- **SHA-256**
	- Part of SHA-2 family
	- Strong and widely used
	- Still safe for many applications

#### 2. Password-specific hashing algorithms
These are designed to be **slow and secure**, which protects against brute-force attacks.

- **bcrypt**
	- Adds **salt automatically**
	- Adjustable cost (work factor)
	- Widely used

- **scrypt**
    - Uses memory + CPU → harder to attack with GPUs
    - Good for high-security systems

- **Argon2**
	- Modern and recommended
	- Configurable (memory, time, parallelism)
	- Best choice today

#### 3. Non-cryptographic hash functions
Used for speed, not security (e.g., hash tables, indexing)
- **MurmurHash**
- **CityHash**

## Salting
---
A **salt** is a **random value added to a password before hashing**.
Instead of:

```
hash(password)
```

You do:
```
hash(password + salt)
```
Each user gets a **unique salt**.

##### Why it’s used
**Problem without salt:**
If two users have the same password:

```
password123 → same hash
```

Attackers can:
- Spot identical passwords
- Use **rainbow tables** (precomputed hash databases)

**With salt:**
```
password123 + randomSalt1 → hash1  
password123 + randomSalt2 → hash2
```

Now:
- Same password ≠ same hash
- Rainbow tables become useless
- Attackers must crack each password individually

## Key Stretching
---
**Key stretching means making hashing deliberately slow** by repeating it many times or using expensive algorithms.

Example idea:
```
hash(hash(hash(hash(password))))
```
Or using algorithms like:
- bcrypt
- Argon2

### Why it’s used

##### Problem:
Modern attackers use:
- GPUs
- ASICs
- Cloud clusters

They can try **billions of passwords per second** if hashing is fast.

##### Solution:
Make hashing **slow enough to hurt attackers but still okay for users**
Example:

- Normal hash: 0.000001 sec
- Stretched hash: 0.1 sec

For a user → unnoticeable  
For attacker → massively expensive

## Peppering
---
A **pepper** is a **secret value added to the password before hashing**, similar to salt—but with one big difference:

- Salt → stored in database
- Pepper → **NOT stored in database**

Example:
```
hash(password + salt + pepper)
```

#### Where is pepper stored?
- Environment variable
- Secure config
- Hardware security module (HSM)

#### Why it’s used
##### Scenario:
If attacker steals your database:

- They get hashes ✔
- They get salts ✔
- But **no pepper ❌**

So they **cannot verify guesses correctly** without the pepper.

### Putting it all together
A secure password storage flow looks like:
```
password → add salt → add pepper → apply slow hash (Argon2/bcrypt)
```


# Encryption
---
**Encryption** is a process of converting readable data(**plain text**) into unreadable data(**cipertext**) using a key, such that it can be reversed later.

```
plaintext → encryption(key) → ciphertext  
ciphertext → decryption(key) → plaintext
```

###### Example (conceptual)
```
"myPassword123" → (encrypt with key) → "a8F#k2!Lm9"
```

Later:
```
"a8F#k2!Lm9" → (decrypt with key) → "myPassword123"
```

### Types of Encryption
##### 1. Symmetric Encryption
Same key is used for encryption and decryption.

- Fast and efficient
- Used for storing data

**Examples:**
- AES 

##### 2. Asymmetric Encryption
Uses two keys:
- Public key (encrypt)
- Private key (decrypt)

**Examples:**
- RSA

Used for:
- Key exchange
- Secure communication (not for bulk password storage)

## Hashing VS Encryption

| Feature                 | Hashing        | Encryption         |
| ----------------------- | -------------- | ------------------ |
| Reversible              | ❌ No           | ✅ Yes              |
| Purpose                 | Verify data    | Protect data       |
| Key required            | ❌ No           | ✅ Yes              |
| Output                  | Fixed size     | Variable           |
| Use in password manager | Login password | Stored credentials |

# SHA-256 & SHA-512
---
They belong to the **SHA-2 family**.

- **SHA-256** → 256-bit output (32 bytes)
- **SHA-512** → 512-bit output (64 bytes)
👉 They are **cryptographic hash functions**, NOT password hashers.

#### ⚙️ Core Idea
```
input → SHA-256 → fixed 256-bit hash  
input → SHA-512 → fixed 512-bit hash
```
Example:

```
"password123"
→ SHA-256 →
ef92b778bafe771e89245b89ecbc...
```

#### Key Properties
| Property            | SHA-256 / SHA-512 |
| ------------------- | ----------------- |
| Deterministic*      | ✅                 |
| Fast                | ✅ VERY FAST       |
| Fixed output        | ✅                 |
| One-way             | ✅                 |
| Salt built-in       | ❌                 |
| Slow (for security) | ❌                 |
That **fast** part is the problem for passwords.

> Deterministic: **Deterministic** just means: **same input → always same output, every single time.**

##### Why SHA-256/512 alone are BAD for passwords
Attackers can do:
```
~ billions of hashes per second (GPU)
```

So:
```
password → SHA-256 → crackable very fast ❌
```

#### SHA Use Cases:
- Data integrity (file checks)
- Digital signatures
- HMAC (API signing)
- Pre-hashing (advanced cases)
- Key derivation building blocks

#### SHA vs bcrypt (Key Insight)

| Feature       | SHA-256     | bcrypt     |
| ------------- | ----------- | ---------- |
| Speed         | ⚡ Very fast | 🐢 Slow    |
| Salt          | ❌ Manual    | ✅ Built-in |
| Password safe | ❌ No        | ✅ Yes      |
SHA is a **primitive**, bcrypt is a **solution**

### 💻 Go Implementation
---
##### SHA-256
```go
package main

import (
	"crypto/sha256"  
	"encoding/hex"
	"fmt"
)

func hashSHA256(input string) string {
	hash := sha256.Sum256([]byte(input))
	return hex.EncodeToString(hash[:])
}

func main(){
	fmt.Println(hashSHA256("password123"))
}

```

#### SHA-512
---
```go
import (  
    "crypto/sha512"  
)  
  
func hashSHA512(input string) string {  
    hash := sha512.Sum512([]byte(input))  
    return hex.EncodeToString(hash[:])  
}
```

### 💻 TypeScript (Node.js)
---
##### SHA-256
```Typescript
import crypto from "crypto";

export function sha256(input: string): string {
	return crypto
			.createHash("sha256")
			.update(input)
			.digest("hex")
}
```

##### SHA-512
---
```typescript
export function sha512(input: string): string {
  return crypto
    .createHash("sha512")
    .update(input)
    .digest("hex");
}
```

#### ❌ DO NOT use SHA for:
- ❌ Password storage
- ❌ Encryption
- ❌ Vault protection

# bcrypt
---
**A password hashing algorithm with built-in salting + key stretching**

#### Why bcrypt exists (Problem it solves)
Normal hashes like:

- SHA-256 → ❌ too fast

Attackers can try billions of guesses / second

👉 bcrypt slows this down:
```
~100ms per hash (configurable)
```

Now brute force becomes:
VERY expensive 🚫

### Core Concepts in bcrypt
###### 1. Cost Factor (Work Factor)
This is the most important knob.
```
cost = 2^n iterations
```
Example:
- cost = 10 → 1024 rounds
- cost = 12 → 4096 rounds

👉 Increasing cost:
- ✔ more secure
- ❌ slower

###### 2. Salt (Built-in)
bcrypt automatically:
- generates random salt
- embeds it into the hash

###### 3. Hash Format (Important!)
A bcrypt hash looks like:
```
$2b$12$KIXQ4hFz8FJd9l5Yp5K7Fe3JzR9r0kYq1xYpQJ7Fz3XQ6VQy5e7eG
```
Breakdown:

| Part                     | Meaning           |
| ------------------------ | ----------------- |
| `$2b$`                   | Algorithm version |
| `12`                     | Cost factor       |
| `KIXQ4hFz8FJd9l5Yp5K7Fe` | Salt              |
| rest                     | Actual hash       |

👉 Everything is stored in **one string**

### How bcrypt works (Flow)
#### Signup:
```
password → bcrypt → storedHash
```

#### Login:
```
password → bcrypt (using stored salt) → compare
```

#### NOTE:
> bcrypt is **intentionally non-deterministic** for its output — it generates a random salt each time:
```
bcrypt("hello") → $2a$10$N9qo8uLOickgx2ZMRZo... (random salt baked in)
bcrypt("hello") → $2a$10$XvjDy3KqHB72kGQVH8k... (different every time)
```
>That's why we use `CompareHashAndPassword` instead of hashing again and comparing — bcrypt extracts the stored salt to reproduce the check.


### 💻 Go Implementation (Production Style)
##### Hash Password
---
```go
package main

import (
	"fmt"
	"goland.org/x/crypto/bcrypt"
)

func HashPassword(password string) (string, error){
	hashed, err := bcrypt.GenerateFromPassword(
		[]byte(password),
		bcrypt.DefaultCost // usually 10-12
	)
	
	return string(hashed),err
}

```

##### Verify Password
---
```go
func checkPassword(password, hash string) bool {
	err := bcrypt.CompareHashAndPassword(
		[]byte(hash),
		[]byte(password),
	)
	
	return err == nil
}

```

### 💻 TypeScript (Node.js)
##### Hash Password
---
```Typescript
import bcrypt from "bcrypt";

export async function hashPassword(password: string) {
  const saltRounds = 12;
  return await bcrypt.hash(password, saltRounds);
}
```

##### Verify Password
---
```Typescript
export async function verifyPassword(  
  password: string,  
  hash: string  
) {  
  return await bcrypt.compare(password, hash);  
}
```

### Important Gotchas
##### 1. bcrypt has input length limit
bcrypt was designed in 1999 and internally operates on a 72-byte input. If your password is longer than 72 bytes, **everything after byte 72 is silently ignored**.

This means these two passwords are treated as **identical** by bcrypt:

```
"correct-horse-battery-staple-some-very-long-passphrase-end-part-HERE"
"correct-horse-battery-staple-some-very-long-passphrase-end-part-XXXX"
```

Only the first 72 bytes are hashed — the differing tail is never seen.

**The idea is**: before passing to bcrypt, first hash the password with SHA-256. SHA-256 always outputs exactly 32 bytes, so we never hit the 72-byte ceiling.
```go
func HashPassword(password string) (string, error) {
	// Step 1: SHA-256 collapse → always 32 bytes, safely within bcrypt's 72-byte limit
	digest := sha256.Sum256([]byte(password))
	
	// Step 2: bcrypt the digest (encode to hex so it's printable)
	hash, err := bcrypt.GenerateFromPassword([]byte(fmt.Sprintf("%x",digest)), bcrypt.DefaultCost)
	if err != nil { return "", err } 
	
	return string(hash), nil
}



func CheckPassword(password, hash string) bool {
	digest := sha256.Sum256([]byte(password))
	err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(fmt.Sprintf("%x", digest)))
	
	return err == nil
}
```

##### 2. Not memory-hard
bcrypt:
- CPU expensive ✅
- Memory cheap ❌
GPUs can still optimize attacks


# Argon2
---
**Argon2** is a password hashing and key derivation algorithm designed to be secure by 
making attackers spend significant CPU time and memory for every password guess.

> Instead of just hashing a password quickly, Argon2 deliberately uses a large amount of memory and computation. This makes it expensive for attackers—especially those using GPUs or specialized hardware—to try many passwords at once.
#### Argon2 Variants
| Variant      | Use Case                                |
| ------------ | --------------------------------------- |
| Argon2d      | GPU-resistant (but side-channel unsafe) |
| Argon2i      | Side-channel safe (but weaker vs GPU)   |
| **Argon2id** | ✅ Hybrid → **BEST choice**              |
>A **side-channel attack** exploits how a system computes something - timing, power usage, memory access patterns - rather than breaking the algorithm mathematically.
>
>**THE CORE IDEA**
>Instead of cracking the hash directly, an attacker observes physical or behavioral leakage from the system during computation.

#### Argon2d
---
In Argon2d, the way memory is accessed depends on the password itself. This makes the process unpredictable and hard to optimize on GPUs. However, because the access pattern depends on secret data, an attacker observing the system might gain information through timing or cache behavior.

##### Argon2i
---
Argon2i avoids leaking information by always accessing memory in a predictable way, regardless of the password. This makes it safe against side-channel attacks, but since the pattern is predictable, attackers can optimize their hardware and potentially reduce memory usage using trade-offs.

##### Argon2id
---
Argon2id is a hybrid variant of Argon2 that combines the side-channel resistance of Argon2i with the GPU resistance of Argon2d, making it the recommended choice for most applications.

Argon2id starts with a predictable memory access pattern to avoid leaking sensitive information, and then switches to a password-dependent pattern to prevent GPU optimizations. This gives us the best of both worlds—security against both side-channel and brute-force attacks.


**Parameters:**

| Parameter           | Meaning          |
| ------------------- | ---------------- |
| **time (t)**        | iterations       |
| **memory (m)**      | RAM used (KB/MB) |
| **parallelism (p)** | threads          |
Argon2id uses:

- **Memory hardness**
- **CPU cost**
- **Parallelism**

So attacker must spend:
```
High CPU + High RAM + Time
```

###### Example Configuration
```
time = 3
memory = 64 MB
parallelism = 4
```
This means:
- Run algorithm 3 times
- Use 64MB RAM
- Use 4 threads

#### Password Storage
---
```go
package main

func hashPassword(password string) string {
	salt := make([]byte, 16)
	rand.Read(salt)
	
	hash := argon2.IDKey(
		[]byte(password),
		salt,
		3, // time
		64*1024, // memory 64MB
		4, // threads
		32, // key length
	)
	
	return base64.StdEncoding.EncodeToString(salt) + ":" + base64.StdEncoding.EncodeToString(hash)
}
```

```go
func verifyPassword(password, stored string) bool {
	parts := strings.Split(stored, ":")
	
	salt, _ := base64.StdEncoding.DecodeString(parts[0])
	hash, _ := base64.StdEncoding.DecodeString(parts[1])
	
	newHash := argon2.IDKey([]byte(password), salt, 3, 64*1024, 4, 32)
	
	return bytes.Equal(hash, newHash)
}
```

#### Key Derivation
 Instead of using a password directly for encryption, we transform it into a secure, fixed-length key using a slow and secure algorithm. This makes the key suitable for encryption and harder to guess.

#### TypeScript
---
```typescript
import argon2 from "argon2";  
  
export async function hashPassword(password: string) {  
  return await argon2.hash(password, {  
    type: argon2.argon2id,  
    memoryCost: 65536,  
    timeCost: 3,  
    parallelism: 4,  
  });  
}
```

```typescript
export async function verifyPassword(password: string, hash: string) {  
  return await argon2.verify(hash, password);  
}
```


#### Why Argon2id beats bcrypt
| Feature       | bcrypt  | Argon2id      |
| ------------- | ------- | ------------- |
| Memory hard   | ❌       | ✅             |
| GPU resistant | ❌       | ✅             |
| Tunable       | Limited | Very flexible |
| Modern        | ❌       | ✅             |

# ENCRYPTION ALGORITHMS: 
---

## AES (Advanced Encryption Standard)
---
AES is a symmetric encryption algorithm that uses the same secret key to encrypt and decrypt data.

> AES takes readable data (plaintext) and transforms it into unreadable data (ciphertext) using a secret key. The same key is later used to reverse the process.

#### AES Basics
| Property   | Value                |
| ---------- | -------------------- |
| Type       | Symmetric            |
| Block size | 128 bits (fixed)     |
| Key sizes  | 128 / 192 / 256 bits |
| Standard   | Widely used globally |
##### Important concept
AES  is a block cipher:
```
Plaintext -> split into 128 bit blocks -> encrypt each block
```

#### Problem: Raw AES is NOT enough

If we just do:
```
cipher = AES(key, plaintext)
```

👉 We run into serious issues:
- Pattern leakage
- No integrity protection
- Replay attacks

#### Example of bad mode: ECB
```
[A][A][A] → encrypted → [X][X][X]
```
👉 Same input → same output  
👉 Attacker sees patterns ❌

AES is used with modes is like:

| Mode    | Status          |
| ------- | --------------- |
| ECB     | ❌ insecure      |
| CBC     | ⚠️ old          |
| CTR     | ✅ stream-like   |
| **GCM** | ✅ modern (BEST) |
## AES-GCM (Modern Standard)
AES-GCM is an authenticated encryption mode that provides both confidentiality and integrity.

> AES-GCM not only encrypts your data, but also ensures that it hasn’t been modified. If someone tampers with the ciphertext, decryption fails.

### Components in AES-GCM
|Component|Purpose|
|---|---|
|Key (32 bytes)|encryption|
|Nonce (12 bytes)|uniqueness|
|Ciphertext|encrypted data|
|Auth Tag|integrity check|
##### Flow
```
plaintext
   ↓
AES-GCM(key, nonce)
   ↓
ciphertext + authTag
```

##### Decryption
```
ciphertext + tag
   ↓
verify tag
   ↓
decrypt
```

👉 If tag fails → ❌ reject

### CRITICAL RULE (Never break this)
###### Nonce must NEVER repeat
```
same key + same nonce = catastrophic failure 💀
```

#### Go Example
```go
func encrypt(data, key []byte) ([]byte, []byte, error) {
	block, _ := aes.NewCiper(key)
	gcm, _ := ciper.NewGCM(block)
	
	nonce := make([]byte, gcm.NonceSize())
	io.ReadFull(rand.Reader, nonce)
	
	ciperText := gcm.Seal(nil, nonce, data, nil)
	
	return cipherText, nonece, nil
}
```

#### Typescript Example
```typescript
import crypto from "crypto";

export function encrypt(text: string, key: Buffer) {
	const iv = crypto.randomBytes(12); // 12-bytes nonce
	
	const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
	
	const encrypted = Buffer.concat([
		ciper.update(text, "utf8"),
		ciper.final(),
	]);
	
	const tag = ciper.getAuthTag();
	
	return {encrypted, iv, tag};
}
```

## RSA (Asymmetric Encryption)
---
RSA is an asymmetric encryption algorithm that uses a public key for encryption and a private key for decryption.

> Anyone can encrypt data using your public key, but only you can decrypt it using your private key.

```
Public key → shared
Private key → secret
```

##### Use cases
|Use|Why|
|---|---|
|Key exchange|share secrets|
|Digital signatures|verify identity|
|TLS/HTTPS|secure communication|
#### Important: RSA is NOT for bulk data
RSA is:
- slow
- heavy

So we use:
**Hybrid Encryption**
```
1. Generate Random AES key
2. Encrypt data with AES
3. Encrypt AES key with RSA
```

```
data → AES → ciphertext
AES key → RSA → encrypted key
```

#### Typescript
```typescript
import crypto from "crypto";

const {publicKey, privateKey} = crypto.generateKeyPairSync("rsa", {
	modulusLength: 2048,
})

//Encrypt
const encrypted = crypto.publicEncrypt(
	publicKey,
	Buffer.from("secret data")
);

// Decrypt
const decrypted = crypto.privateDecrypt(
	privateKey,
	encrypted
);

```

#### Final Mental Model (VERY IMPORTANT)
---
###### 🔐 Symmetric (AES / XChaCha)
```
fast + used for data encryption
```

---
###### 🔐 Asymmetric (RSA)
```
slow + used for key exchange
```

---
##### 🔐 Argon2
```
password → key
```


## Nonce vs IV
A nonce/IV is a unique value used with a key to ensure that encrypting the same plaintext multiple times produces different ciphertexts.

- **Nonce** = “number used once” (strict meaning)
- **IV (Initialization Vector)** = more general term

In practice:
```
AES-GCM → uses nonce (12 bytes)
AES-CBC → uses IV (16 bytes)
```

##### Why Nonce exists (deep intuition)
```
password123 → encrypt → ABC
password123 → encrypt → ABC  ❌
```
attacker learns:
- same data
- patterns
- relationships

With nonce:
```
password123 + nonce1 → XYZ
password123 + nonce2 → PQR
```

Now, No pattern leakage.

```
IV → older block cipher modes (CBC, CFB) → must be random 
Nonce → modern AEAD modes (GCM, ChaCha20) → must be unique (random is safest)
```

#### NONCE REUSE ATTACK (Critical)

Scenario
```
Same key + same nonce used twice
```

###### What happens internally:
```
C1 = P1 XOR K
C2 = P2 XOR K
```

attacker computes:
```
C1 XOR C2 = P1 XOR P2
```

##### Impact
- attacker gets relationship between plaintexts
- if one plaintext is known → other is revealed

##### Real-world consequence
- complete loss of confidentiality
- possible forgery (in GCM)

### Authentication Tag (GCM magic)
---
> **Authentication tag is a cryptographic checksum that ensures data has not been modified.**
> 
> > It’s like a fingerprint of the ciphertext and metadata. If anything changes—even 1 bit—the tag verification fails.

###### In AES-GCM:
```
ciphertext + tag → sent
```

```
verify(tag)
    ↓
valid → decrypt
invalid → reject ❌
```

### Tag Forgery Attack (if misused)
If we:

- reuse nonce
- or misuse GCM

👉 attacker can:
```
forge valid ciphertext + tag
```

GCM becomes mathematically predictable under reuse

## BLOCK CIPHER & STREAM CIPHER
---
### Block Cipher

Encrypts data in **fixed-size chunks** (blocks). You feed it a block, it outputs an encrypted block.

```
Plaintext:  [  16 bytes  ] [  16 bytes  ] [  16 bytes  ] [ 5 bytes + padding ]
                 ↓                ↓               ↓              ↓
            [ Block 1   ] [  Block 2   ] [  Block 3   ] [  Block 4 (padded)  ]
                 ↓                ↓               ↓              ↓
Ciphertext: [ Enc Block ] [ Enc Block  ] [ Enc Block  ] [ Enc Block          ]
```

The cipher itself is just a **mathematical permutation** — given a key and a 16-byte block, produce a scrambled 16-byte block. That's it. How you _chain_ these blocks together is called the **mode of operation** (CBC, GCM, CTR etc.).

**Algorithms:** AES, DES, 3DES, Blowfish, Twofish

---

### Stream Cipher

Generates a **keystream** — an infinite pseudorandom sequence of bytes — and XORs it with your plaintext byte by byte. No blocks, no padding.

```
Plaintext:   H    e    l    l    o
             ↓    ↓    ↓    ↓    ↓
Keystream:  0x4F 0x2A 0x91 0x3C 0x77   (pseudorandom, derived from key+nonce)
             ↓    ↓    ↓    ↓    ↓
Ciphertext: 0x07 0x4B 0xFD 0x50 0x1F

Decrypt: just XOR ciphertext with same keystream again → plaintext back
```

No padding needed. Encrypt 5 bytes → get 5 bytes back.

**Algorithms:** ChaCha20, RC4 (broken), Salsa20

---

### Where It Gets Interesting — Block Ciphers Acting as Stream Ciphers

AES is a block cipher, but in **CTR mode or GCM mode** it _behaves like a stream cipher_:

```
CTR mode:
  AES(key, nonce+0) → keystream block 1
  AES(key, nonce+1) → keystream block 2
  AES(key, nonce+2) → keystream block 3
  ...XOR with plaintext → no padding needed
```

So AES-GCM is technically a block cipher **used as a stream cipher**. This is why:

- No padding required in GCM (unlike CBC)
- Encryption and decryption are the same operation (just XOR)

---

### Mode of Operation Matters a Lot for Block Ciphers

| Mode    | How blocks are chained                    | Padding needed | Notes                                                 |
| ------- | ----------------------------------------- | -------------- | ----------------------------------------------------- |
| **ECB** | Each block independent                    | Yes            | ❌ Never use — identical blocks = identical ciphertext |
| **CBC** | Each block XORed with previous ciphertext | Yes            | IV must be random, sequential only                    |
| **CTR** | Counter fed into AES to make keystream    | No             | Parallel encryption possible                          |
| **GCM** | CTR + authentication tag (AEAD)           | No             | ✅ What you should use                                 |

The famous **ECB penguin** illustrates why ECB is broken:

```
Original image encrypted with AES-ECB:
Still clearly shows the penguin outline
because identical pixel blocks → identical ciphertext blocks
```

---

### AEAD — The Modern Standard

Both AES-GCM and XChaCha20-Poly1305 are **AEAD** (Authenticated Encryption with Associated Data):

```
Regular encryption:  confidentiality only
                     attacker can flip bits in ciphertext → corrupted plaintext

AEAD:                confidentiality + integrity + authenticity
                     any tampering → authentication tag fails → decryption rejected
```

The `-Poly1305` in XChaCha20-Poly1305 and the `G` (Galois) in AES-GCM are both the **authentication** component bolted onto the stream cipher part.

---

### Summary

```
AES          → block cipher (16-byte blocks)
  + CBC mode → block cipher behaving as block cipher (padding, sequential)
  + GCM mode → block cipher behaving as stream cipher + authentication ✅

ChaCha20          → stream cipher (pure keystream XOR)
  + Poly1305      → stream cipher + authentication ✅
  XChaCha20       → ChaCha20 with extended 192-bit nonce ✅

RC4          → stream cipher ❌ broken, never use
DES / 3DES   → block cipher ❌ too weak now
```


# XChaCha20-Poly1305
---
 **XChaCha20-Poly1305 is an authenticated encryption algorithm that uses a 256-bit key, a 192-bit nonce, and provides both confidentiality and integrity.**

> It encrypts your data using a fast stream cipher (ChaCha20) and protects it from tampering using a MAC (Poly1305), while allowing you to safely use large random nonces.

### Core Components
|Component|Role|
|---|---|
|**XChaCha20**|encryption (stream cipher)|
|**Poly1305**|authentication (MAC)|
|**Nonce (24 bytes)**|uniqueness|
|**Key (32 bytes)**|secret|
#### Why XChaCha20 exists

Regular ChaCha20 uses:
```
96-bit nonce (12 bytes)
```

Problem:
- risk of nonce reuse
- hard to manage safely at scale
#### XChaCha20 solves this

192-bit nonce (24 bytes)

👉 This gives:
- **huge nonce space**
- safe random nonces
- no need for counters

#### Key Idea
XChaCha20 uses:

> **HChaCha20 → derive subkey → ChaCha20 encryption**

##### Flow:
```
master key + nonce (first 16 bytes)
        ↓
HChaCha20
        ↓
derived subkey
        ↓
ChaCha20 with remaining nonce
```

### Encryption Flow (Step-by-step)

##### Internally:
```
Key (32 bytes)  
Nonce (24 bytes)

→ derive subkey (HChaCha20)  
→ generate keystream  
→ XOR with plaintext → ciphertext  
→ generate Poly1305 tag
```

###### Output:
```
ciphertext + authentication tag
```

##### Decryption
```
verify tag
    ↓
valid → decrypt
invalid → reject ❌
```

### Why it’s considered safer than AES-GCM (in practice)
---

1. Nonce Safety

| AES-GCM                 | XChaCha20                         |
| ----------------------- | --------------------------------- |
| 12 bytes                | 24 bytes                          |
| reuse = catastrophic 💀 | reuse = still bad but less likely |

2. No hardware dependency
- AES-GCM relies on:
    - AES-NI (CPU instructions)
- ChaCha20:
    - fast everywhere
    - consistent performance

3. Better misuse resistance
👉 With random nonces:
- AES-GCM → risky
- XChaCha20 → safe


### Poly1305
---
 **Poly1305 is a one-time message authentication code that ensures data integrity.**

#### How it works (intuition)
```
tag = MAC(key, ciphertext)
```

👉 If anything changes:
```
tag mismatch → reject ❌
```

###### Important rule

Poly1305 key must be:
```
unique per encryption
```

👉 XChaCha20 ensures this automatically ✔

### Go Example
---
```go
import (  
"golang.org/x/crypto/chacha20poly1305"  
)

func encrypt(key, plaintext, []byte) ([]byte, []byte, error) {
	aead, _ := chacha20poly1305.NewX(key)
	
	nonce := make([]byte, aead.NonceSize())
	rand.Read(nonce)
	
	ciperText := aead.Seal(nil, nonce, plaintext, nil)
	
	return ciperText, nonce nil
}
```


