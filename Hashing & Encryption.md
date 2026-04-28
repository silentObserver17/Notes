# HASHING
---
A Hashing Algorithm is a function that takes input(like a password) and converts it into fixed length string of characters, called hash value or digest.

It is a one way transformation:
```
Input: `"myPassword123"`
Output: `"5f4dcc3b5aa765d61d8327deb882cf99"` (example)
```

#### Key properties of hashing algorithms

A good hashing algorithm has these characteristics:
- **Deterministic**: Same input -> Same output every time.
- **Fixed Output Length:** No matter how big the input is.
- **Fast Computation**: > Cryptographic hash functions are designed to be fast, but password hashing functions are intentionally slow.
- **Pre-Image Resistance**: - > Given a hash, it should be infeasible to find the original input.
- **Collision Resistance:** Hard to find two inputs with same hash.
- **Avalanche effect** → small change → completely different hash
#### Why hashing is important in a password manager

We never store raw passwords. Instead we store hashes:
- User enters password -> hash it -> compare it with stored hash.
- Even if our database leaks, attacker won't see actual passwords.

### Types of Hashing Algorithm
---
##### 1. Cryptographic Hash Functions
Designed for **security, integrity, and verification**

**Properties:**
- Deterministic
- Fixed output size
- Pre-image resistance
- Collision resistance
- Avalanche effect

**Examples:**
- **MD5** → Fast but broken (collision attacks)
- **SHA-1** → Deprecated (not secure)
- **SHA-256** → Secure, widely used (blockchain, SSL)

##### 2. Password Hashing / Key Derivation Functions (KDFs)
Designed to be **slow and resistant to brute-force attacks**

**Why slow?**  
→ Makes large-scale attacks impractical

**Examples:**
- **bcrypt**
    - Built-in salt
    - Adjustable cost factor
- **scrypt**
    - Memory + CPU intensive
    - Resists GPU attacks
- **Argon2**
    - Winner of Password Hashing Competition
    - Configurable (memory, time, parallelism)
    - Recommended today

##### 3. Non-Cryptographic Hash Functions
Designed for **speed, not security**

**Used in:**
- Hash tables
- Caching
- Load balancing

**Examples:**
- MurmurHash
- CityHash

>👉 _“For password storage, we should never use fast hash functions like SHA-256 directly. Instead, we use KDFs like bcrypt or Argon2 with salting and work factors.”_

### Salting
---
**Definition:**  
A **random value added to a password before hashing**

**How it works:**
```
hash = H(password + salt)
```
##### Why it’s used

**Problem without salt:** If two users have the same password:

```
password123 → same hash
```

Attackers can:
- Spot identical passwords
- Use **rainbow tables** (precomputed hash databases)

**With salt:**
```
password123 + randomSalt1 → hash1  
password123 + randomSalt2 → hash2
```

Now:
- Same password ≠ same hash
- Rainbow tables become useless
- Attackers must crack each password individually

**Key points:**
- Salt must be **unique per user**
- Salt is **stored in the database** (not secret)
- Modern algorithms like **bcrypt** and **Argon2** handle salting automatically

### Key Stretching
---
**Definition:**  
Making hashing intentionally **slow** by applying it multiple times or increasing computational cost

Example idea:
```
hash(hash(hash(hash(password))))
```

OR via cost parameters (better approach)

**Why it’s needed:**
- Slows down **brute-force attacks**
- Makes each guess expensive

**Key points:**
- Implemented via **work factor / cost**
- Built into:
    - **bcrypt** (cost factor)
    - **scrypt** (CPU + memory)
    - **Argon2** (time, memory, parallelism)

👉 Real-world idea:
- Fast hash (like SHA-256): millions/sec ❌
- bcrypt: maybe 100/sec ✅ (depends on cost)

### Peppering
---
**Definition:**  
A **secret value added to passwords before hashing**, similar to salt but **kept hidden**

**How it works:**
```
hash = H(password + salt + pepper)
```

**Why it’s needed:**
- Adds **extra layer of security**
- Even if DB is leaked → attacker still needs pepper

**Key points:**
- Pepper is **NOT stored in DB**
- Stored in:
    - Environment variables
    - Secrets manager
- Same pepper can be used for all users (or rotated)

---
### Key Differences
| Feature          | Salt 🧂                | Pepper 🌶️               |
| ---------------- | ---------------------- | ------------------------ |
| Secret?          | ❌ No                   | ✅ Yes                    |
| Stored in DB?    | ✅ Yes                  | ❌ No                     |
| Unique per user? | ✅ Yes                  | ❌ Usually same           |
| Purpose          | Prevent rainbow tables | Add extra security layer |

>“Passwords should be hashed using KDFs like bcrypt or Argon2, which include salting and key stretching. Salting prevents rainbow table attacks, key stretching slows brute-force attempts, and optionally peppering adds an extra secret layer stored outside the database.”


###  🌈 Rainbow Table Attack
---
A **precomputed lookup attack** used to reverse hashes.

Instead of hashing passwords on the fly, attackers:
1. Precompute hashes for millions/billions of common passwords
2. Store them in a table
3. When they get a leaked hash → just look it up

**How it works:**
```
password → hash
```

Attacker builds:
```
"123456" → hash1  
"password" → hash2  
"admin" → hash3  
```

Then if your DB leaks:
```
hash2 → instantly maps to "password"
```

No brute force needed. Just lookup.

##### Why it’s dangerous
- Extremely fast attack
- Works well on **unsalted hashes**
- Common passwords get cracked instantly

> _“Rainbow table attacks rely on precomputed hashes and are completely mitigated by proper salting.”_

### Side-Channel Attacks
---
Instead of breaking the algorithm mathematically, attackers exploit **information leakage from implementation**.
They observe _how_ a system behaves, not just _what_ it outputs.

##### Common types

###### ⏱️ Timing Attack
- Measure how long operations take
- Example: password comparison exits early → leaks info

```
if (input[i] != stored[i]) return false;
```

Faster failure = attacker learns correct prefix

###### 🔌 Power Analysis
- Used in hardware (smart cards, embedded systems)
- Power consumption reveals operations

###### 📡 Cache / Memory Attacks
- Observe memory access patterns
- Can leak sensitive data

###### 💥 Why it’s dangerous
- Works even if algorithm is **cryptographically secure**
- Targets **implementation flaws**, not theory

> _“Side-channel attacks exploit implementation leaks like timing or memory access, so defenses focus on constant-time operations and secure implementations.”_

## SHA-256
---
SHA-256 takes **any input → produces a fixed 256-bit hash**.

👉 Design goals:
- One-way (pre-image resistant)
- Collision resistant
- Deterministic
- Avalanche effect

#### High-Level Flow
Think of SHA-256 like a pipeline:

```
Input → Padding → Blocks → Compression Rounds → Final Hash
```

Think of it like a **meat grinder** — you can put anything in, but you can never reconstruct the original from what comes out.

```
"hello"        →  2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
"hello!"       →  ce06092fb948d9af2d6f72c0a30d37b6b3f0d176c5b4bcf5a5143f2b89bfb40
"hello world"  →  b94d27b9934d3e08a52e52d7da7dabfac484efe04294e576ca0588e1d2b21c4c
```

Notice — changing even **one character** completely changes the output.

#### Key Properties
| Property                | Meaning                                                             |
| ----------------------- | ------------------------------------------------------------------- |
| **Fixed output**        | Always 256 bits / 64 characters, no matter the input size           |
| **One-way**             | You cannot reverse it back to the original                          |
| **Deterministic**       | Same input always gives same output                                 |
| **Avalanche effect**    | Tiny change in input = completely different output                  |
| **Collision resistant** | Nearly impossible for two different inputs to produce the same hash |

##### Why SHA-256 is Fast
- Uses simple CPU operations (bitwise ops)
- No memory hardness
- Optimized for performance

👉 That’s why:  
❌ Bad for passwords  
✅ Great for integrity

#### SHA Use Cases:
- Data integrity (file checks)
- Digital signatures
- HMAC (API signing)
- Pre-hashing (advanced cases)
- Key derivation building blocks

### Limitations
- Fast → vulnerable to brute-force (for passwords)
- Needs salting externally
- No built-in cost factor

👉 That’s why we use:
- bcrypt
- Argon2
for password storage


>“SHA-256 processes input in 512-bit blocks, applies padding, and runs each block through 64 rounds of bitwise transformations using functions like Ch and Maj. It maintains an internal state of 8 words and produces a 256-bit hash. It’s designed to be fast and collision-resistant, making it suitable for integrity but not for password hashing.”

#### Go Implementation
---
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

