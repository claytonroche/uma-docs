# ⚡ Snapshot Tutorial

**Adding Transaction Data to Proposal**

Now that the oSnap module has been configured with Snapshot, you can use the transaction builder when creating a Snapshot proposal. The example proposal below includes an ETH transfer of 0.000005.

<figure><img src="../../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

**Proposing Transaction**

After the voting period has ended, if the proposal passes and meets the criteria set in the rules of the oSnap module, the 0.000005 ETH transfer can be proposed. Anyone can propose the transactions by clicking the 'Request execution' button.

<figure><img src="../../.gitbook/assets/image (3) (2) (1).png" alt=""><figcaption></figcaption></figure>

After transactions have been proposed, a bond is taken from the Proposer and a challenge period starts where anyone can dispute a proposal. The Snapshot proposal will alert the users when the liveness period is complete which is when the transaction can be executed and the bond will be returned to the Proposer.

<figure><img src="../../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

**Executing Transaction**

After the challenge period has been completed, the Snapshot proposal gives the user the option to 'Execute transaction batch'. Signing this transaction will execute the proposal and return the bond to the proposer.

<figure><img src="../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

After executing our example proposal, the below shows the 0.000005 ETH transfer being executed.

<figure><img src="../../.gitbook/assets/image (2) (1).png" alt=""><figcaption></figcaption></figure>

