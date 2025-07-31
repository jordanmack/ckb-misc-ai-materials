# Appendix: Lock/Type Script Validation Rules

**Author:** janx  
**Date:** Feb 2024

This section describes the validation rules that a lock/type script needs to include in order to meet the requirements of the CoBuild protocol.

From a convenience and practicality standpoint, we recommend all lock/type scripts support legacy WitnessArgs, then add support for SighashAll Variant, and support for Otx Variant.

## Legacy Flow Handling

Considering forward compatibility, lock / type scripts that support the CoBuild protocol may also need to support the legacy flow, which is the existing process based on WitnessArgs structure.

A lock/type script can simultaneously support both CoBuild flow and legacy flow. However, for a specific transaction, a lock/type script must choose one of the two for verification: either use CoBuild flow or use legacy flow.

Usually, all scripts in a transaction should use the same flow for verification. But there's an exception: if a tranasction uses script A that supports both flows and script B that only supports legacy flow, it's possible that script A verifies according to CoBuild flow while script B verifies according to legacy flow.

Lock/type scripts should follow these rules to determine the flow to use:

- If there is a witness of the type WitnessLayout in current transaction, use the CoBuild Flow.
  - Note: The condition here is its existence, not uniqueness, not if different WitnessLayout witnesses coexist, nor its compliance with other CoBuild rules.
- Otherwise, use the Legacy Flow.

## Lock Script

The main responsibility of a CKB lock script is to perform ownership (owner signature) verification - checking that the input cell is unlocked under the correct conditions, while also ensuring that data in transactions are not tampered with. A lock script that supports both SighashAll and Otx modes should verify transactions according to the following rules:

1. Maintain a set of variables: is, ie, os, oe, cs, ce, hs, he and set them all to 0.
   - is: input start
   - ie: input end
   - os: output start
   - oe: output end
   - cs: cell dep start
   - ce: cell dep end
   - hs: header start
   - he: header end

2. Loop through all witnesses of current tx (loop variable i), check if there's a witness in current tx using WitnessLayout::OtxStart type.

3. If no witness in current tx uses WitnessLayout::OtxStart, skip to step 8 for further validation.

4. Ensure only one witness is of type WitnessLayout::OtxStart in current tx or else exit with an error message. Set loop variable i as this witness's index value.

5. Set values of is and ie as start_input_cell in OtxStart, os and oe as start_output_cell in OtxStart, cs and ce as start_cell_deps in OtxStart, hs and he as start_header_deps in OtxStart.

6. Starting from witness at index i+1 , loop through each WitnessLayout::Otx typed witness (exit loop directly if encounter any witness not in WitnessLayout::Otx type), validate each WitnessLayout::Otx witness (i.e. each Otx in this tx) as follows:

   a. Let variable 'otx' store the WitnessLayout::Otx type witness, then the following data constitutes current Otx (open transaction):
      i. Input cells with index within [ie, ie + otx.input_cells)
      ii. Output cells with index within [oe, oe + otx.output_cells)
      iii. Cell deps with index within [ce, ce + otx.cell_deps)
      iv. Header deps with index within [he, he + otx.header_deps)
      v. The current WitnessLayout::Otx typed witness

   b. Check otx, if all values for 'otx.input_cells', 'otx.output_cells', 'otx.cell_deps', and 'otx.header_deps' are 0, the script should exit with an error.

   c. Parse otx.message, check each Action in the message, the script_hash in Action must either be the same as the type script hash of one input/output cells in transaction, or be the same as the lock script hash of one input cells in transaction. Otherwise, the script shall terminate with an error.
      
      Note that this check applies to the entire transaction not just current otx.

   d. Check input cells in the range [ie, ie + ox.input_cells), skip to next iteration if none use current lock script.

   e. Calculate corresponding signing message hash based on Otx mode rules defined in Appendix: CoBuild Hashes.

   f. Fetch SealPair from seals, using the current lock script's lock script hash as key . Exit with error if not found.

   g. Extract signature from SealPair according to the lock's own logic, verify it against previously generated signing message hash. Exit with error if verification fails.

   h. ie += otx.input_cells; oe += otx.output_cells; ce += otx.cell_deps; he += otx.header_deps

7. Assume the first witness not of type WitnessLayout::Otx starting from index i+1 is at index j , check that all witnesses in the the range [0, i) and [j, +infinity) are not of type WitnessLayout::Otx or else exit with an error.

8. Check the input cells within the range of [0, is) and [ie, +infinity), if any of them uses current lock script, perform a SighashAll signature verification:

   a. Check if there's a WitnessLayout::SighashAll variant among all witnesses in the current tx.

   b. If there exists a witness of type WitnessLayout::SighashAll, ensure that only one such witness exists in the current tx and store its witness.message in variable message. If no such witness exists in the current tx, set variable message as empty.

   c. Assuming that message is not empty, check that for each Action contained in message, the script_hash in Action must either be the same as the type script hash of one input/output cells in transaction, or be the same as the lock script hash of one input cells in transaction. Otherwise, the script shall terminate with an error.

   d. Read the first witness from the script group corresponding to current lock script. If this witness isn't of type WitnessLayout::SighashAll or WitnessLayout::SighashAllOnly, exit with error.

   e. Ensure that apart from the first witness position inside current lock script group, there're either no other witnesses or they're empty.

   f. Extract seal from the first witness within current lock script group and retrieve signature according to the lock script's own custom logic.

   g. Calculate signing message hash by following the SighashAll/SighashAllOnly rules outlined under Appendix: CoBuild Hashes.

   h. Verify the extracted signature along with the signing message hash; if verification fails then exit with error.

