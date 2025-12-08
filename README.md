# EntropyInputProof

Input proofs with EntropyOracle integration

## 🚀 Standard workflow
- Install (first run): `npm install --legacy-peer-deps`
- Compile: `npx hardhat compile`
- Test (local FHE + local oracle/chaos engine auto-deployed): `npx hardhat test`
- Deploy (frontend Deploy button): constructor arg is fixed to EntropyOracle `0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361`
- Verify: `npx hardhat verify --network sepolia <contractAddress> 0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361`

## 📋 Overview

This example demonstrates **input-proof** concepts in FHEVM with **EntropyOracle integration**:
- What input proofs are
- Why they're required for external encrypted inputs
- How to use them correctly with EntropyOracle
- Security implications
- Entropy-enhanced input validation

## 🎯 What This Example Teaches

This tutorial will teach you:

1. **What are input proofs** and why they're required
2. **How input proofs validate** encrypted inputs
3. **How to generate input proofs** using FHEVM SDK
4. **Security implications** of input proofs
5. **How to enhance input validation** with entropy
6. **The importance of input proofs** for security

## 💡 Why This Matters

Input proofs are a critical security mechanism in FHEVM:
- **Prevents invalid encrypted values** from being used
- **Validates encryption correctness** before processing
- **Ensures values come from FHEVM SDK** (trusted source)
- **Protects against malicious inputs** and tampering
- **Entropy adds randomness** to validated inputs

## 🔍 How It Works

### Contract Structure

The contract has three main components:

1. **Store with Proof**: Validates and stores encrypted value using input proof
2. **Entropy Request**: Requests randomness from EntropyOracle
3. **Store with Proof and Entropy**: Validates input, then combines with entropy

### Step-by-Step Code Explanation

#### 1. Constructor

```solidity
constructor(address _entropyOracle) {
    require(_entropyOracle != address(0), "Invalid oracle address");
    entropyOracle = IEntropyOracle(_entropyOracle);
}
```

**What it does:**
- Takes EntropyOracle address as parameter
- Validates the address is not zero
- Stores the oracle interface

**Why it matters:**
- Must use the correct oracle address: `0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361`

#### 2. Store with Input Proof

```solidity
function storeWithProof(
    externalEuint64 encryptedInput,
    bytes calldata inputProof
) external {
    require(!initialized, "Already initialized");
    
    // FHE.fromExternal requires inputProof to validate the encrypted input
    // This is a security measure to ensure the encrypted value is valid
    euint64 internalValue = FHE.fromExternal(encryptedInput, inputProof);
    
    // After validation, allow contract to use
    FHE.allowThis(internalValue);
    
    storedValue = internalValue;
    initialized = true;
}
```

**What it does:**
- Accepts encrypted value from external source (frontend)
- **Validates encrypted value using input proof**
- Converts external to internal format (only if proof is valid)
- Grants permission to use value
- Stores validated encrypted value

**Key concepts:**
- **Input proof**: Cryptographic proof validating the encrypted value
- **`FHE.fromExternal()`**: Requires input proof to validate
- **Validation**: Ensures encryption is correct and from FHEVM SDK
- **Security**: Prevents invalid or malicious encrypted inputs

**Why input proof is required:**
- Validates that encrypted value is properly encrypted
- Ensures value came from FHEVM SDK (trusted source)
- Prevents invalid or malicious encrypted values
- Security mechanism against tampering

**What happens without proof:**
- `FHE.fromExternal()` will fail
- Invalid encrypted values cannot be used
- Contract remains secure

#### 3. Request Entropy

```solidity
function requestEntropy(bytes32 tag) external payable returns (uint256 requestId) {
    require(msg.value >= entropyOracle.getFee(), "Insufficient fee");
    
    requestId = entropyOracle.requestEntropy{value: msg.value}(tag);
    entropyRequests[requestId] = true;
    
    return requestId;
}
```

**What it does:**
- Validates fee payment
- Requests entropy from EntropyOracle
- Stores request ID
- Returns request ID

