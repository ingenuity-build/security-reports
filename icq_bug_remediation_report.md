### ICQ Bug Remediation Report
This report describes a vulnerability discovered in the ICQ implementation on Quicksilver. The issue was responsibly disclosed to Quicksilver by [Puneet Mahajan](https://github.com/puneet2019) from pSTAKE Finance on May 29, 2023 at 1100 UTC, and was fixed within 25.5 hours via a chain halt on May 30, 2023 at 1100 UTC and subsequent upgrade on May 30, 2023 at 1215 UTC. Puneet and the pSTAKE team are applauded for their forthright attitude, collaboration, and diligence. Thanks to this effort, no malicious exploit occurred.


### Executive Summary

**Date of Report:** May 30, 2023.

**Time between Bug Report & Resolution:** 25.5 hours

**Authors:** Quicksilver Development team.

**Status:** Resolved WITHOUT outstanding action items.

**Bug Identified:** Ability to mint qAssets greater than the value of the corresponding deposit.

**Root Cause of Bug:** The use of Events to inform Quicksilver of the corresponding deposit, which are not part of consensus and thus not covered by Merkle tree proofs.

**Remediation:** Decode transaction object from the TxProof Data field, and operate only on that proven object.

**Remediation Impact on Quicksilver Chain:**

1. Chain halt for 1 hours and 15 minutes.


### **ICQ on Quicksilver**

Interchain queries (ICQ) are a mechanism by which chains can collect and consume data from external chains. Given that the querying chain is connected via IBC to the source chain, the IBC light client can be used to validate Merkle tree proofs, thus cryptographically verifying the veracity of the data consumed.

Quicksilver uses ICQ for maintaining information about validators, account balances, delegations and rewards. It is also used to witness deposit transactions into the protocol. This specific code path differs from the regular KV store logic, as transactions themselves are not provable in the same way.

In order to prove a transaction, we must request a TxProof from the Tendermint layer on the source chain. This proof contains the transaction data, the apphash (application hash) of the block in which the transaction was included and the proof of inclusion of the given Transaction in the Merkle Tree that generated the apphash.

Using these data, we can assert cryptographically that the transaction was included in the block with that apphash. Once the block is validated as genuine via the IBC consensus state, the transaction can be asserted to be valid.

Quicksilver uses this mechanism to determine when deposits have happened on chain.


### **Vulnerability Summary**

Events are emitted when a cosmos-sdk blockchain executes a task. All transactions, and numerous non-transaction-based actions emit events. Relayers (both IBC and ICQ) use Events to trigger transactions.

When ICQ queried for a transaction with a given hash, the returned TxResponse contained Events corresponding to the deposit transaction. This TxResponse object was submitted along with the TxProof to Quicksilver.

The ICQ implementation erroneously used these Events as a source of truth. However, Events do not factor into consensus state because they are not committed to the Merkle Tree and are not part of the apphash. The transaction hash submitted was itself checked for inclusion in a block, but the events could be changed by the node responding to the ICQ query. Local replications of this bug on devnet allowed the development team to mint arbitary quantities of qAssets regardless of initial deposit size.


### **Vulnerability Remediation**

The fix changed the deposit callback logic to reconstitute the transaction from the proof, and use those fields - cryptographically verified to be accurate and to have been executed on the source chain - to make decisions about what assets to mint, and where to send them. The TxResponse object is no longer used in any form by ICQ and the Interchainstaking module.


### **Timeline**

####  _Times in UTC._

**May 29, 2023:**

**1200**: Vulnerability reported to Quicksilver team by contributors.

**1500**: Vulnerability is deemed valid, and can be replicated reliably.

**1800**: First fix is produced; internal testing shows that assets cannot be arbitrarily minted via the original exploit. Fix is deployed to devnet to monitor for correctness.

**May 30, 2023:**

**0200**: Fix is shared with reporters.

**0800**: Validators contacted to request chain halt at height `2148750`.

**0800**: Reporters revert that the original exploit is fixed, but a more complex variation is still applicable.

**0900**: Final fix is identified, implemented and shared with reporters, who confirm they are no longer able to execute the exploit.

**1100**: Chain halts.

**1145**: [v1.2.13](https://github.com/ingenuity-build/quicksilver/releases/tag/v1.2.13) is released.

**1215**: Chain resumes.
