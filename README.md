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
- **Patterns and Practices:**
  - **Carry-Value Pattern:**
    - Is the [**carry-value pattern**](https://docs.ton.org/develop/smart-contracts/security/secure-programming#4-use-a-carry-value-pattern) used appropriately to manage tokens and state across messages?

## Security Checks

- **Function Modifiers:**
  - Are all important functions correctly marked with the `impure` modifier if they modify state?
  - Review [modifying and non-modifying methods](https://docs.ton.org/develop/func/statements#methods-calls).
- **Flag Handling:**
  - Ensure careful processing of received **flags** to avoid logic errors.
- **Replay Attacks:**
  - Assess the possibility of **replay attacks**:
    - Could data from a testnet be replayed on the mainnet?
    - Implement a `nonce` mechanism for one-time operations.
- **Randomness Implementation:**
  - Is the random number generation sufficiently secure?
    - Ensure that the **seed** cannot be manipulated.
    - [TON Documentation "Random number generation"](https://docs.ton.org/develop/smart-contracts/guidelines/random-number-generation)
    - [Lessons on developing smart contracts on FunC "Random in TON"](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/lessons/bonus/random/random.md)
- **Storage Management:**
  - **Variable Ordering:**
    - Verify the correct order of variables when packing into storage.
  - **Manual Storage Handling:**
    - Be aware that the developer manually manages storage, and data in register `c4` is completely overwritten.
  - **Variable Overriding:**
    - Check for potential overwriting of state variables due to variable name collisions or namespace pollution.
  - **Nested Storage:**
    - Consider using nested storage (variables within variables) for better organization.
- **Data Writing Limits:**
  - **Unlimited Data Writing:**
    - Is there a risk of **unlimited data writing** to the contract?
    - Avoid patterns that lead to infinite data growth, such as infinite sharding.
  - **Tokenization:**
    - Tokenize assets or data into separate contracts when necessary.
- **Parsing and Serialization:**
  - Use `end_parse()` wherever possible when reading data from storage or message payloads to ensure proper parsing termination.

## Message Formation

- **Key and Magic Number Verification:**
  - **Consistency:**
    - Ensure all keys and magic numbers in message formation align with the contract logic.
    - [TON Documentation "Messeges"](https://docs.ton.org/develop/smart-contracts/messages)
  - **Resource Management:**
    - Check that message parameters do not deplete the contract's balance unduly, especially regarding storage payments.

## Gas Management

- **Deliberate Gas Handling:**
  - **Expense Calculation:**
    - Calculate gas expenses meticulously and verify that sufficient gas is provided for operations.
  - **Data Structures:**
    - Be cautious with data structures that can grow indefinitely, as they increase gas costs over time.
  - **Returning Excess Gas:**
    - Implement logic to return **excess gas** to the sender when appropriate.
  - **Partial Execution Risks:**
    - Recognize that gas exhaustion leads to **partial execution**, which can introduce critical issues.
  - **Efficiency in Storage Parsing:**
    - Avoid parsing and repacking the entire storage on every function call to reduce gas consumption.

## Contract Updates

- **Data Consistency:**
  - **Storage Compatibility:**
    - Ensure that updates do not violate the existing data storage logic (watch for `storage` collisions).
  - **Deferred Code Updates:**
    - Understand that code updates do not affect the current transaction:
      - Changes take effect only after the successful completion of the current execution.
  - **Decentralization Concerns:**
    - Evaluate how updates might impact the decentralization of the contract.

## Third-Party Code Execution

- **Avoid External Code Execution:**
  - **Safety Risks:**
    - Executing third-party code is unsafe because `out of gas` exceptions cannot be caught with `CATCH`.
  - **Potential Exploits:**
    - An attacker could use `COMMIT` to alter the contract state before raising an `out of gas` exception.
  - **Best Practices:**
    - Keep contract logic self-contained and avoid incorporating untrusted external code.

## Additional Recommendations

- **Magic Numbers and Constants:**
  - Replace magic numbers with named constants for clarity and maintainability.
- **Error Handling:**
  - **Arithmetic Overflows and Underflows:**
    - These throw errors in TON; ensure they are properly handled.
  - **Formulas and Algorithms:**
    - Verify the correctness of all formulas and algorithms used.
  - **Logical Vulnerabilities:**
    - Look for logical loopholes that could be exploited.
- **Namespace Management:**
  - Use unique prefixes or modules to prevent variable name collisions.
- **Documentation:**
  - Maintain clear and thorough documentation of the contract's functionality and design decisions.
- **Code Review:**
  - Have the contract code reviewed by independent auditors to catch potential issues.
- **Compliance with Standards:**
  - Ensure the contract adheres to relevant TON standards and best practices.

By following this checklist, you can systematically assess the security and robustness of TON smart contracts, identifying potential vulnerabilities and ensuring reliable operation within the TON ecosystem.
