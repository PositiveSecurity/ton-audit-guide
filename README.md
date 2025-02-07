# Checklist for Auditing TON Smart Contracts

## General

- **Mapping Message Flows:**
  - Map all message flows of the contract to understand how messages are processed and routed.
- **Key Questions:**
  - **Partial Execution of Transactions:**
    - What happens if a transaction is **partially executed** due to gas exhaustion?
    - How does the contract handle partial execution?
  - **Entry Points:**
    - Identify all **entry points** of the contract.
  - **Input Data Processing:**
    - How are **input data** and incoming messages processed?
    - Are incoming errors appropriately handled?
  - **Authorization Checks:**
    - Are there **authorization checks** for all functions and message handlers?
  - **Contract Design and Centralization:**
    - Examine the contract structure for any unnecessary **centralization**.
  - **External Message Handling:**
    - How does the contract handle **external messages**?
    - Ensure that `accept_message()` is used **only after** proper validations to prevent gas draining attacks.
    - Verify the implementation of `recv_external`.
  - **Preventing Freezing and Deletion:**
    - What mechanisms are in place to prevent the contract from being **frozen** or **deleted**?

## Asynchronous Execution

- **Understanding Asynchrony:**
  - **Message Delivery Guarantees:**
    - Messages are guaranteed to be delivered but not in a predictable timeframe.
  - **Message Order Uncertainty:**
    - The order of messages can only be predicted if messages are sent to the same contract, in which case [logical time](https://docs.ton.org/develop/smart-contracts/guidelines/message-delivery-guarantees#what-is-a-logical-time) is used.
      - If multiple messages are sent to different contracts, the order in which they are received is not guaranteed.
- **Key Considerations:**
  - **Concurrent Processes:**
    - What happens if another process runs in parallel?
    - How can this affect the contract, and how can it be mitigated?
  - **State Changes Between Messages:**
    - Can any required values change between message flows?
  - **External Dependencies:**
    - On which parameters or states of other contracts does this contract rely?
  - **Operation Independence:**
    - Ensure that operations are independent of the sequence of message arrivals.

## Common errors in TON

### What to look out for

- **The public nature of blockchain:**
  - All data that is transmitted in a message is public and available to everyone
  - You can't send passwords, keys, and so on to the blockchain
- **Bounced message handlers:**
  - If there was no such processing in Jetton, you could send tokens to the void. The [TON Stablecoin](https://github.com/ton-blockchain/stablecoin-contract) project has a [processing](https://github.com/ton-blockchain/stablecoin-contract/blob/main/contracts/jetton-minter.fc#L62-L73) bounced `op::internal_transfer` message that is sent to the Jetton wallet when minting tokens. If there is no processing, then `total_supply` will increase at minting and will be irrelevant since the tokens have not arrived on the wallet and they cannot be in circulation.
- **Restrictions on data recording:**
  - **Unrestricted data recording:**
    - Is there a risk of **unlimited data writing** to the contract?
    - Avoid patterns that result in infinite data growth, such as infinite sharding.
  - **Tokenization:**
    - Tokenize assets or data into separate contracts if necessary.
- **Logical errors:**
  - Errors in formulas, algorithms, and data structures are common. For example, if you perform division first and then multiplication, you may lose calculation accuracy and get a rounding error
    - `let result: Int = x / z * y;`
  - Standard errors
    - Code duplication
    - Unreachable code
    - Redundant code
    - Inefficient algorithms (on gas)
    - Inappropriate order of expressions in conditional statements
    - Errors in data parsing
    - Constant conditions in `if`
- **Replay Attack:**
  - You can protect yourself with `seqno` (one-time numbers)
    - [Checks `seqno`](https://github.com/ton-blockchain/ton/blob/master/crypto/smartcont/wallet3-code.fc#L15)
    - [Signature Checked](https://github.com/ton-blockchain/ton/blob/master/crypto/smartcont/wallet3-code.fc#L17)
- [**Carry-Value Pattern**](https://docs.ton.org/develop/smart-contracts/security/secure-programming#4-use-a-carry-value-pattern)
  - Transferring a value between contracts
    - The sender subtracts `amount` from its balance and sends it with `op::internal_transfer`.
    - The receiver takes the `amount` via message and adds it to its own balance (or rejects it).
  - In TON, it is not possible to get actual data via a request, because by the time the response reaches the requestor, the data may no longer be up to date.
- **Smart Contract Update:**
  - **Storage compatibility:**
    - Ensure that updates do not break existing storage logic (watch out for `storage` collisions).
    - The `set_code` and `set_data` functions are used to update the `c3` and `c4` registers, and can completely overwrite the registers
  - **Delayed code updates:**
    - Realize that code updates do not affect the current transaction:
      - Changes take effect **only after** the current execution has successfully completed.
  - **Decentralization Issues:**
    - Evaluate how updates may affect the decentralization of the contract.
- **Use of signed and unsigned numbers:**
  - If `int x` is sent in a message and `x` is added to `uint y` in the code, it is possible for the expression `x + y` to become `x - y`
- **Exit codes:**
  - The use of reserved exit codes can cause confusion when handling errors
  - TON Blockchain reserved exit code values from 0 to 127
    - [TVM exit codes](https://docs.ton.org/v3/documentation/tvm/tvm-exit-codes)
  - Tact [utilizes](https://docs.tact-lang.org/book/exit-codes/) exit codes from 128 to 255
- **Sending messages from a loop:**
  - A loop can be infinite or have too many iterations, which can lead to unpredictable contract behavior.
  - Continuous looping without completion can lead to an out-of-gas attack.
  - Attackers can use unlimited loops to create denial-of-service attacks.
- **Parsing and serialization:**
  - Use `end_parse()` whenever possible when reading data from storage or message payloads to ensure that parsing completes correctly.

### Sending messages

- **Check Flags:**
  - Verify that all keys and magic numbers match the contract logic when generating the message.
  - Make sure flags are not duplicated.
- **Resource Management:**
  - Check that the message parameters do not excessively deplete the contract balance, especially for storage payments.
  - Sending a message with `mode=SendRemainingValue`/`mode=64` or `mode=128` will cause the operations following that message to fail.
- The [documentation](https://docs.ton.org/v3/documentation/smart-contracts/message-management/message-modes-cookbook) is the best way to understand how messages are generated.

### Gas Control

- **Moderate handling of gas:**
  - **Calculation of costs:**
    - Carefully calculate gas costs and verify that there is enough gas to operate.
  - **Data Structures:**
    - Be careful of data structures that can grow **infinitely** as they increase gas costs over time.
  - **Return excess gas:**
    - Implement logic to return **excess gas** to the sender when appropriate.
    - Failure to refund excess gas amounts paid can result in a buildup in the contract.
  - **Partial fulfillment risks:**
    - Recognize that running out of gas results in **partial performance**, which can lead to critical issues.
  - **Storage Parsing Efficiency:**
    - Avoid parsing and repacking the entire storage in each function call to reduce gas consumption.
    - Use nested storage.

### Random number generation in TON

- **Safe randomness generation**
  - Validators can affect randomness
  - Hackers can figure out the formula for generating randomness.
  - [Be smart about randomness generation](https://docs.ton.org/v3/guidelines/smart-contracts/security/random-number-generation)
  - [Randomness in Tact](https://docs.tact-lang.org/cookbook/random/)
  - [Lessons on developing smart contracts on FunC “Random in TON”](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/lessons/bonus/random/random.md)
- **Pay attention to the use of:**
  - `random()` in FunC requires `randomize_lt()` or `randomize(x)`
  - `nativeRandom` or `nativeRandomInterval` in Tact instead of `randomInt` and `random` respectively

## Possible errors in FunC

- **Function modifiers:**
  - Are all important functions correctly labeled with the `impure` modifier if they change state?
- **Modifying variables:**
  - Review [modifying and non-modifying methods](https://docs.ton.org/develop/func/statements#methods-calls).
- **Storage management:**
  - **Variable ordering:**
    - Verify the correct ordering of variables when packing into the repository.
  - **Manual storage management:**
    - Remember that the developer manually manages the storage and the data in the `c4` register is completely overwritten.
  - **Variable overrides:**
    - Check if state variables can be overwritten due to variable name collisions or namespace contamination.
    - This is possible [due to redeclaration](https://docs.ton.org/v3/documentation/smart-contracts/func/docs/statements#variable-declaration) of variables in FunC
  - **Nested Storage:**
    - Consider using nested storage (variables within variables) for better organization.

### Third-Party Code Execution

- **Avoid External Code Execution:**
  - **Safety Risks:**
    - Executing third-party code is unsafe because `out of gas` exceptions cannot be caught with `CATCH`.
  - **Potential Exploits:**
    - An attacker could use `COMMIT` to alter the contract state before raising an `out of gas` exception.
  - **Best Practices:**
    - Keep contract logic self-contained and avoid incorporating untrusted external code.

## Possible errors in Tact

- **Modifying the parameters of an incoming message or function:**
  - In Tact, function arguments are passed by value, which means that any mutations applied to these arguments will only affect the local copy of the variable within the function.
- **Use of assembler:**
  - The use of TVM Assembly is a potentially dangerous operation that requires additional attention from the auditor.
- **Use of native functions:**
  - Potentially dangerous operation requiring additional attention of the auditor.
- **Double initialization of variables:**
  - The language allows initializing state variables at declaration time, but it is better to initialize variables in `init()` to avoid such errors
  - If this happens, the value from `init()` will be stored in the contract
- **Direct modification of inherited variables:**
  - Variables from `trait` may have their own update rules, so direct modification may violate the contract invariant.
- **Variables with optional value:**
  - If a developer creates an optional variable or field, they should use its functionality by referring to a null value somewhere in the code.
  - Otherwise, the optional type should be removed to simplify and optimize the code.

## Additional Recommendations

- **Magic numbers, flags, and constants:**
  - Replace magic numbers with named constants for clarity and ease of maintenance.
- **Documentation:**
  - Maintain clear and thorough documentation of the contract's functionality and design decisions.
- **Code Review:**
  - Have the contract code reviewed by independent auditors to catch potential issues.
- **Compliance with Standards:**
  - Ensure the contract adheres to relevant TON standards and best practices.

By following this checklist, you can systematically assess the security and robustness of TON smart contracts, identifying potential vulnerabilities and ensuring reliable operation within the TON ecosystem.