#### TypeScript (Node.js) Implementation
---
```typescript
import crypto from "crypto";

export function sha256(input: string): string {
	return crypto
			.createHash("sha256")
			.update(input)
			.digest("hex")
}
```


## bcrypt
---
**A password hashing algorithm with built-in salting + key stretching**

#### What problem does bcrypt solve?

👉 **Fast hashes (like SHA-256) are bad for passwords**
- Attackers can try **millions of guesses per second**
- Even with salt → still brute-forceable

👉 bcrypt solves this by being:
- **Slow (intentionally)**
- **Adaptive (cost can increase over time)**

##### Core Idea
bcrypt =  **hashing + salting + key stretching (built-in)**

####  Does bcrypt use salt?

👉 Yes—and this is important.
- Automatically generates a **random salt per password**
- Stores it as part of the final hash

Example (conceptual):
```
$2b$10$<salt><hash>
```

- `10` → cost factor
- Salt is embedded
- Hash is stored together

> “bcrypt automatically handles salting, so developers don’t need to manage it separately.”

#### What is the cost factor?
This is the **most important concept in bcrypt**.

👉 Cost controls how slow hashing is.
cost = 10 → 2^10 iterations  
cost = 12 → 2^12 iterations

👉 Increasing cost:
- Makes hashing slower
- Makes brute-force harder

#### Real-world intuition
| Cost | Approx Time      |
| ---- | ---------------- |
| 8    | Very fast (weak) |
| 10   | Acceptable       |
| 12   | Good standard    |
| 14+  | High security    |
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

👉 Everything is stored in **one string**
#### Why is bcrypt secure?
 **✅ Slow hashing**
- Limits brute-force speed

 **✅ Built-in salt**
- Prevents rainbow table attacks

 **✅ Adaptive cost**
- Future-proof against faster hardware

#### Limitations of bcrypt (important!)
- **Not memory-hard**
	- Can still be optimized on GPUs
That’s why **Argon2** is considered better today

-  **Input length limit (~72 bytes)**
	- Longer passwords get truncated

- **Slower but not enough for future threats**
	- Modern recommendation is shifting toward Argon2


#### bcrypt vs SHA-256
| Feature        | SHA-256     | bcrypt   |
| -------------- | ----------- | -------- |
| Speed          | Very fast ⚡ | Slow 🐢  |
| Salt           | Manual      | Built-in |
| Cost factor    | ❌           | ✅        |
| Password safe? | ❌           | ✅        |

> “SHA-256 is designed for speed and integrity, while bcrypt is designed for slow password hashing with built-in salting and cost control.”

#### NOTE:
> bcrypt is **intentionally non-deterministic** for its output — it generates a random salt each time:

```
bcrypt("hello") → $2a$10$N9qo8uLOickgx2ZMRZo... (random salt baked in)
bcrypt("hello") → $2a$10$XvjDy3KqHB72kGQVH8k... (different every time)
```

> That's why we use `CompareHashAndPassword` instead of hashing again and comparing — bcrypt extracts the stored salt to reproduce the check.

#### When should you use bcrypt?
👉 Use bcrypt when:
- You need **secure password storage**
- Simplicity is preferred
- Argon2 is not available

👉 Prefer **Argon2** when:
- You want modern, stronger protection

>“bcrypt is a password hashing function that incorporates salting and key stretching with an adjustable cost factor to make brute-force attacks computationally expensive.”

#### Go Implementation (Production Style)
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

```ts
import bcrypt from "bcrypt";

export async function hashPassword(password: string) {
  const saltRounds = 12;
  return await bcrypt.hash(password, saltRounds);
}
```

##### Verify Password
---
```ts
export async function verifyPassword(  
  password: string,  
  hash: string  
) {  
  return await bcrypt.compare(password, hash);  
}
```

# Argon2
---
**Argon2** is a password hashing and key derivation algorithm designed to be secure by making attackers spend significant CPU time and memory for every password guess.

#### Why was Argon2 created?
bcrypt was good, but had limitations:
- Not memory-hard
- Vulnerable to GPU/ASIC optimizations

👉 Argon2 was designed to fix this.

#### Argon2 Variants

| Variant      | Use Case                                |
| ------------ | --------------------------------------- |
| Argon2d      | GPU-resistant (but side-channel unsafe) |
| Argon2i      | Side-channel safe (but weaker vs GPU)   |
| **Argon2id** | ✅ Hybrid → **BEST choice**              |
> A **side-channel attack** exploits how a system computes something - timing, power usage, memory access patterns - rather than breaking the algorithm mathematically.
> 
> **THE CORE IDEA** Instead of cracking the hash directly, an attacker observes physical or behavioral leakage from the system during computation.

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

#### Core Idea
Argon2 =  
👉 **Slow + Memory-Hard + Configurable**

Most important word here:
👉 **Memory-hard**

##### What does memory-hard mean?
It requires **a lot of RAM** to compute the hash.

👉 Why this matters:
- GPUs are great at parallel compute
- But **memory is expensive and limited per core**

👉 Result:
- Attackers can’t scale easily
- Brute-force becomes much harder

#### Key Parameters
- **Time Cost (t)**
	- Number of iterations
	- More iterations → slower
- **Memory Cost (m)**
	- Amount of RAM used
	- More memory → harder for attackers
- **Parallelism (p)**
	- Number of threads
	- Controls CPU parallel usage

> “Argon2 allows tuning of time, memory, and parallelism, making it adaptable to different security requirements.”

#### Does Argon2 use salt?
Yes (like bcrypt)
- Automatically uses a **unique salt**
- Prevents rainbow table attacks

#### What about pepper?
👉 Not built-in, but we can add it externally

#### Why is Argon2 better than bcrypt?
| Feature         | bcrypt  | Argon2 |
| --------------- | ------- | ------ |
| Memory-hard     | ❌       | ✅      |
| GPU resistance  | Medium  | High   |
| Configurable    | Limited | Highly |
| Modern standard | ❌       | ✅      |
**Key takeaway:**
> bcrypt slows attackers  
> Argon2 **slows + limits their hardware advantage**

###### SUMMARY:
>“Argon2 is a modern password hashing algorithm that improves on bcrypt by introducing memory hardness and configurable parameters, making it resistant to GPU-based brute-force attacks.”

#### Implementation:

###### GO:
---
```go
package password

import (
	"crypto/rand"
	"crypto/subtle"
	"encoding/base64"
	"errors"
	"fmt"
	"strings"

	"golang.org/x/crypto/argon2"
)

type Params struct {
	Memory      uint32
	Iterations  uint32
	Parallelism uint8
	SaltLength  uint32
	KeyLength   uint32
}

var DefaultParams = &Params{
	Memory:      64 * 1024, // 64 MB
	Iterations:  3,
	Parallelism: 4,
	SaltLength:  16,
	KeyLength:   32,
}

// GenerateHash creates a hash in PHC string format
func GenerateHash(password string, p *Params) (string, error) {
	salt, err := generateRandomBytes(p.SaltLength)
	if err != nil {
		return "", err
	}

	hash := argon2.IDKey(
		[]byte(password),
		salt,
		p.Iterations,
		p.Memory,
		p.Parallelism,
		p.KeyLength,
	)

	b64Salt := base64.RawStdEncoding.EncodeToString(salt)
	b64Hash := base64.RawStdEncoding.EncodeToString(hash)

	// PHC format
	encoded := fmt.Sprintf(
		"$argon2id$v=19$m=%d,t=%d,p=%d$%s$%s",
		p.Memory,
		p.Iterations,
		p.Parallelism,
		b64Salt,
		b64Hash,
	)

	return encoded, nil
}

func ComparePassword(password, encodedHash string) (bool, error) {
	p, salt, hash, err := decodeHash(encodedHash)
	if err != nil {
		return false, err
	}

	newHash := argon2.IDKey(
		[]byte(password),
		salt,
		p.Iterations,
		p.Memory,
		p.Parallelism,
		p.KeyLength,
	)

	if subtle.ConstantTimeCompare(hash, newHash) == 1 {
		return true, nil
	}
	return false, nil
}

// --- helpers ---

func generateRandomBytes(n uint32) ([]byte, error) {
	b := make([]byte, n)
	_, err := rand.Read(b)
	return b, err
}

func decodeHash(encoded string) (*Params, []byte, []byte, error) {
	parts := strings.Split(encoded, "$")
	if len(parts) != 6 {
		return nil, nil, nil, errors.New("invalid hash format")
	}

	var version int
	_, err := fmt.Sscanf(parts[2], "v=%d", &version)
	if err != nil || version != 19 {
		return nil, nil, nil, errors.New("incompatible version")
	}

	p := &Params{}
	_, err = fmt.Sscanf(parts[3], "m=%d,t=%d,p=%d",
		&p.Memory, &p.Iterations, &p.Parallelism)
	if err != nil {
		return nil, nil, nil, err
	}

	salt, err := base64.RawStdEncoding.DecodeString(parts[4])
	if err != nil {
		return nil, nil, nil, err
	}
	p.SaltLength = uint32(len(salt))

	hash, err := base64.RawStdEncoding.DecodeString(parts[5])
	if err != nil {
		return nil, nil, nil, err
	}
	p.KeyLength = uint32(len(hash))

	return p, salt, hash, nil
}
```