#### 4. Store with Proof and Entropy

```solidity
function storeWithProofAndEntropy(
    externalEuint64 encryptedInput,
    bytes calldata inputProof,
    uint256 requestId
) external {
    require(entropyOracle.isRequestFulfilled(requestId), "Entropy not ready");
    
    // Validate input with proof
    euint64 internalValue = FHE.fromExternal(encryptedInput, inputProof);
    FHE.allowThis(internalValue);
    
    // Get entropy
    euint64 entropy = entropyOracle.getEncryptedEntropy(requestId);
    FHE.allowThis(entropy);
    
    // Combine validated value with entropy
    euint64 enhancedValue = FHE.xor(internalValue, entropy);
    FHE.allowThis(enhancedValue);
    
    storedValue = enhancedValue;
    initialized = true;
}
```

**What it does:**
- Validates encrypted input using input proof
- Gets encrypted entropy from oracle
- **Grants permission** to use entropy (CRITICAL!)
- Combines validated value with entropy using XOR
- Stores entropy-enhanced value

**Key concepts:**
- **Input validation first**: Proof validates input before entropy enhancement
- **Entropy enhancement**: Adds randomness to validated input
- **Security layers**: Input proof + entropy = double security

**Why validate then enhance:**
- Input proof ensures value is valid
- Entropy adds randomness to valid value
- Result: Validated and entropy-enhanced value

## 🧪 Step-by-Step Testing

### Prerequisites

1. **Install dependencies:**
   ```bash
   npm install --legacy-peer-deps
   ```

2. **Compile contracts:**
   ```bash
   npx hardhat compile
   ```

### Running Tests

```bash
npx hardhat test
```

### What Happens in Tests

1. **Fixture Setup** (`deployContractFixture`):
   - Deploys FHEChaosEngine, EntropyOracle, and EntropyInputProof
   - Returns all contract instances

2. **Test: Store with Proof**
   ```typescript
   it("Should store value with input proof", async function () {
     const input = hre.fhevm.createEncryptedInput(contractAddress, owner.address);
     input.add64(42);
     const encryptedInput = await input.encrypt();
     
     // inputProof is automatically generated by FHEVM SDK
     await contract.storeWithProof(
       encryptedInput.handles[0],
       encryptedInput.inputProof
     );
     
     expect(await contract.isInitialized()).to.be.true;
   });
   ```
   - Creates encrypted input using FHEVM SDK
   - SDK automatically generates input proof
   - Calls `storeWithProof()` with handle and proof
   - Verifies storage succeeded

3. **Test: Entropy Request**
   ```typescript
   it("Should request entropy", async function () {
     const tag = hre.ethers.id("test-input-proof");
     const fee = await oracle.getFee();
     await expect(
       contract.requestEntropy(tag, { value: fee })
     ).to.emit(contract, "EntropyRequested");
   });
   ```
   - Requests entropy with unique tag
   - Pays required fee
   - Verifies request event is emitted

### Expected Test Output

```
  EntropyInputProof
    Deployment
      ✓ Should deploy successfully
      ✓ Should have EntropyOracle address set
    Input Proof Validation
      ✓ Should store value with input proof
      ✓ Should fail with invalid proof
    Entropy-Enhanced Validation
      ✓ Should request entropy
      ✓ Should store with proof and entropy

  6 passing
```

**Note:** Input proofs are automatically generated by FHEVM SDK when encrypting values. Never create proofs manually.

## 🚀 Step-by-Step Deployment

### Option 1: Frontend (Recommended)

1. Navigate to [Examples page](/examples)
2. Find "EntropyInputProof" in Tutorial Examples
3. Click **"Deploy"** button
4. Approve transaction in wallet
5. Wait for deployment confirmation
6. Copy deployed contract address

### Option 2: CLI

