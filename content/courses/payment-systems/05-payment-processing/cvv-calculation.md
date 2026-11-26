---
title: "How a CVV is Calculated"
description: "Understanding Card Verification Value generation and validation"
weight: 1
draft: false
---

# Understanding Card Verification Value generation and validation

A Card Verification Value (CVV ) is not a random number, nor is it stored directly in plain text in a database. Instead, it is **calculated using a strict cryptographic algorithm that combines unique card data with secret bank keys**.

Because the process relies on secret keys known only to the issuing bank, it is mathematically impossible to guess or reverse-engineer a CVV just by looking at a card number.

## Types of CVV

| Type          | Acronym of                                                                | Where it is used                                                                                                                    |
| ------------- | ------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `CVV / CVC`   | Classic Card Verification Value (Visa) / Card Validation Code (Mastercard | Magstripe / track 1–2                                                                                                               |
| `iCVV`        | Integrated CVV (from integrated circuit).<br>Also called "chip CVV"       | Chip track equivalent                                                                                                               |
| `CVV2 / CVC2` | Printed on the card back (or front for Amex-style layouts).               | Printed / CNP<br>It is used for e-commerce / MOTO. It is never stored on the magnetic stripe or chip.                               |
| `dCVV`        | Dynamic CVV                                                               | Used in some contactless. card products, it includes the Application Transaction Counter (ATC) making change for each transactiuon. |

## Basic algorithm for CVV calculation

Data required:

* **Card Verification Keys (CVK):** A pair of secret, 128-bit cryptographic keys (referred to as CVK-A and CVK-B) owned strictly by the issuing bank.
* **Primary Account Number (PAN):** The 16-digit card number
* **Expiration Date:** Four digits representing the year and month in YYMM format.
* **Service Code:** A 3-digit code embedded in the card data that specifies the card's rules (e.g., whether it can be used internationally or at ATMs).

Both Visa and MC shares the same DES-based scheme. What changes between them is the service-code field.

**Step 1: Data Concatenation (Padding)**

```c
data = padWith0to32Bytes(pan + yymm + svc)
```
  
The PAN, expiration date (in yymm format), and service code are concatenated into a single continuous block of digits. This string is padded with trailing zeros to make it exactly 32 hex characters long, 

**Step 2: Divide it in block A and block B**

``` c
blockA = data[0..15]
blockB = data[16..31]
```

Split into two distinct 16-character blocks: **Block A** and **Block B**.

**Step 3: Initial Encryption**  

``` c
 temp1 = DES(blockA, cvkA)
```

Block A is encrypted using the secret bank key CVK-A.

**Step 4: XOR and Double Encryption**  

```c
temp2 = DES(temp1 ^ blockB, cvkA)
temp3 = decryptDES(temp2, cvkB)
temp4 = DES(temp3, cvkA)
```

The result from Step 3 is combined with Block B using an XOR (exclusive OR) logic operation. This combined block is then encrypted using CVK-A, and finally decrypted via CVK-B and re-encrypted back through CVK-A ([[The 3DES cycle]]).

**Step 5: Decimalization (Extracting the 3 Digits)**  

```c
cvv  = decimalize(temp4)[0..2]
```

The final encrypted result is a long hexadecimal string containing both numbers (0–9) and letters (A–F). Because a CVV must be numbers only, a **decimalization table** filters the string:

- The algorithm sweeps from left to right, copying all numeric digits (0–9) first.
-  If it hasn't gathered 3 digits by the end, it sweeps through a second time, converting the hex letters (A through F) into numbers (0 through 5) until it has exactly three digits.

***The first three digits resulting from this process become the card's CVV.***

## ## CVV / CVC (strip)

**Input:** PAN + expiry (YYMM) + service code (then pad)
**Service code:** taken from track 2; enters the 3-digit SVC field as-is (e.g. `101`, `201`)

```c
data = padWith0to32Bits(pan + yymm + svc)
```

## CVV2 calculation (eCommerce and MOTO)

**Input:** PAN + expiry + SVC fixed to `000` (then pad)

```c
data = padWith0to32Bits(pan + yymm + '000')
```

## iCVV (chip)

**Input:** PAN + expiry + SVC fixed to `999` (then pad)

```c
data = padWith0to32Bits(pan + yymm + '999')
```

## dCVV (dynamic)

- **Input:** PAN + expiry + SVC fixed to `000` + **ATC** (then pad)
- **Adds ATC:** enter the current counter (padded if required)`

```c
data = padWith0to32Bits(pan + yymm + '000' = atc)
```

For dCVV case MDK is used for the encryption, not the CVK.

Unlike standard CVV calculations that exclusively process unchanging card data, a dCVV calculation introduces a completely unpredictable, moving variable: the **ATC (Application Transaction Counter)**. 

- **The Problem:** If you pass a moving variable through a standard static CVK, the algorithm breaks its intended design principles.
	- Using a static CVK for a dynamic variable forces a **stateless, fixed-input verification key** to do the heavy lifting of a **stateful, variable-input derivation key**
	- To handle a moving variable safely, the architecture _must_ switch to an **MDK**, which is designed from the ground up to safely mix a master secret with an unpredictable counter, deriving a temporary "session key" that isolates each individual transaction from the rest of the network.
	
- **The MDK Solution:** The bank assigns an **MDK** (often explicitly designated as an **MDK-CVC3** or **dCVK Master Key**). When a contactless chip or a banking app generates a dCVV, it uses the changing counter (ATC) to derive a temporary session key from that MDK. That unique session key is what mathematically seals and outputs the temporary 3-digit dCVV.

For the calculation it includes a **previous step - Key Derivation** 

### Deriving the Unique Card Key (UDK)

Before a card ever leaves the factory, the bank must derive a permanent **Unique Card Key (UDK)** that belongs strictly to that specific piece of plastic.

1. **The Inputs:** The bank takes the card's **PAN** (Primary Account Number) and the **PAN Sequence Number** (a 2-digit number used to distinguish multiple cards on the same account).
2. **The Formatting (Padding):** The PAN and sequence number are compressed into a decimal string, right-aligned, and padded with leading zeros to create a fixed-length 16-character hexadecimal block.
3. **The Split Encryption:** Because an MDK is a 128-bit key split into two halves (MDK-A and MDK-B), the bank performs two parallel encryptions:
    - The padded PAN block is encrypted with **MDK-A** to create **Left-UDK**.
    - The padded PAN block is inverted (XORed with `0xFF...`) and encrypted with **MDK-B** to create **Right-UDK**.
4. **The Combine:** Left-UDK and Right-UDK are concatenated together to form the permanent **UDK** stored deep inside your card's secure chip.

### Deriving the Dynamic Session Key (The Live Step)

When you tap your card at a terminal or hit "generate" in an app, the card chip (or the issuing host) takes that permanent **UDK** and derives a temporary **Session Key** using the moving variable.

Let's look at how this happens using the **Application Transaction Counter (ATC)**:

1. **Isolating the Counter:** The chip reads the current ATC (e.g., if this is your 42nd time using the card, the ATC is `0x002A`).
2. **Creating the Derivation Data String:** The chip creates a 16-character hex string. It places the ATC at the very front and fills the rest with alternating patterns or zeros depending on the specific Visa profile (e.g., `002A000000000000`).
3. **The 3DES Session Derivation:** This data string is passed through a 3DES encryption cycle using the permanent **UDK** as the key block:
    - The string is encrypted with UDK-A.
    - The result is decrypted with UDK-B.
    - The result is encrypted again with UDK-A.
        
4. **The Result:** The output of this loop is the **Dynamic Session Key**.

---

[Next: ARQC & ARPC →](/courses/payment-systems/05-payment-processing/arqc-arpc/)
