# Appendix: CoBuild-Only or CoBuild+Legacy?

**Author:** janx  
**Date:** Feb 2024

There are currently two input data (witness) encoding modes for CKB scripts:

1. **CoBuild mode:** Encode input data in the WitnessLayout format defined in this document;
2. **Legacy mode:** Encode input data in the WitnessArgs format.

CKB transactions only care whether the included scripts and witnesses can be verified, not how these scripts parse witnesses. This means that scripts using different witness encoding mode can coexist within a transaction.

A CKB script can also support both modes: for example, such a script can first look for witness in the WitnessLayout format, and if it's missing, it tries to find a WitnessArgs witness.

Usually when 'CoBuild mode' or 'Legacy mode' is mentioned, it refers to a certain script being in one of these modes, after deciding which input data encoding to use. At this point, this script has determined either to only read from WitnessLayout witnesses (CoBuild Mode), or only from WitnessArgs witnesses (Legacy Mode).

However when discussing "a script supports CoBuild mode" below , it means that a script is able to operate under CoBuild mode (not "only" or "already determined") and read from WitnessLayout witnesses. Correspondingly, "a script supports Legacy mode", means that this script is able to operate under Legacy Mode (not "only" or "already determined") and reads from WitnessArgs witnesses.

## Lock Script

In the transition period (from now until CoBuild becomes widely used as default), we do not recommend lock script supporting CoBuild mode only without supporting Legacy mode.

It's not difficult for a lock script to support both CoBuild mode and Legacy mode. The two modes' difference lie only on the calculation of signing message hash. A well structured lock script could reuse almost all verification codes in the two modes.

If a lock script wants to support as many CKB apps as possible, the optimal solution is to support both CoBuild and Legacy modes.

## Type Script

The considerations for type script are somewhat different from those of lock script.

Without considering some extreme cases in OTX, a lock script that only supports CoBuild mode cannot be used with a type script that can only process WitnessArgs typed witnesses in the same cell.

What about the other way around? Can a type script that only supports CoBuild mode be used with a lock script (such as the default secp256k1-blake160 lock) that can only process WitnessArgs typed witnesses? The answer is yes.

### Compatibility Example

Let's look at this example:

![CoBuild-Only or CoBuild+Legacy](CoBuild-Only%20or%20CoBuild+Legacy.png)

Assume here I1 and I2 both use the secp256k1-blake160 lock, while I1 uses a type script that only supports CoBuild mode. A valid transaction can use the above layout: witness #0 and witness #1 are of the WitnessArgs type, witness #2 is of CoBuild mode's WitnessLayout::SighashAll type which contains an Action required by the type script in I1.

According to the conventions used in legacy lock scripts, although the lock scripts used by I1 and I2 cannot parse the WitnessLayout typed witness#2, these lock scripts will still sign all data of witness#2. Meanwhile, the type script used by I1 could parse Message from witness#2, and find its corresponding Action to complete the verification.

We can see that there won't be compatibility issues for type scripts supporting only CoBuild mode without supporting Legacy mode. For type script, you could choose either to support CoBuild mode or be compatible with both modes. Our suggestion new type scripts to consider supporting CoBuild mode first, and being compatible with Legacy mode if necessary.

## Action May Not Exist

In most cases, there will be a WitnessLayout::SighashAll witness in CoBuild transactions, which contains the Message, as well as the Actions corresponding to the type scripts used. However, it should be noted that the CoBuild protocol does not require that a WitnessLayout::SighashAll must exist.

Suppose there is a type script that only supports CoBuild mode. One problem it needs to face is: Should it be able to handle situations where its own Action is missing when a WitnessLayout::SighashAll doesn't exist?

Different type scripts can take different actions here based on their own requirements. A type script can choose whether to exit with an error directly or ignore the issue and continue verification when its required 'Action' does not exist.

It should be pointed out that for most scenarios, existence of an Action can greatly simplify on-chain verification. It's usually difficult to guess the user action by reverse analyzing transaction data.

Therefore, we suggest adding this validation rule for type scripts: it must find the required Action in transaction before it can proceed.

## NervosDAO Type Script

The NervosDAO type script is a special example for CoBuild process because its code was deployed into genesis block with unspendable lock (lock hash = 0x0000…, all zero). This means NervosDAO cannot be upgraded via methods other than hard fork.

If we still want the NervosDAO type script to be compatible with CoBuild without upgrading it through hard fork, additional special handling is needed. One approach is to hardcode specific logic related to NervosDAO within CoBuild toolchain. If a CoBuild tool detects a transaction involving NervosDAO, it should perform some preprocessing:

1. If the transaction does not contain a NervosDAO Action, CoBuild tools should construct one using raw transaction data for UX presentation.
2. If it contains a NervosDAO Action, these tools should validate the action, take the responsibility that would normally be taken by the NervosDAO type script.

Remember, during transition period (from now until CoBuild becomes widely used), we don't recommend CKB lock scripts to only support CoBuild mode without supporting Legacy mode. However, due to the speciality of Nervos DAO type script, we suggest that for all operations involving Nervos DAO, the tools should generate transactions using WitnessArgs regardless of whether or not the lock scripts used support CoBuild. Even if for some operations the generated transaction can use CoBuild because the NervosDAO type script doesn't need to read witness in that case, the generated transaction shouldn't. This way we simplify implementation by the requirement of using WitnessArgs in all cases.

Then fully support for CoBuild can be added to NervosDAO type script when it is going to be upgraded for some reason in future (no plan for now).