👉 **Argon2 doesn’t auto-generate the salt for you in low-level APIs like Go’s `argon2.IDKey`.**  
👉 It **uses** a salt, but **you must provide it**.

###### TypeScript:
---
```ts
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

```ts
export async function verifyPassword(password: string, hash: string) {  
  return await argon2.verify(hash, password);  
}
```

# Encryption
---
Encryption = converting **plaintext → ciphertext** using a **key**
```
plaintext + key → ciphertext
```

Goal:
- Keep data **confidential** (only authorized parties can read it)

#### Encryption vs Hashing
| Feature    | Hashing    | Encryption      |
| ---------- | ---------- | --------------- |
| Reversible | ❌ No       | ✅ Yes           |
| Key used   | ❌ No       | ✅ Yes           |
| Purpose    | Integrity  | Confidentiality |
| Output     | Fixed Size | Variable        |
> “Hashing is one-way, while encryption is reversible using a key.”

### Types of Encryption
#### Symmetric Encryption
Same key for encryption + decryption
```
Encrypt(key, data) → ciphertext  
Decrypt(key, ciphertext) → data
```

##### Example:
👉 **Advanced Encryption Standard(AES)**
- Fast ⚡
- Used everywhere (TLS, databases, disk encryption)

#### Asymmetric Encryption
Two keys:
- Public key (encrypt)
- Private key (decrypt)

**Example:**
👉 **RSA**

#### ⚡ Key Difference
| Feature | Symmetric | Asymmetric   |
| ------- | --------- | ------------ |
| Speed   | Fast ⚡    | Slow 🐢      |
| Keys    | 1         | 2            |
| Use     | Bulk data | Key exchange |

> “Asymmetric encryption is mainly used for key exchange, while symmetric encryption is used for actual data encryption.”


#### How Real Systems Work
Real-world systems use **both**:
###### Example: HTTPS
1. Use **RSA** (or similar) to securely share a key
2. Use **AES** for fast communication

👉 Hybrid encryption model

### Block Cipher vs Stream Cipher
---
#### Block Cipher
👉 Encrypts **fixed-size blocks** (e.g., 128 bits)

 **Example:**
👉 Advanced Encryption Standard

###### Properties:
- Deterministic (same input → same output with same key)
- Needs **modes of operation** (CBC, GCM, etc.)

#### 🌊 Stream Cipher
👉 Encrypts **data bit-by-bit or byte-by-byte**

**Example:**
👉 ChaCha20

###### Properties:
- Generates a **keystream**
- Combined with plaintext using XOR

#### ⚡ Key Difference
| Feature     | Block Cipher | Stream Cipher     |
| ----------- | ------------ | ----------------- |
| Unit        | Blocks       | Continuous stream |
| Needs mode? | Yes          | No                |
| Speed       | Moderate     | Very fast         |

> “Block ciphers operate on fixed-size blocks and require modes, while stream ciphers generate a keystream and encrypt data continuously.”

#### Modes of Operation
Block ciphers alone are not enough. 
Modes define:
- How to handle **data longer than one block**
- How to introduce **randomness (IV/nonce)**
- How to ensure **security properties** like:
    - Confidentiality
    - Integrity (in modern modes)
#### ❌ ECB
- Same plaintext → same ciphertext  
    👉 leaks patterns → insecure

---
#### ⚠️ CBC
- Uses IV
- Chains blocks together
- No authentication ❌

---
#### ✅ GCM (Modern Standard)

👉 AEAD mode
- Encryption + integrity
- Fast + secure

> “Never use ECB; prefer AEAD modes like GCM.”

### IV vs Nonce
---
##### 🔹 IV (Initialization Vector)
- Random value
- Used in modes like CBC

---
##### 🔹 Nonce (Number used once)
- Must be **unique (not necessarily random)**
- Used in modern modes like **GCM**, **ChaCha20**

#### ⚠️ Key Difference

| Feature        | IV     | Nonce               |
| -------------- | ------ | ------------------- |
| Requirement    | Random | Unique              |
| Reuse allowed? | ❌ No   | ❌ No                |
| Used in        | CBC    | GCM, stream ciphers |
> “A nonce must be unique per encryption, while an IV is typically random—reusing either can break security.”

#### Padding (important for block ciphers)

Block ciphers need fixed-size input.
👉 If data isn’t aligned → padding is added

**Example:**
- PKCS#7 padding

⚠️ Why important?
- Incorrect padding handling → vulnerabilities (padding oracle attacks)

#### Padding Oracle Attack
After decryption, the receiver **validates the padding**. If padding is invalid, it typically returns an error. This error response is the **oracle**.

In cryptography, an oracle is any system that gives you a **yes/no answer** about something secret. Here, the server tells you (directly or indirectly):

> "Was the padding on this decrypted block valid?"

That single bit of information is enough to decrypt the entire message — without knowing the key.

#### Keystream (for stream ciphers)

Stream ciphers work like:
```
ciphertext = plaintext ⊕ keystream
```

👉 If keystream is reused:
- Security breaks instantly ❌

#### Key Management (VERY IMPORTANT)

Encryption is only as strong as:  
👉 **how you manage keys**
##### Includes:
- Key generation (secure randomness)
- Key storage (KMS, env, HSM)
- Key rotation
- Access control

> “Most real-world crypto failures come from poor key management, not weak algorithms.”

#### Common Pitfalls (high-value)
- ❌ Reusing nonce
- ❌ Using ECB
- ❌ No authentication (just encryption)
- ❌ Hardcoding keys
- ❌ Rolling your own crypto

## AEAD — The Core Abstraction
---
**Authenticated Encryption with Associated Data**

AEAD gives you **everything you want in one primitive**:
- 🔒 Confidentiality (encryption)
- ✅ Integrity (no tampering)
- 🔑 Authentication (trusted source)

##### Mental Model
```
ciphertext, tag = AEAD_Encrypt(key, nonce, plaintext, associated_data)
plaintext = AEAD_Decrypt(key, nonce, ciphertext, associated_data, tag)
```

#### What is “Associated Data”?
This is one of the most important concepts.

👉 Data that is:
- **NOT encrypted**
- BUT **must be authenticated**

 **Example**
- HTTP headers
- User ID
- Metadata

If attacker modifies it → decryption fails ❌

> “AEAD allows binding metadata to ciphertext without encrypting it.”

#### Key Components of AEAD
---
##### 1. Key
- Secret
- Shared between parties
##### 2. Nonce (VERY IMPORTANT)
- Unique per encryption
- Prevents reuse attacks

👉 Reusing nonce = catastrophic ❌
##### 3. Plaintext
- Data you want to encrypt
#### 4. Ciphertext
- Encrypted output
##### 5. Authentication Tag
- Ensures integrity
- Detects tampering

#### AEAD Workflow (Step-by-Step)

##### Encryption:
1. Take plaintext
2. Mix with key + nonce
3. Produce:
    - ciphertext
    - authentication tag
##### Decryption:
1. Verify tag
2. If valid → decrypt
3. If not → reject

👉 Important:

> “Decryption should never happen before authentication.”


#### Popular AEAD Schemes
---
#### 🔹 AES-GCM
👉 Based on **Advanced Encryption Standard**

- Very fast (hardware accelerated)
- Widely used (TLS, HTTPS)

#### 🔹 ChaCha20-Poly1305
👉 Uses **Poly1305**
- Better on mobile / low-end CPUs
- Resistant to timing attacks

#### 🔹 XChaCha20-Poly1305
- Extended nonce version
- Safer nonce handling
- Great for distributed systems


##### AES-GCM vs ChaCha20-Poly1305
|Feature|AES-GCM|ChaCha20-Poly1305|
|---|---|---|
|Speed|Fast (with hardware)|Fast (software)|
|Platform|CPUs with AES-NI|Mobile / IoT|
|Nonce size|96-bit|96-bit (XChaCha: 192-bit)|

>“In modern systems, we never use raw encryption primitives. We use AEAD constructions like AES-GCM or ChaCha20-Poly1305 to ensure both confidentiality and integrity in a single operation.”

### IND-CPA / IND-CCA Security
---
These are formal **security definitions** for encryption schemes — they answer the question: _"What does it mean for an encryption scheme to be secure?"_

#### IND — Indistinguishability
A scheme is **indistinguishable** if an attacker cannot tell which of two messages was encrypted, even after seeing the ciphertext.

There's a **challenger** (the system) and an **attacker** (you trying to break it).

The attacker's goal is simple:

> Pick two messages. Get one encrypted. Guess which one. Win.

If you can't guess better than a coin flip — the scheme is **secure**.

#### IND-CPA — Indistinguishability under Chosen Plaintext Attack
**You have an encryption machine. That's it.**
You can type anything into it and get back ciphertext. Then you submit your two messages, get the challenge ciphertext C*, and try to guess.

**The key question it asks:**
> "Even if I can encrypt whatever I want — can I figure out what's inside C*?"

If no → **IND-CPA secure.**
**What it kills:** ECB mode. Because in ECB, encrypting "cat" always gives the same ciphertext. So you encrypt "cat" yourself, compare it to C* — and instantly know if C* is "cat". A randomized scheme (random IV/nonce) prevents this.

```
Attacker                        Challenger
   |                                |
   |-- encrypt("hello") ---------->|
   |<-- C1 ------------------------|
   |-- encrypt("world") ---------->|
   |<-- C2 ------------------------|
   |                                |
   |-- submit M0, M1 ------------->|
   |<-- Encrypt(Mb) = C* ----------|
   |                                |
   |-- encrypt("anything") ------->|  ← still allowed after seeing C*
   |<-- C3 ------------------------|
   |                                |
   |-- guess b ------------------->|
