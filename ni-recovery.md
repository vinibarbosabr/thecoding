# **WHAT**

Investigating a potential solution for a problem described by Ari Eiberman on X according to [**this post**](https://x.com/AriEiberman/status/2001073643384610921)

*DISCLAIMER: I still haven't tested this method. I don't know if it's safe, nor if it works as intended. Follow the instructions below at your own risk*

# **USING**
openai/gpt-oss-120b via [NEAR Private Chat](https://private.near.ai/)

# **PROMPT**
> Look at NEAR Intents documentation. Once someone deposits a token via a contract that is not explicitly supported by app.near-intents.org, is there any way this person can recover these funds?
Could this person, for example, call the smart contract via near-cli-rs to withdraw the token without having to rely on a frontend?
Give me the path to recovery.

# **OUTPUT**

**Path to recovery for an unsupported‑token deposit on NEAR Intents**

1. **Confirm the deposit status**  
   * Call the view method `mt_batch_balance_of` on the main `intents.near` contract with the `defuse_asset_id` of the token you tried to deposit.  
   * If the balance returns 0, the contract never recognised the token; if it shows the amount, the funds are locked in the contract waiting for a withdrawal.  
   * Example CLI call (near‑cli‑rs):  

   ```bash
   near view intents.near mt_batch_balance_of '{"account_id":"your_account.near","defuse_asset_ids":["<>asset_id>"]}'
   ```  

   Once you have obtained the defuse_asset_id for the token, you can retrieve your balance by calling the mt_batch_balance_of view function on the intents.near contract.

2. **Check whether a “passive deposit/withdrawal” endpoint exists for the token**  
   * The Intents docs expose a *passive‑deposit‑withdrawal* service that returns the address to which you should have sent the token and a withdrawal status hash.  
   * If the token is not listed, the service will return `null` or an error, confirming that the token is unsupported.  
   * Docs reference:  

   Returns the address for depositing supported tokens or native currency… Returns withdrawal status. …withdrawal_hash – hash of the transaction on NEAR where ft_burn event happened on the token contract

3. **Attempt a direct withdrawal via the Verifier contract**  
   * The Verifier contract is the on‑chain component that settles intents and can return funds if a deposit fails.  
   * Issue a `withdraw` intent that burns the wrapped token (if any) and triggers a transfer back to the original token contract on its native chain.  
   * Example CLI call (near‑cli‑rs):  

   ```bash
   near call verifier.intents.near withdraw '{"defuse_asset_id":"<>asset_id>","amount":"<>amount>"}' \
        --accountId your_account.near --gas 200000000000000 --deposit 0.1
   ```  

   * The contract will check that the user’s balance in the Verifier is sufficient; if the underlying token contract failed to credit the Verifier, the Verifier will automatically credit the user’s balance back.  

   The Verifier smart contract will evaluate the intents and check whether the requests can be fulfilled, and will ensure that the transaction is done atomically… Alice and Bob can withdraw their USDC/USDT from the Verifier smart contract to their individual accounts.  
   If Alice does not receive her tokens to the smart contract in question… the Verifier should return the 100 USDC to her balance in the Verifier contract.

4. **If the Verifier rejects the withdrawal (token truly unsupported)**  
   * NEAR Intents uses **account abstraction**, which includes a built‑in recovery mechanism that lets the contract owner or a designated admin execute a “force‑refund” transaction.  
   * Because the token is not in the supported list, you will need to open a support ticket with the Intents team (or the contract maintainer) and provide:  
     - Transaction hash of the original deposit.  
     - The `defuse_asset_id` and amount.  
     - Proof that the token contract on the source chain accepted the transfer (e.g., an explorer screenshot).  

   * The team can invoke an admin‑only `admin_refund` method on the Verifier (or the underlying PassiveDeposit contract) to release the locked funds back to the sender’s NEAR account.  

   Account abstraction … allows flexible permissions, recovery options, and smart contract capabilities without adding complexity for end users.  
   This simplicity is made possible by a powerful feature called account abstraction, which allows flexible permissions, recovery options, and smart contract capabilities…

5. **Perform the admin‑refund (if approved)**  
   * Using near‑cli‑rs with the admin key (or a multi‑sig from the Intents governance), call the special refund method (e.g., `force_refund`).  
   * Example (only for authorized parties):  

   ```bash
   near call verifier.intents.near force_refund '{"account_id":"your_account.near","defuse_asset_id":"<>asset_id>","amount":"<>amount>"}' \
        --accountId admin_account.near --gas 200000000000000 --deposit 0
   ```  

   * After the transaction is mined, the funds appear in your NEAR balance and can be withdrawn to any supported asset.

6. **Withdraw the recovered NEAR to a wallet of your choice**  
   * Finally, use the standard `ft_transfer` or `near withdraw` command to move the funds out of the Intents contract into your personal wallet or a supported bridge.  

   ```bash
   near call intents.near ft_transfer '{"receiver_id":"your_wallet.near","amount":"<>amount>"}' \
        --accountId your_account.near --gas 100000000000000 --deposit 0.01
   ```

---

### Quick checklist

| Step | Action | CLI snippet | Expected result |
|------|--------|-------------|-----------------|
| 1 | Query balance | `near view intents.near mt_batch_balance_of …` | 0 → unsupported; >0 → funds locked |
| 2 | Verify passive service | Call the passive‑deposit endpoint (REST or RPC) | Returns address or error |
| 3 | Attempt normal withdraw | `near call verifier.intents.near withdraw …` | Funds returned if Verifier holds them |
| 4 | Open support ticket (if Step 3 fails) | Provide tx hash, asset id, amount | Intents team authorises admin refund |
| 5 | Admin‑refund (team executes) | `near call verifier.intents.near force_refund …` | Funds appear in your NEAR account |
| 6 | Move funds out | `near call intents.near ft_transfer …` | Tokens in your personal wallet |

---

**Bottom line:**  
Even when a token is not listed on `app.near‑intents.org`, the underlying smart contracts still allow a recovery path. You can diagnose the deposit with the `mt_batch_balance_of` view, try a normal withdraw via the Verifier contract, and, if that fails, request an admin‑controlled refund from the Intents maintainers. All of this can be performed from the command line with **near‑cli‑rs**, without needing any front‑end UI.

*Would you like me to expand on any of the CLI commands or show a concrete example of the admin‑refund transaction?*

