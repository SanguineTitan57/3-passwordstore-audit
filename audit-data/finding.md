### 1. [H-1] Storing the password on-chain makes it publicly visible

**Description:**  
All data stored on-chain is **public** and visible to anyone. The `PasswordStore::s_password` variable is intended to be hidden and only accessible by the owner through the `PasswordStore::getPassword` function.  

Below is a demonstration of how anyone can read such data directly from the blockchain.

**Impact:**  
Anyone can read the supposedly private password, **completely compromising** the intended functionality of the protocol.

**Proof of Concept:**  
The following steps show how any user can read the password directly from the blockchain. We use Foundry’s `cast` tool to read the contract’s storage without being the owner.

1. **Start a local chain:**
   ```bash
   make anvil
   ```

2. **Deploy the contract:**

   ```bash
   make deploy
   ```

3. **Read the storage value:**
   We use slot `1` since it holds `s_password` in the contract.

   ```bash
   cast storage <ADDRESS_HERE> 1 --rpc-url http://127.0.0.1:8545
   ```

   Example output:

   ```
   0x6d7950617373776f726400000000000000000000000000000000000000000014
   ```

4. **Parse the bytes into a string:**

   ```bash
   cast parse-bytes32-string 0x6d7950617373776f726400000000000000000000000000000000000000000014
   ```

   Output:

   ```
   myPassword
   ```

**Recommended Mitigation:**
The contract’s architecture should be reconsidered. One approach is to **encrypt the password off-chain** and then store only the encrypted value on-chain.
The user would then need to remember a separate decryption key off-chain.
Also, consider **removing the view function** to prevent users from accidentally exposing the decryption key by making an on-chain call.

Example (for reference):

```solidity
function getPassword() external returns (string memory) {
    if (msg.sender != s_owner) {
        revert PasswordStore__NotOwner();
    }
    return s_password;
}
```

---

### 2. [H-2] `PasswordStore::setPassword` lacks access control, allowing anyone to change the password

**Description:**
The `PasswordStore::setPassword` function has no access control, meaning **any address** can call it and modify the stored password.

**Impact:**
Any user can set or overwrite the password, **completely breaking** the intended functionality of the contract.

**Proof of Concept:**
The fuzz test below demonstrates that any arbitrary address can call `setPassword`:

<details>
<summary>Code</summary>

```solidity
function test_anyone_can_set_password(address randomAddress) public {
    vm.assume(randomAddress != owner);
    vm.startPrank(randomAddress);

    string memory expectedPassword = "myNewPassword";
    passwordStore.setPassword(expectedPassword);

    vm.startPrank(owner);
    string memory actualPassword = passwordStore.getPassword();
    assertEq(actualPassword, expectedPassword);
}
```

</details>

**Recommended Mitigation:**
Add **access control** to `PasswordStore::setPassword` so that only the owner can modify the password.

<details>
<summary>Code</summary>

```solidity
function setPassword(string memory newPassword) external {
    // Add access control here
    if (msg.sender != s_owner) {
        revert PasswordStore__NotOwner();
    }
    s_password = newPassword;
    emit SetNetPassword();
}
```

</details>

---

### 3. [I-1] Incorrect NatSpec tag in `PasswordStore::getPassword` documentation

**Description:**
The NatSpec for `PasswordStore::getPassword` incorrectly includes a `@param` tag for a parameter that does not exist.

```solidity
/*
 * @notice This allows only the owner to retrieve the password.
 * @param newPassword The new password to set.
 */
function getPassword() external view returns (string memory) {}
```

The function signature is `getPassword()` with **no parameters**, but the NatSpec suggests it accepts `newPassword`.

**Impact:**
The NatSpec documentation is **misleading and inaccurate**, which could confuse developers and auditors.

**Recommended Mitigation:**
Remove the incorrect `@param` line from the NatSpec block.

```diff
/*
 * @notice This allows only the owner to retrieve the password.
- * @param newPassword The new password to set.
 */
```