```


#### IND-CCA — Indistinguishability under Chosen Ciphertext Attack
**You have an encryption machine AND a decryption machine.**
Same game, but now you can also hand any ciphertext to the decryption machine and get the plaintext back. The only rule: you can't hand it C* directly.

**The key question it asks:**

> "Even if I can decrypt whatever I want (except C* itself) — can I still figure out what's inside C*?"

If no → **IND-CCA secure.**

```
Attacker                        Challenger
   |                                |
   |-- decrypt(C_any) ------------>|  ← before challenge
   |<-- plaintext -----------------|
   |                                |
   |-- submit M0, M1 ------------->|
   |<-- C* = Encrypt(Mb) ----------|
   |                                |
   |-- decrypt(C_modified) ------->|  ← after challenge, C_modified ≠ C*
   |<-- plaintext -----------------|
   |                                |
   |-- guess b ------------------->|
```

##### The Two CCA Variants
|                      | Decryption oracle available     |
| -------------------- | ------------------------------- |
| **CCA1** (Lunchtime) | Only **before** you see C*      |
| **CCA2** (Adaptive)  | **Before and after** you see C* |
CCA1 is called "lunchtime attack" — imagine the attacker gets 30 minutes alone with the decryption machine, then it's taken away. CCA2 is more powerful — the attacker keeps access throughout.

**CCA2 is the one that matters in practice.**

#### One Line Summary

|          | Simple version                                                                  |
| -------- | ------------------------------------------------------------------------------- |
| **CPA**  | Can you break it if you can encrypt freely?                                     |
| **CCA1** | Can you break it if you can also decrypt freely, but only before the challenge? |
| **CCA2** | Can you break it if you can decrypt freely, even after seeing the challenge?    |
**CCA2 is the gold standard. AES-GCM achieves it. Always use AEAD.**

## AES-GCM
---
AES-GCM is an authenticated encryption mode that provides both confidentiality and integrity.

> AES-GCM not only encrypts your data, but also ensures that it hasn’t been modified. If someone tampers with the ciphertext, decryption fails.

```
AES-GCM  =  AES-CTR  (confidentiality)
          + GHASH    (authentication/integrity)
```

#### What Problem Are We Solving?
Encryption alone is **not enough**.

#### ❌ Problem with plain encryption
An attacker can:
- Modify ciphertext
- Flip bits
- Inject data

👉 You might decrypt **tampered data without knowing**

### Goal of AES-GCM
We want:
- 🔒 Confidentiality → hide data
- ✅ Integrity → detect changes
- 🔑 Authentication → ensure trusted origin

This is exactly what **AEAD** provides.

### How AES-GCM Is Built

AES-GCM is not “one thing” — it's a **composition of two ideas**:

#### Part 1: AES in CTR Mode (Encryption)
AES is a **block cipher**, but GCM uses it like a **stream cipher**.
##### Core idea:
Instead of encrypting plaintext directly:
- Generate a **keystream**
- XOR it with plaintext
##### Conceptually:
```
Keystream Block = AES(key, nonce || counter)
Ciphertext = Plaintext ⊕ Keystream
```

##### Why this is powerful
- Same AES primitive, but now:
    - Works on arbitrary length data
    - Fully **parallelizable** — each block is independent
    - Very fast

👉 CTR mode turns AES into a **pseudorandom generator**

#### Part 2 — GHASH (The Authentication Half)
GHASH is a **polynomial MAC** (Message Authentication Code) computed over:

- The **ciphertext** (not plaintext)
- **AAD** — Additional Authenticated Data (headers, metadata you want authenticated but not encrypted)
- Their **lengths**

#### What is AAD?
AAD is data that doesn't need to be encrypted but must not be tampered with. Example: HTTP headers, packet routing info, a user ID. You want it readable but protected from modification.

##### Conceptually:
Think of GHASH like:

> “Take all data → mix it mathematically → produce a fingerprint”
>  Any change to AAD, ciphertext, or lengths produces a completely different tag.

#### Putting It Together

##### Encryption Flow
```
1. Generate Keystream (AES-CTR)
2. XOR with plaintext -> cipertext
3. Feed ciphertext + AAD -> GHASH
4. Produce Authentication Tag
```

Decryption Flow
```
1. Recompute GHASH
2. Verify tag
3. If tag is valid -> decrypt using CTR
4. If invalid → reject everything
```
👉 **Authentication is mandatory before trusting output**

#### The Role of Nonce (CRITICAL)
AES-GCM uses a **96-bit nonce** (recommended). It must be **unique per encryption** under the same key. It does not need to be secret.

Nonce = “number used once”
##### Why nonce exists?
It ensures:
- Every encryption produces a **different keystream**

#### What happens if nonce is reused?
This is the most important AES-GCM property.
##### ❌ Keystream reuse

`C1 ⊕ C2 = P1 ⊕ P2`

👉 attacker learns relationship between plaintexts

##### ❌ Tag forgery
- GHASH becomes predictable
- Attacker can forge valid messages

> **Nonce reuse completely breaks AES-GCM**
    Not “weakens” — **breaks**

#### Security Guarantees
AES-GCM provides:

##### 🔐 IND-CPA (confidentiality)
Attacker cannot distinguish encryptions
##### 🔐 IND-CCA (authenticated encryption)
Even if attacker:
- modifies ciphertext
- injects chosen inputs

👉 They **cannot produce valid forgery**

#### Tag Size
The auth tag is 128 bits by default. Some protocols truncate it (e.g. to 96 or 64 bits). Shorter tags:

- Reduce security margin against forgery
- 64-bit tag → attacker has 1/2⁶⁴ chance of forging — borderline acceptable only in constrained environments
- **Always use 128-bit tags if possible**

#### Why AES-GCM Is So Fast
**Reasons:**
- CTR mode → parallelizable. (Counter 1, 2, 3... are all known **before any encryption happens**. So we can compute all keystream blocks **simultaneously** — on separate CPU cores, in separate threads, or in hardware pipelines)
- GHASH → hardware optimized
- CPUs support AES instructions (AES-NI)

👉 That’s why:

- TLS uses it
- Cloud systems use it

### Limitations
1. **Nonce misuse fragile**

 If you encrypt two different messages with the **same key + same nonce**:

```
C1 = P1 XOR Keystream
C2 = P2 XOR Keystream   ← identical keystream

C1 XOR C2 = P1 XOR P2
```

The attacker XORs the two ciphertexts and gets the **XOR of the two plaintexts**. With some known plaintext (which is common — HTTP headers, file format magic bytes), they can recover both messages fully.

This is called a **two-time pad** — a reuse of a one-time pad, which completely breaks it.

> **Rule: One key + one nonce = must only ever encrypt one message. Ever.**

2.  **Counter Overflow / Wrap-around**

The counter is finite. If you encrypt enough data under one key+nonce pair, the counter wraps back to 0 — and you're reusing keystream blocks.

For AES-GCM's 32-bit counter with 96-bit nonce:

```
Max blocks = 2³²  =  ~4 billion blocks
           = ~68 GB per nonce