## Type Script

CKB type scripts mainly verify application-specific on-chain state transitions. For instance, in UDT type scripts validate token issuance and transfer. In CoBuild process, apart from validating those state transitions, type scripts should also verify consistency between message and full/open transaction behaviors. During message consistency verification, a type script needs to validate each otx individually as well as a possible SighashAll witness containing message.

A type script supporting both SighashAll and Otx variant should follow these validation rules:

1. Execute the app-specific state transition logic defined by the type script.

2. Maintain a set of variables is, ie, os, oe, cs, ce, hs, he all initialized to 0.

3. Loop through all witnesses in current tx (loop variable i), checking if there's a witness of type WitnessLayout::OtxStart.

4. If no witness of type WitnessLayout::OtxStart exists in current tx, jump to step 9 for further verification.

5. Ensure only one witness is of type WitnessLayout::OtxStart in the transaction; otherwise exit with error. Set variable i as index value of this witness.

6. Set is and ie values equal to start_input_cell in OtxStart; similarly set os and oe values equal to start_output_cell in OtxStart; set cs and ce values equal to start_cell_deps in OtxStart; finally, set hs and he values equal to start_header_deps in OtxStart.

7. Starting from the witness with index of i + 1, loop through each witness of type WitnessLayout::Otx (exit the loop immediately when encountering a witness that is not of type WitnessLayout::Otx) and perform the following verification for each WitnessLayout::Otx typed witness (i.e. for each Otx in tx):

   a. Assume that the variable otx stores the witness of type WitnessLayout::Otx, then the following data constitutes the current Otx (Open Transaction):
      i. Input cells within index range [ie, ie + otx.input_cells)
      ii. Output cells within index range [oe, oe + otx.output_cells)
      iii. Cell deps within index range [ce, ce + otx.cell_deps)
      iv. Header deps within index range [he, he + otx.header_deps)
      v. The currently processed witness of type WitnessLayout::Otx

   b. Check input cells within the range of [ie, ie+otx.input_cells) and output cells within the range of [oe, oe+otx.output_cells). If all these input/output cells do not use current type script then skip to next iteration.

   c. Verify message in WitnessLayout::OTX based on current OTX scope. (Refer to the next section for specific verification process).

   d. ie += otx.input_cells; oe += otx.output_cells; ce += otx.cell_deps; he += otx.header_deps

8. Assume starting from the witness at index i+1, the first witness that isn't of type WitnessLayout::OTX has an index j. Make sure witnesses within [0,i) and [j,+infinity) ranges are not of Witnesslayout::OTX type, otherwise exit with error.

9. Check input and output cells within [0, is) and [ie, +infinity), if any of these cells use the current type script, perform the following verification:

   a. Scan all witnesses of the current transaction to determine whether there is a witness of WitnessLayout::SighashAll type.

   b. If no witness of WitnessLayout::SighashAll type exists in the current transaction, consider the script validation successful and exit execution with return code 0.

   c. Check if there's exactly one Witnesslayout::SighashAll typed witness in current tx, else exit with an error.

   d. Verify message in WitnessLayout::SighashAll, based on the scope including input cells within [0, is) & [ie, +infinity) and output cells within [0, os) & [oe, +infinity). (Refer to the next section for specific verification rules.)

## Message Validation

This section describes how the type script validates messages.

Whether it's an Otx or SighashAll mode transaction, we need to validate the Action in message corresponding with this type script against a range of related input/output cells.

Note that the validation is against "a range of cells": due to the possible existence of otx , this range may not necessarily be the entire tx but could be intervals like [3,6), or the union of two intervals such as [0,4) || [8,11). However there will only ever be union of two ranges maximum; never more than that.

The type script needs to traverse the actions in message, find an Action where Action.script_hash matches its own script hash. This Action is the one corresponding to the current type script.

If multiple Actions are found that correspond to the type script hash, it is recommended to throw an error. The purpose is to prevent third parties from inserting Actions calling the same type script into the transaction later to override user's Action.

Third-party Actions may appear after user's Action on wallet UX or even be hide due to insufficient display space, leading users to believe that only their own actions are being signed.

### Example Code

**Note:** The on-chain type script cannot fetch the complete ScriptInfo of the current Action, nor can it verify the correctness of Action.script_info_hash.

Therefore, type scripts can make the following two assumptions:

1. The content of Action, including Action.script_info_hash, has been covered by lock scripts during verification process, ensuring no tampering.

2. In the signing process, wallets will get the correct ScriptInfo through some offchain channel, verify the consistency between the ScriptInfo and Action.script_info_hash, parse 'Action.data' with the correct 'ScriptInfo.schema', and present the correctly parsed data to the cell holder for signing.

Although a type script does not have the ScriptInfo, it should have information like ScriptInfo.schema and ScriptInfo.message_type in its source code. Type scripts can use these information to parse Action.data, get the structured action data (as input) and validating cells. For instance, if Action.data presents a Spore NFT Mint operation, the type script needs to verify that cells within certain scope indeed expressed a mint operation; if Action.data presents a UDT transfer, similar verification is required by type script to ensure correct amount of UDT has been transferred to the right recipient and change UDT returned to payer.