1. **Create deploy script** (`scripts/deploy.ts`):
   ```typescript
   import hre from "hardhat";

   async function main() {
     const ENTROPY_ORACLE_ADDRESS = "0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361";
     
     const ContractFactory = await hre.ethers.getContractFactory("EntropyInputProof");
     const contract = await ContractFactory.deploy(ENTROPY_ORACLE_ADDRESS);
     await contract.waitForDeployment();
     
     const address = await contract.getAddress();
     console.log("EntropyInputProof deployed to:", address);
   }

   main().catch((error) => {
     console.error(error);
     process.exitCode = 1;
   });
   ```

2. **Deploy:**
   ```bash
   npx hardhat run scripts/deploy.ts --network sepolia
   ```

## ✅ Step-by-Step Verification

### Option 1: Frontend

1. After deployment, click **"Verify"** button on Examples page
2. Wait for verification confirmation
3. View verified contract on Etherscan

### Option 2: CLI

```bash
npx hardhat verify --network sepolia <CONTRACT_ADDRESS> 0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361
```

**Important:** Constructor argument must be the EntropyOracle address: `0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361`

## 📊 Expected Outputs

### After Store with Proof

- `isInitialized()` returns `true`
- `getStoredValue()` returns validated encrypted value
- Value is validated and secure
- `ValueStored` event emitted

### After Store with Proof and Entropy

- `isInitialized()` returns `true`
- `getStoredValue()` returns entropy-enhanced validated value
- Value is validated, entropy-enhanced, and secure
- `ValueStoredWithEntropy` event emitted

## ⚠️ Common Errors & Solutions

### Error: `Invalid input proof`

**Cause:** Wrong or missing input proof passed to `FHE.fromExternal()`.

**Example:**
```typescript
// ❌ Wrong: Creating proof manually
const fakeProof = "0x...";
await contract.storeWithProof(handle, fakeProof); // ❌ FAILS!
```

**Solution:**
```typescript
// ✅ Correct: Use proof from FHEVM SDK
const input = hre.fhevm.createEncryptedInput(contractAddress, userAddress);
input.add64(42);
const encryptedInput = await input.encrypt();
// encryptedInput.inputProof is automatically generated
await contract.storeWithProof(
  encryptedInput.handles[0],
  encryptedInput.inputProof // ✅ Use SDK-generated proof
);
```

**Prevention:** Always use the input proof generated by FHEVM SDK. Never create proofs manually.

---

### Error: `SenderNotAllowed()`

**Cause:** Missing `FHE.allowThis()` call after `FHE.fromExternal()`.

**Solution:**
```solidity
euint64 value = FHE.fromExternal(encryptedInput, inputProof);
FHE.allowThis(value); // ✅ Required!
```

**Prevention:** Always call `FHE.allowThis()` after `FHE.fromExternal()`.

---

### Error: `Entropy not ready`

**Cause:** Calling `storeWithProofAndEntropy()` before entropy is fulfilled.

**Solution:** Always check `isRequestFulfilled()` before using entropy.

---

### Error: `Insufficient fee`

**Cause:** Not sending enough ETH when requesting entropy.

**Solution:** Always send exactly 0.00001 ETH:
```typescript
const fee = await contract.entropyOracle.getFee();
await contract.requestEntropy(tag, { value: fee });
```

---

### Error: Verification failed - Constructor arguments mismatch

**Cause:** Wrong constructor argument used during verification.

**Solution:** Always use the EntropyOracle address:
```bash
npx hardhat verify --network sepolia <CONTRACT_ADDRESS> 0x75b923d7940E1BD6689EbFdbBDCD74C1f6695361
```

## 🔗 Related Examples

- [EntropyEncryption](../encryption-encryptsingle/) - Entropy-based encryption
- [EntropyCounter](../basic-simplecounter/) - Using input proofs
- [Category: input-proof](../)

## 📚 Additional Resources

- [Full Tutorial Track Documentation](../../../frontend/src/pages/Docs.tsx) - Complete educational guide
- [Zama FHEVM Documentation](https://docs.zama.org/) - Official FHEVM docs
- [GitHub Repository](https://github.com/zacnider/entrofhe/tree/main/examples/input-proof-inputproofexplanation) - Source code

## 📝 License

BSD-3-Clause-Clear