```

Encrypting more than this under one key+nonce **reuses keystream** — same consequence as nonce reuse.

3. **Authentication Tag Truncation Weakens Security**

GCM commonly uses a 128-bit tag.
If shortened (e.g., 64-bit or less):
- higher forgery probability
- more dangerous at scale

Short tags may be acceptable in constrained environments but require careful limits.

4.  **Less Forgiving Than ChaCha20-Poly1305 on Some Devices**
On systems without AES hardware acceleration:

- AES-GCM may be slower
- more battery-intensive
- harder to implement efficiently than ChaCha20-Poly1305

4. **Limited Number of Encryptions Per Key**

Even with unique nonces, security degrades after many encryptions under one key.

There are bounds on:
- Number of messages
- Total blocks encrypted

At large scale (high-volume systems), keys should be rotated.

### AES-GCM Security Properties Summary

| Property        | Provided by                           |
| --------------- | ------------------------------------- |
| Confidentiality | AES-CTR                               |
| Integrity       | GHASH tag                             |
| Authentication  | GHASH tag                             |
| AAD protection  | GHASH tag                             |
| IND-CPA         | Randomized nonce → unique keystream   |
| IND-CCA2        | Auth tag rejects modified ciphertexts |

> “AES-GCM is an AEAD construction that combines AES in counter mode for confidentiality with a polynomial hash (GHASH) for authentication. It encrypts data by generating a keystream and XORing it with plaintext, while simultaneously computing an authentication tag over both ciphertext and associated data. It provides IND-CCA security under the assumption that nonces are never reused, but fails catastrophically if nonce reuse occurs.”

## XChaCha20-Poly1305
---
> **XChaCha20-Poly1305 = ChaCha20 (stream cipher) + Poly1305 (MAC) with an extended nonce**

It’s an **AEAD construction**, just like AES-GCM, but with a _different philosophy_:
- No block cipher
- No GHASH
- Pure software-friendly design

```
XChaCha20-Poly1305  =  XChaCha20     (confidentiality)
                     + Poly1305      (authentication/integrity)
```

### Part 1 — ChaCha20 (The Core)
ChaCha20 is a **stream cipher** — it generates a keystream and XORs with plaintext. Just like CTR mode conceptually, but the keystream generation is completely different.

It operates on a **4×4 matrix of 32-bit words** (512 bits = 64 bytes total state):
- **Constants** — 4 fixed words (hardcoded, not secret)
- **Key** — 256 bits (8 words)
- **Counter** — 32 or 64 bits
- **Nonce** — 96 bits (3 words) in ChaCha20

##### The Quarter Round — Core Operation
ChaCha20's mixing is built from a **quarter round** — 4 simple operations on 4 words (a, b, c, d):

```
a += b;  d ^= a;  d <<<= 16;
c += d;  b ^= c;  b <<<= 12;
a += b;  d ^= a;  d <<<= 8;
c += d;  b ^= c;  b <<<= 7;
```

Just **Add, XOR, Rotate** — repeated. No lookup tables, no S-boxes, no branches. This is called an **ARX cipher** (Add-Rotate-XOR).

##### The 20 Rounds
ChaCha**20** means **20 rounds** of mixing. Each round applies quarter rounds to columns then diagonals of the 4×4 matrix:

After 20 rounds of mixing, the state is heavily scrambled. But here's the issue — the quarter round operations (Add, XOR, Rotate) are **all reversible**:

- Addition → reverse with subtraction
- XOR → reverse with XOR again
- Rotation → reverse with opposite rotation

So if someone gets the final scrambled state, they could **work backwards** through all 20 rounds and recover the key.

#### The Fix — Add the Original State
Before the 20 rounds start, ChaCha20 **saves a copy** of the initial state. After all 20 rounds, it adds that original copy back word by word:
```
Initial state (save a copy):
[ C C C C ]
[ K K K K ]
[ K K K K ]
[ N N ctr ]

        ↓  20 rounds of mixing

Scrambled state:
[ X X X X ]
[ X X X X ]
[ X X X X ]
[ X X X X ]

        ↓  ADD original state back (word by word)

Final keystream block:
[ X+C  X+C  X+C  X+C  ]
[ X+K  X+K  X+K  X+K  ]
[ X+K  X+K  X+K  X+K  ]
[ X+N  X+N  X+ctr X+N ]
```

#### Keystream Generation
```
Initial State (Key + Nonce + Counter=0)
        ↓
   20 rounds of mixing
        ↓
   Add original state
        ↓
  64 bytes of keystream block 0

Initial State (Key + Nonce + Counter=1)
        ↓
   20 rounds of mixing
        ↓
  64 bytes of keystream block 1
```

Each counter value produces an independent keystream block — **fully parallelizable**, same as CTR mode.
```
Ciphertext = Plaintext XOR Keystream
```

---
## Part 2 — ChaCha20 vs XChaCha20
This is the key difference to understand.
##### The Problem with ChaCha20's Nonce
ChaCha20 uses a **96-bit nonce**. That's fine for a single session, but:

- If you generate nonces **randomly**, 96 bits gives you ~2⁴⁸ encryptions before a collision becomes likely (birthday bound)
- For a high-volume system encrypting millions of messages, this is a **real risk**
- Managing counters to guarantee uniqueness across distributed systems is operationally hard

> 96-bit random nonce = not safe for large scale random nonce generation

#### XChaCha20 — Extended Nonce
XChaCha20 extends the nonce to **192 bits** — more than double. Now random nonce generation is safe even at massive scale (2⁹⁶ messages before collision risk).

**But how?** You can't just stuff 192 bits into ChaCha20's state — there's no room.

##### HChaCha20 — The Bridge
XChaCha20 solves this with an intermediate step called **HChaCha20**:

```
Step 1: Take first 128 bits of the 192-bit nonce
        Feed into HChaCha20 with the key
        Output: a new 256-bit subkey

Step 2: Use that subkey as the actual ChaCha20 key
        Use remaining 64 bits of nonce as ChaCha20 nonce
        Proceed normally
```

```
192-bit nonce:  [ Nonce Part 1 (128 bits) | Nonce Part 2 (64 bits) ]
                         ↓
                   HChaCha20(Key, Nonce Part 1)
                         ↓
                    Subkey (256 bits)
                         ↓
              ChaCha20(Subkey, Nonce Part 2, Counter)
                         ↓
                      Keystream
```
HChaCha20 is just the ChaCha20 core function without the final addition step — it's a **key derivation step**, not encryption.
#### Why this works
The 128-bit first half of the nonce is mixed into the key itself via HChaCha20. So effectively the entire 192-bit nonce influences the keystream — just through key derivation rather than direct state injection.

#### ChaCha20 vs XChaCha20 — Summary

| |ChaCha20|XChaCha20|
|---|---|---|
|Nonce size|96 bits|192 bits|
|Safe for random nonces?|Risky at scale|✅ Yes|
|Extra step|None|HChaCha20 key derivation|
|Performance|Baseline|Negligibly slower (one HChaCha20 call)|
|Use case|Short sessions, counter-based nonces|Long-lived keys, random nonces, large scale|

---
### Part 3 — Poly1305 (The Authentication Half)
ChaCha20 only gives **confidentiality**.

We still need:
- Integrity
- Authentication

Poly1305 is a **one-time MAC(MESSAGE AUTHENTICATION CODE)** — it computes an authentication tag over the ciphertext and AAD.
```
tag = Poly1305(key, message)
```

##### How it works (conceptually)
- Treat message as numbers
- Evaluate polynomial modulo:
```
2^130 - 5
```

👉 Very fast and secure

##### Critical Property
> Poly1305 key must NEVER be reused

##### XChaCha20-Poly1305 Combined
```
XChaCha20-Poly1305

Key + 192-bit Nonce
        │
        ├──→ HChaCha20 ──→ Subkey
        │                     │
        │         ┌───────────┤
        │         ↓           ↓
        │    ChaCha20      ChaCha20
        │    (counter=0)   (counter=1,2,3...)
        │         │               │
        │    first 32 bytes    Keystream
        │         │               │
        │    Poly1305 key      XOR Plaintext
        │         │               │
        └─────────┤           Ciphertext
                  ↓               │
              Poly1305 ←──────────┘
              (over Ciphertext + AAD)
                  │
              Auth Tag (128 bits)

Output: [ Ciphertext ] + [ 128-bit Tag ]
```

#### Decryption & Verification

Identical principle to AES-GCM:
```
1. Recompute Poly1305 tag from received ciphertext + AAD
2. Constant-time compare with received tag
3. Mismatch → reject entirely, no decryption
4. Match → XChaCha20 decrypt
```

#### Why XChaCha20 is Powerful
##### ✅ 1. Huge nonce space
- 192-bit nonce
- Practically impossible to collide

👉 Safe for:
- distributed systems
- random nonce generation

##### ✅ 2. Misuse-resistant (practically)

Not fully misuse-proof, but:

> Random nonce reuse probability becomes negligible

##### ✅ 3. No need for strict counters

Unlike GCM:
- You don’t need global coordination

##### ✅4. Pure software performance
- No hardware dependency

##### ✅ 5.Constant-time
- Avoids timing attacks
### Limitations

| Limitation                               | Detail                                                                                                    |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Nonce reuse                              | Same catastrophe as CTR — keystream reuse + Poly1305 key exposed → full break                             |
| No random access penalty                 | Technically parallelizable but Poly1305 is sequential over ciphertext                                     |
| Poly1305 is one-time                     | r(KEY) must never repeat — handled automatically if nonce is unique                                       |
| Software-only speed advantage disappears | On hardware with AES-NI, AES-GCM is faster — ChaCha20 shines on devices without AES hardware acceleration |
##### Think of XChaCha20-Poly1305 as:
> 🔧 A system that:
> 
> - Uses a large random nonce to derive a fresh encryption context
> - Encrypts data as a stream
> - Authenticates everything with a polynomial MAC


> “XChaCha20-Poly1305 is an AEAD construction that extends ChaCha20-Poly1305 by using a 192-bit nonce. It derives a subkey using HChaCha20 from part of the nonce, and then performs standard ChaCha20 encryption with Poly1305 authentication. This design enables safe use of random nonces and avoids the strict nonce management requirements of traditional AEAD schemes, while providing IND-CCA security.”


## AES-GCM vs XChaCha20-Poly1305 — Real World Tradeoffs
---
### The Fundamental Difference in Design Philosophy

```
AES-GCM              →  Designed for HARDWARE
XChaCha20-Poly1305   →  Designed for SOFTWARE
```

This single fact explains almost every tradeoff below.

| Dimension                    | **AES-GCM**                                           | **XChaCha20-Poly1305**                            |
| ---------------------------- | ----------------------------------------------------- | ------------------------------------------------- |
| **Core primitive**               | Block cipher (AES-CTR + GHASH)                        | Stream cipher (ChaCha20 + Poly1305)               |
| **Nonce size**                   | 96-bit (standard)                                     | 192-bit                                           |
| **Nonce requirement**            | **Strictly unique**                                   | Unique, but random is practically safe            |
| **Failure on nonce reuse**       | **Catastrophic** (breaks confidentiality + integrity) | Also bad, but collision probability is negligible |
| **Performance (desktop/server)** | **Very fast** with AES-NI                             | Good, but often slightly slower                   |
| **Performance (mobile/IoT)**     | Can be slower                                         | **Often faster and more stable**                  |
| **Constant-time behavior**       | Depends on hardware/impl                              | **Naturally constant-time**                       |
| **Parallelization**              | Excellent (CTR + GHASH)                               | Good, but less parallel                           |
| **Ease of safe use**             | **Harder** (nonce discipline required)                | **Easier** (large nonce space)                    |
| **Standardization**              | NIST, TLS, widely mandated                            | Not NIST, but widely adopted                      |
| **Typical use**                  | TLS, cloud infra, databases                           | libsodium, messaging, mobile, crypto libs         |

### 🧠 Tradeoff #1: **Nonce Management (The Biggest One)**

#### AES-GCM

- Requires **strict nonce uniqueness per key**
- You _must_ ensure:
    - No reuse
    - No collisions
    - Often requires counters or coordination

##### 💣 If you fail:
- Keystream reuse → plaintext leaks
- GHASH becomes forgeable → integrity breaks

👉 This is a **fragile invariant**

---

#### XChaCha20-Poly1305

- 192-bit nonce
- Designed to safely allow **random nonces**

###### Why this matters:
- Collision probability is astronomically low
- No global coordination needed

👉 This is a **robust invariant**

---

### 🧠 Real-world implication

| Scenario                         | Better choice |
| -------------------------------- | ------------- |
| Distributed microservices        | **XChaCha20** |
| Multi-region systems             | **XChaCha20** |
| Centralized system with counters | AES-GCM       |

---
### ⚡ Tradeoff #2: **Performance Reality**

#### AES-GCM

###### 🚀 Extremely fast when:
- CPU supports AES-NI (Intel/AMD)
- Hardware acceleration available

👉 That’s why:
- TLS prefers AES-GCM on servers

---
#### XChaCha20-Poly1305

###### 🧠 Designed for:
- Consistent performance across devices
- No reliance on hardware

👉 Often faster on:

- Mobile devices
- Embedded systems
- Older CPUs

---
### ⚠️ Key insight

> AES-GCM is **conditionally fast**  
> XChaCha20 is **predictably fast**


---
### 🔒 Tradeoff #3: **Side-Channel Resistance**

#### AES-GCM

- Historically vulnerable if:
    - No AES-NI
    - Table-based implementations used

👉 Modern CPUs mitigate this

---
#### XChaCha20

- Uses:
    - Addition
    - XOR
    - Rotation

👉 Naturally:

- Constant-time
- Side-channel resistant

---
#### 🧠 Takeaway

> XChaCha20 is **safer by design** in hostile environments

---
### 🧩 Tradeoff #4: **Parallelization**

#### AES-GCM

- CTR mode → parallel encryption
- GHASH → parallelizable

👉 Excellent for:
- High-throughput systems
- Bulk encryption

---
#### XChaCha20
- Stream-based → less parallel-friendly
- Still fast, but different scaling

---
### 💣 Tradeoff #5: **Failure Modes**

This is _huge_ and often ignored.

---

#### AES-GCM Failure Mode

> **Brittle**

Single nonce reuse:

- breaks confidentiality
- enables forgeries

👉 “One mistake = total failure”

---
#### XChaCha20 Failure Mode

> **Resilient in practice**

- Still unsafe if reused
- BUT:
    - Random nonce collision ≈ impossible

👉 “Hard to misuse accidentally”

> “AES-GCM offers superior performance on hardware with AES acceleration and is widely standardized, but requires strict nonce uniqueness and fails catastrophically if misused. XChaCha20-Poly1305 trades some peak performance for a larger nonce space, enabling safe random nonce usage and making it more robust in distributed systems. In practice, AES-GCM is preferred in controlled environments like TLS, while XChaCha20-Poly1305 is favored in systems where nonce management is difficult or misuse resistance is important.”


## RSA (Asymmetric Encryption)
---
RSA is an asymmetric encryption algorithm that uses a public key for encryption and a private key for decryption.

> Anyone can encrypt data using your public key, but only you can decrypt it using your private key.

```
Public key → shared
Private key → secret
```

##### Use cases
| Use                | Why                  |
| ------------------ | -------------------- |
| Key exchange       | share secrets        |
| Digital signatures | verify identity      |
| TLS/HTTPS          | secure communication |
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

### 🧠 RSA vs Symmetric (Deep Insight)

|Property|RSA|AES / ChaCha|
|---|---|---|
|Type|Asymmetric|Symmetric|
|Speed|Slow|Fast|
|Key usage|Public/private|Shared|
|Use case|Key exchange, signatures|Bulk encryption|

> “RSA is an asymmetric cryptosystem based on the hardness of integer factorization. It uses a public-private key pair where encryption is performed using modular exponentiation with the public key, and decryption uses the private key. In practice, RSA is used with padding schemes like OAEP for encryption and PSS for signatures, and is primarily used for key exchange and authentication rather than bulk data encryption.”


## Message Authentication Codes (MACs)
---
A **MAC** is a short tag attached to a message that proves two things simultaneously:
- **Integrity** — the message has not been tampered with
- **Authenticity** — the message came from someone who holds the shared secret key

#### The Core Problem
Suppose you send:
```
message → "Transfer ₹10,000"
```

An attacker can:
- Modify it → `"Transfer ₹1,00,000"`
- Forward it

👉 Encryption alone **does not prevent tampering**

Unlike a hash, a MAC requires a secret key. Unlike a digital signature, it uses a _symmetric_ key — the same key both generates and verifies the tag.
```
MAC(key, message) → tag

Sender:   tag = MAC(k, m)   →  sends (m, tag)
Receiver: tag' = MAC(k, m)  →  checks tag == tag'
```

#### Security Goal: Existential Unforgeability (EU-CMA)
The formal security definition is **Existential Unforgeablility under Chosen Message Attack.**

An adversary wins if they can produce a valid `(m, tag)` pair for _any_ message `m` — even one they chose — without knowing the key. The adversary is allowed to query a MAC oracle polynomially many times. If no adversary can win with non-negligible probability, the MAC is secure.

This is a strong guarantee: it rules out forgery even when the attacker can adaptively choose messages and observe their tags.

#### Why Hashing Alone Fails
---
#### 🔑 First, how SHA-256 actually works internally

SHA-256 uses a construction called **Merkle-Damgård**. It doesn't hash your message all at once — it processes it in **512-bit chunks**, feeding each chunk into a compression function along with the previous output:

```
Initial State (IV)
      │
      ▼
┌─────────────┐
│  compress   │ ◄── chunk 1 of message
└─────────────┘
      │
      ▼
┌─────────────┐
│  compress   │ ◄── chunk 2 of message
└─────────────┘
      │
      ▼
   Final Hash
```

**Critical detail:** The final hash output _is_ the internal state after the last compression. It's not sealed or finalized in a way that prevents resumption.

##### The Length Extension Attack
Imagine you send a message and a tag:
- **Message:** "Buy 10 apples"
    
- **Key:** (Secret, you don't know it)
    
- **Tag:** $SHA256(Secret \parallel \text{"Buy 10 apples"})$

As an attacker, I see the tag `abcd123...`. Because of how SHA-256 works, I know that `abcd123...` is the **exact internal state** the hash function was in after processing our message.

I don't need the secret key to continue grinding! I can:
1. Take that tag (`abcd123...`) and manually set it as the "starting state" of my own hash function.
    
2. Add my own data: " and 100 oranges".
    
3. Tell the hash function to keep grinding.

The result is a valid tag for $(Secret \parallel \text{"Buy 10 apples"} \parallel \text{" and 100 oranges"})$, and I did it all without ever knowing what the `Secret` was.

#### Consequences of a **Length Extension Attack**
###### 1. Unauthorized Command Injection
In the early days of the web, many APIs used simple keyed hashes for authentication.
- **Original Request:** `user_id=10&action=view_profile&signature=abcd...`
    
- **The Attack:** An attacker sees this and appends `&action=delete_account` or `&is_admin=true`.
- **The Result:** The server receives the long string, sees a "valid" signature at the end (because the attacker re-calculated it), and processes the malicious command. The server thinks, "Well, the signature matches the data, so the person with the secret key must have sent this!"

##### 2. File and Data Corruption
If a system uses keyed hashes to verify the integrity of a downloaded file (like a software update), an attacker could:
- Append malicious code or a "backdoor" to the end of a legitimate file.
    
- Update the hash tag using length extension.
    
- The installer checks the hash, sees it matches, and runs the malicious code.

##### 3. Financial Fraud
Imagine a banking instruction sent via a keyed hash:

- **Message:** `from=Alice&to=Bob&amount=100`
    
- **Attacker appends:** `&amount=5000`
    
- In many parsing systems (like URL query strings), if a variable is defined twice, the **last one** wins. The server might see two "amount" fields and only process the second one, effectively stealing $4,900 more than Alice intended.

### Core Constructions
---
#### 1. HMAC (Hash-based MAC)

HMAC, or **Hash-based Message Authentication Code**, is a specific type of message authentication code (MAC) that involves a cryptographic hash function and a secret cryptographic key.

Think of it as a **digital wax seal**. It doesn't just prove that the message hasn't been tampered with (integrity); it also proves that the person who sent it is who they claim to be (authenticity), because only someone with the secret key could have created that specific seal.

###### How It Works
Unlike a simple hash (like SHA-256), which anyone can generate for any piece of data, an HMAC requires a **Secret Key**. If the data or the key changes by even a single bit, the resulting HMAC will be completely different.

#### The Two-Pass Process
HMAC is more than just hashing a key and a message together. It uses a "nested" hashing approach to protect against certain types of cryptographic attacks (like length-extension attacks).

1. **Inner Hash:** The key is mixed with an "inner padding" (ipad) and hashed along with the message.
    
2. **Outer Hash:** The result of the first hash is then mixed with the key and an "outer padding" (opad) and hashed again.

The mathematical representation looks like this:

$$HMAC(K, m) = H((K \oplus opad) \parallel H((K \oplus ipad) \parallel m))$$
##### Why Use HMAC?
- **Integrity:** You can be sure the data wasn't altered in transit.
    
- **Authenticity:** You can be sure the sender knows the secret key.
    
- **Efficiency:** It is much faster than digital signatures (like RSA) because it relies on symmetric cryptography rather than complex asymmetric math.

##### Common Use Cases
- **API Security:** When you make a request to an API (like AWS or Stripe), you often sign the request with an HMAC so the server knows it’s really you.
    
- **JWT (JSON Web Tokens):** HMAC is frequently used to sign tokens to ensure they haven't been forged by a user.
    
- **Challenge-Response:** Used in protocols to verify a user's identity without ever sending their actual password over the network.


HMAC is the "sturdy tractor" of the MAC world—reliable and everywhere. But in the world of high-speed web traffic and mobile devices, we need something more like a "Formula 1 engine."

This is where **AEAD (Authenticated Encryption with Associated Data)** comes in. Instead of you manually combining a cipher and a MAC (which is where many developers make mistakes), AEAD modes like **AES-GCM** and **ChaCha20-Poly1305** do both at once

#### 2. GMAC (used in AES-GCM)

The "MAC" part of AES-GCM is actually called **GHASH**. When used on its own, it’s called **GMAC**.
##### How it works:

It uses "Galois Field" math (a type of binary polynomial arithmetic). It takes the ciphertext, breaks it into blocks, and multiplies them together in a very specific mathematical field ($GF(2^{128})$).

- **The Big Advantage:** **Parallelism.** Unlike HMAC, which has to process data bit-by-bit in order, GMAC can be "unrolled." Modern CPUs (Intel and AMD) have a special instruction called **CLMUL** specifically designed to do this math instantly.
    
- **The Catch:** It is notoriously difficult to implement safely in software. If you don't have that specific hardware acceleration, it is slow and vulnerable to **timing attacks** (where an attacker guesses your key by measuring how many milliseconds the math took).

#### Poly1305 (used in ChaCha20-Poly1305)

Poly1305 is the "software-optimized" alternative to GMAC.
##### How it works:

It is a **One-Time Authenticator**. It uses prime-field math ($2^{130} - 5$) to calculate the tag.

- **The Big Advantage:** **Speed in software.** Poly1305 doesn't need special CPU instructions to be fast. It runs beautifully on mobile phones, older computers, and IoT devices that don't have dedicated AES hardware.
    
- **The Catch:** It is a "one-time" MAC. For every single message, a new sub-key must be generated (usually by the ChaCha20 cipher). If you ever use the same Poly1305 key for two different messages, **the security is instantly broken** and an attacker can forge messages.

#### The Great Showdown: GMAC vs. Poly1305
|**Feature**|**GMAC (AES-GCM)**|**Poly1305 (ChaCha20-Poly1305)**|
|---|---|---|
|**Primary Strength**|Blazing fast on servers (with AES-NI).|Blazing fast on mobile/software.|
|**Math Style**|Binary Polynomials ($GF(2^{128})$).|Prime Field Arithmetic ($2^{130}-5$).|
|**Vulnerability**|Timing attacks (if no hardware).|Nonce reuse (catastrophic failure).|
|**Best For**|Data centers, high-end laptops.|Smartphones, Smartwatches, VPNs (WireGuard).|

#### Why did we move away from HMAC in these cases?

1. **Performance:** HMAC requires two passes over the data (inner and outer hash). AEAD MACs like GMAC and Poly1305 are designed to process the data in **one pass** as it is being encrypted.
    
2. **Atomicity:** By using an AEAD mode, you don't have to worry about whether to "Encrypt-then-MAC" or "MAC-then-Encrypt." The algorithm handles the binding between the ciphertext and the tag for you, eliminating a huge class of cryptographic errors.

#### The "Golden Rule" of AEAD MACs

Both of these rely on a **Nonce** (a number used only once).
- In **HMAC**, if you reuse a key, it’s not ideal, but it’s not an immediate disaster.
    
- In **AES-GCM** or **ChaCha20-Poly1305**, if you reuse a **Nonce + Key** combination, the attacker can mathematically derive the authentication key and forge any message they want.

> “A Message Authentication Code (MAC) is a cryptographic primitive that provides data integrity and authentication using a shared secret key. It ensures that a message has not been modified and originates from a trusted source. Secure MAC constructions like HMAC prevent attacks such as length extension, while modern systems integrate MACs into AEAD schemes like AES-GCM and ChaCha20-Poly1305 to provide authenticated encryption with strong security guarantees.”


### KEK/DEK Pattern — Key Wrapping & Envelope Encryption
---
##### The Core Idea
You never encrypt data directly with your master key. Instead you maintain a two-layer hierarchy:

```
Master Secret
     ↓
    KEK  ─────────────┐
     ↓                │
 [wraps]              │
     ↓                │
    DEK  ──encrypt──→ Data
```

The KEK's **only job** is to wrap DEKs. It never sees plaintext data. The DEK's **only job** is to encrypt one logical item, then it's discarded from memory.

##### Why Not Just Encrypt Everything With the KEK?
This is the central question. The answer is multi-dimensional:

##### 1. Key Rotation Becomes Cheap
If you encrypt 10 million items with the KEK directly, rotating the KEK means **re-encrypting 10 million items**. With envelope encryption:
```
Old KEK → re-wrap 10 million small DEKs  (fast, DEK re-encrypt is trivial)
New KEK → done

vs.

Old KEK → decrypt + re-encrypt 10 million blobs  (expensive, risky)
```

You only touch the tiny DEK blobs (~32 bytes each), not the actual data.

##### 2. Blast Radius of Compromise
If a single DEK is compromised, only **one item** is exposed. If the KEK is compromised, you rotate it and re-wrap DEKs — the actual plaintext data was never directly accessible through the KEK alone. An attacker needs both the wrapped DEK **and** the KEK to decrypt anything.

#### Cryptographic Hygiene — Key/Data Separation
Every cryptographic key has a recommended **usage limit** before you should rotate. AES-GCM with random nonces, for example, risks nonce collision beyond ~2³² encryptions under the same key (birthday bound). By giving each item its own DEK:

- Each DEK encrypts exactly one item → zero collision risk
- The KEK only performs `N_items` wrap operations, which is far fewer than the data volume

##### 4. Access Control Granularity

You can implement **per-user or per-item key access** without multiple copies of the data:

```
Item ciphertext  (one copy)
     │
     ├── wrappedDek_userA  (KEK_A wraps the DEK)
     ├── wrappedDek_userB  (KEK_B wraps the same DEK)
     └── wrappedDek_admin  (KEK_admin wraps the same DEK)
```

Revoke userA → delete their wrapped DEK. The ciphertext and other users are untouched.

```ts
export function encryptVaultItem(input: {
  plaintext: string;
  kek: Uint8Array;
  aead: XChaCha20Poly1305Aead;
  associatedData?: Uint8Array;
}): EncryptedVaultItem {
  if (input.kek.length !== input.aead.keyLength) {
    throw new Error("kek length does not match cipher key length");
  }

  const itemDek = randomBytes(input.aead.keyLength);
  try {
    const nonce = randomBytes(input.aead.nonceLength);
    const wrapNonce = randomBytes(input.aead.nonceLength);

    const ciphertext = input.aead.encrypt({
      key: itemDek,
      nonce,
      plaintext: utf8ToBytes(input.plaintext),
      associatedData: input.associatedData,
    });

    const wrappedDek = input.aead.encrypt({
      key: input.kek,
      nonce: wrapNonce,
      plaintext: itemDek,
      associatedData: utf8ToBytes(DEK_WRAP_AAD),
    });

    return {
      version: "xchacha20poly1305-v1",
      nonce: toBase64(nonce),
      ciphertext: toBase64(ciphertext),
      wrappedDek: toBase64(wrappedDek),
      wrapNonce: toBase64(wrapNonce),
    };
  } finally {
    itemDek.fill(0);
  }
}

export function decryptVaultItem(input: {
  payload: EncryptedVaultItem;
  kek: Uint8Array;
  aead: XChaCha20Poly1305Aead;
  associatedData?: Uint8Array;
}): string {
  if (input.payload.version !== "xchacha20poly1305-v1") {
    throw new Error("unsupported vault item version");
  }
  if (input.kek.length !== input.aead.keyLength) {
    throw new Error("kek length does not match cipher key length");
  }
  if (!input.payload.nonce || !input.payload.wrapNonce || !input.payload.ciphertext || !input.payload.wrappedDek) {
    throw new Error("payload is missing required encrypted fields");
  }

  const wrapNonce = decodeBase64Field(input.payload.wrapNonce, "wrapNonce", input.aead.nonceLength);
  const wrappedDek = decodeBase64Field(input.payload.wrappedDek, "wrappedDek");
  const nonce = decodeBase64Field(input.payload.nonce, "nonce", input.aead.nonceLength);
  const ciphertext = decodeBase64Field(input.payload.ciphertext, "ciphertext");

  const itemDek = input.aead.decrypt({
    key: input.kek,
    nonce: wrapNonce,
    ciphertext: wrappedDek,
    associatedData: utf8ToBytes(DEK_WRAP_AAD),
  });
  if (itemDek.length !== input.aead.keyLength) {
    throw new Error("wrapped DEK has unexpected key length");
  }

  const plaintext = input.aead.decrypt({
    key: itemDek,
    nonce,
    ciphertext,
    associatedData: input.associatedData,
  });

  return bytesToUtf8(plaintext);
}
```


### 🔐 Encryption — `encryptVaultItem`
```
plaintext  +  kek  +  aead
      │
      ▼
① itemDek = randomBytes(32)
      │         └── born here, lives only in memory
      │
      ▼
② nonce     = randomBytes(24)    ← for data layer
  wrapNonce = randomBytes(24)    ← for key wrap layer
  (both are public, not secret)
      │
      ▼
③ LAYER 1 — Encrypt data with DEK
  ciphertext = aead.encrypt({
      key:   itemDek,
      nonce: nonce,
      pt:    utf8ToBytes(plaintext),
      aad:   input.associatedData
  })                              ← KEK never touches data
      │
      ▼
④ LAYER 2 — Wrap DEK with KEK
  wrappedDek = aead.encrypt({
      key:   kek,
      nonce: wrapNonce,
      pt:    itemDek,             ← DEK is plaintext here
      aad:   "dek-wrap-v1"       ← domain separation
  })
      │
      ▼
⑤ itemDek.fill(0)                ← zeroed in finally block,
      │                             even if exception thrown
      ▼
⑥ return {
      version:    "xchacha20poly1305-v1",
      nonce:      base64(nonce),       ← public
      ciphertext: base64(ciphertext),  ← data encrypted by DEK
      wrappedDek: base64(wrappedDek),  ← DEK encrypted by KEK
      wrapNonce:  base64(wrapNonce),   ← public
  }
```

### 🔓Decryption — `decryptVaultItem`
```
payload  +  kek  +  aead
      │
      ▼
① Version check
  payload.version !== "xchacha20poly1305-v1"  → throw
  any missing field (nonce/ciphertext/etc)    → throw
      │
      ▼
② Decode base64 fields → raw bytes
  wrapNonce  = decodeBase64(payload.wrapNonce)
  wrappedDek = decodeBase64(payload.wrappedDek)
  nonce      = decodeBase64(payload.nonce)
  ciphertext = decodeBase64(payload.ciphertext)
      │
      ▼
③ LAYER 1 — Unwrap DEK using KEK
  itemDek = aead.decrypt({
      key:   kek,               ← re-derived: Argon2id(passphrase, salt)
      nonce: wrapNonce,
      ct:    wrappedDek,
      aad:   "dek-wrap-v1"     ← must match encrypt-time AAD exactly
  })
  ↑ Poly1305 tag verified here
  wrong passphrase → wrong KEK → tag mismatch → THROW ❌
  correct passphrase → itemDek recovered ✓
      │
      ▼
④ itemDek.length !== 32  →  throw   ← corrupted payload guard
      │
      ▼
⑤ LAYER 2 — Decrypt ciphertext with DEK
  plaintext = aead.decrypt({
      key:   itemDek,
      nonce: nonce,
      ct:    ciphertext,
      aad:   input.associatedData
  })
  ↑ Poly1305 tag verified here
  tampered ciphertext → tag mismatch → THROW ❌
      │
      ▼
⑥ return bytesToUtf8(plaintext)  →  "your original string" ✓
```

#### Where Each Error Fires

| Scenario              | Fails at step | Why                                              |
| --------------------- | ------------- | ------------------------------------------------ |
| Wrong passphrase      | ③             | KEK wrong → Poly1305 tag on `wrappedDek` fails   |
| Tampered ciphertext   | ⑤             | DEK correct but data tag fails                   |
| Tampered `wrappedDek` | ③             | Tag on the wrap blob fails                       |
| Wrong AAD on unwrap   | ③             | AAD is part of tag computation — mismatch = fail |
| Swapped blob types    | ③             | `"dek-wrap-v1"` AAD rejects non-wrap blobs       |
| Missing fields        | ①             | Early guard throws before any crypto runs        |

> “The KEK/DEK pattern, also known as envelope encryption, separates key management from data encryption by using a master key (KEK) to encrypt per-item data keys (DEKs). Each DEK encrypts individual data items, providing compartmentalization and efficient key rotation. In modern systems, AEAD is used both for encrypting data and wrapping DEKs, ensuring confidentiality and integrity. This design limits the impact of key compromise and allows re-keying without re-encrypting all data.”
