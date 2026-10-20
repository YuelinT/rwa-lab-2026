# STUDENT-QUESTIONS.md — Discussion questions (goes inside your submission zip)

Answer directly under each question. 150–300 words each — **reasoning over length**.

The sections follow the four planks of the bridge. There are no standard answers; these are the
real point of the lab.

---

## A. Custody — where the asset actually is (plank a)

**A1.** In Lab 1, `totalCollateral()` read the vault's own ERC-20 balance, and the chain could
prove the invariant. Here, `MockTBillCustodian.realHoldings()` is just a number that a permissioned
address can move. Write the equivalent of `totalCollateral() == totalSupply()` for *this* system.
What does it actually assert, and what does it no longer prove?

> Your answer:
>
> The closest solvency relation for this system is:
>
> `vault.reserveBalance() + custodian.realHoldings() >= vault.totalClaimValue()`.
>
> The left-hand side represents the fund’s reported resources. `reserveBalance()` is the liquid USDC still held on-chain, while `realHoldings()` represents the value reportedly moved into off-chain T-Bills. The right-hand side is the total value promised to all tBILL holders, calculated from the outstanding share supply and the latest attested NAV. If the fund has no surplus, fees, or timing differences, the relation could be written with equality. Using `>=` is more practical because a solvent fund may hold excess reserves.
>
> If all inputs are honest and measured consistently, this relation asserts accounting solvency: the reported cash and T-Bill holdings are sufficient to cover the current value of outstanding claims. However, unlike Lab 1’s on-chain collateral invariant, it does not prove the existence or ownership of the underlying assets. `realHoldings()` is only an integer that an address with `CUSTODIAN_ROLE` can change, and `totalClaimValue()` depends on a NAV supplied by `REPORTER_ROLE`. The chain cannot inspect a custody account, confirm legal title, identify encumbrances, or verify that the T-Bills can be liquidated promptly. A mistaken, compromised, stale, or colluding operator could therefore make the equation appear healthy without changing any real asset. The relation is useful for reconciliation, but external audits, independent custody statements, and legal ownership records remain necessary.


**A2.** The T-Bills are held by an SPV, a legal entity set up so the fund's assets are bankruptcy-
remote from the issuer. Explain in your own words why a *legal* structure is part of a *technical*
design. If the SPV's custodian goes bankrupt, what does the on-chain token entitle its holder to?

> Your answer:

> The SPV is part of the technical security model because smart contracts cannot create or enforce ownership rights over off-chain T-Bills. The SPV should legally own the securities, while a custodian holds them on the SPV’s behalf. This separates the fund assets from the token issuer’s ordinary business and liabilities. If the issuer becomes insolvent, bankruptcy remoteness is intended to prevent the issuer’s creditors from claiming the T-Bills. Without this legal separation, the contracts could continue displaying correct balances and NAV values even though token holders had no enforceable priority over the underlying assets.
>
> If the custodian becomes bankrupt, a tBILL holder does not automatically own or control a specific Treasury security. The holder owns an on-chain share whose rights depend on the fund documents, custody agreement, shareholder register, and applicable insolvency law. Properly segregated client assets should normally remain outside the custodian’s bankruptcy estate, allowing the SPV to recover them or transfer them to another custodian. Nevertheless, recovery may be delayed, disputed, or reduced if records are inaccurate, assets were commingled, or securities are missing. Redemptions could therefore be suspended even while the smart contracts continue functioning normally.
>
> The design should combine segregated custody accounts, independent reconciliation, periodic audits, clear beneficial-ownership terms, insurance, and emergency governance. Code can manage token balances and redemption tickets, but only the legal structure determines what those claims are worth when an off-chain institution fails.

---

## B. Attestation — how the chain learns the truth (plank b)

**B1.** `attest()` is the only function that moves the NAV, and it is gated by `REPORTER_ROLE`.
Everything downstream — subscription pricing, redemption payouts, both invariants — believes that
number. Name the failure modes of a single trusted reporter, and one mechanism (technical,
financial, or legal) that mitigates each.

> Your answer:
>
> A single trusted reporter creates four main failure modes. First, key compromise allows an attacker to post an arbitrary NAV. Because the vault uses the same value for subscriptions and redemptions, one false update can reprice the entire fund. Threshold signatures and hardware security modules can reduce this risk. A quorum of independent operators should approve every report.
>
> Second, an honest reporter may make an operational error. The wrong source, decimal scale, timestamp, or sign can produce a large pricing error. The contract should apply range checks, rate-of-change limits, and two-person review for exceptional updates. A circuit breaker should stop transactions when the new value deviates materially from recent observations.
>
> Third, the reporter may fail or become unavailable. The last value then remains on-chain after it has lost economic relevance. The feed should publish updatedAt, and the vault should reject reports older than a defined maximum age. Separate controls should suspend subscriptions and redemptions until a fresh price is confirmed.
>
> Fourth, the reporter may act dishonestly or face a conflict of interest. An operator might overstate NAV to support redemptions or understate it before a related party subscribes. Independent fund administration, periodic audits, public reconciliation reports, contractual liability, and an economic bond can improve accountability.
>
> Finally, collusion remains possible even with several signers. The signers should therefore represent separate institutions and use different data sources. Governance should document escalation, replacement, and emergency procedures. A robust design combines technical validation, institutional independence, economic incentives, and legal enforcement.

**B2.** A production feed would check whether `updatedAt` is stale; this lab's mock does not. A
**frozen** NAV is a different attack from a **wrong** NAV. Describe how a stale-but-honest NAV can
be exploited by someone who knows the true value has moved.

> Your answer:
>
> A stale-but-honest NAV was accurate when published, but it no longer reflects the current value. The vulnerability comes from asymmetric information and transaction timing, not from a false report.
>
> If the true NAV falls while the recorded NAV remains high, an informed holder can enqueue a redemption at an inflated price. Because RedemptionQueue locks assetsLocked at enqueue time, a later price correction does not reduce the promised payout. Early redeemers extract more USDC than the economic value of their shares. The resulting shortfall is borne by remaining holders or later claimants. This creates a first-mover advantage and may accelerate a run.
>
> If the true NAV rises while the recorded NAV remains low, a trader can subscribe cheaply. The vault calculates shares from the stale price, so the trader receives more shares than the correct NAV would permit. Once the feed updates, the value of those shares rises. Existing holders are diluted. The trader may later redeem at the corrected NAV and capture a transfer of value from the fund.
>
> This risk is greater during market stress, weekends, rapid interest-rate changes, or operational outages. A whitelist does not prevent it because the trader may be fully KYC-verified and authorised.
>
> A production feed should provide both the price and updatedAt. The vault should enforce a maximum age and reject transactions when the feed is stale. Subscriptions and redemptions should have separate pause controls. Additional measures include forward pricing at the next confirmed NAV, delayed settlement, deviation limits, independent secondary sources, and transaction cut-off times. Existing queue tickets also need an explicit policy: preserve the locked price, reprice the ticket, or allow cancellation. This choice determines who bears the NAV timing risk.

---

## C. Redemption — T+1 does not settle on-chain (plank c)

**C1.** The queue locks the payout at the NAV of **enqueue** time, not **settle** time. If the NAV
rises between the two, who gains and who loses — the redeemer, or the holders who stayed? Is that
the right allocation of risk, and how would you change it?

> Your answer:
>
> The current design fixes the redemption value when a request enters the queue. This gives the redeemer certainty, but it transfers NAV risk during the T+1 period.
>
> If NAV rises after enqueue, the redeemer receives the lower, earlier value. Holders who remain in the fund benefit because the appreciation stays with the remaining portfolio. If NAV falls, the result is reversed. The redeemer keeps the higher locked value, while remaining holders absorb the loss. A well-informed investor may therefore enqueue before an expected decline. This creates a timing advantage and may encourage early withdrawals during stress.
>
> The allocation is defensible when the fund promises a fixed redemption quote. It also prevents the issuer from reducing a payout after the investor has surrendered the shares. However, it is not fully neutral. The underlying T-Bills are sold after the request, so the final proceeds may differ from the locked amount. Other holders then bear this mismatch.
>
> I would use forward pricing. The queue would escrow the shares and record the request time, but it would not immediately fix the final USDC amount. The payout would use the next independently confirmed NAV after the daily cut-off or after the asset sale. Requests from the same pricing window could be processed as a batch. FIFO order could still determine settlement priority. A liquidity fee or swing-pricing adjustment could allocate transaction costs to redeeming investors. The rules should also define cancellation rights and extreme-deviation limits. This approach reduces timing arbitrage and aligns the redemption price more closely with the value realised by the fund.

**C2.** In a 2008 money-market fund breaking the buck, and in USDC's 2023 depeg, redemptions were
handled very differently. Compare the two. For a tokenized T-Bill fund, what does it mean to "close
the redemption channel", and what should happen to the queue when it does? Then look at the lever
this lab actually ships: `pause()` (`PAUSER_ROLE`) freezes transfers, minting **and** redemption
together — one switch, no partial setting. Why is that the wrong tool for a liquidity squeeze, and
what would you rather have?

> Your answer:
>
> The 2008 money-market fund crisis and the 2023 USDC depeg illustrate different responses to loss of confidence. In 2008, a fund that broke the buck suspended redemptions. Investors could not exit, even though the portfolio still contained assets. The measure protected the remaining pool from a disorderly liquidation, but it trapped investors and intensified uncertainty. In 2023, USDC traded below one dollar after part of its reserves became inaccessible during the Silicon Valley Bank failure. The official redemption commitment remained available, and confidence recovered after the reserve position became clearer. Continued convertibility helped anchor the market price.
>
> For a tokenized T-Bill fund, closing the redemption channel should mean suspending new redemption requests when liquid cash is unavailable or the NAV cannot be trusted. It should not delete existing tickets. Their ownership, order, and locked terms must remain auditable. Tickets that are already settled should remain claimable. Unsettled tickets should keep their FIFO position and resume settlement when cash returns. The fund should publish the reason, expected duration, and available reserves.
>
> The lab’s global pause is too broad for a liquidity squeeze. It blocks transfers, minting, and the token movements required for redemption. A cash shortage does not necessarily imply that ownership transfers or subscriptions are unsafe. New subscriptions may even add liquidity. A global pause may also prevent orderly exits and increase panic.
>
> I would introduce separate controls for subscriptions, transfers, new redemption requests, settlement, and claims. Already funded claims should never be blocked merely because new redemptions are suspended. Each control should have a defined trigger, a responsible role, an expiry time, and public disclosure. Emergency actions should require multisignature approval and later review.


---

## D. Admission — the guest list and the backdoor (plank d)

**D1.** Every balance change in `TBillToken._update` checks **both** endpoints against the
whitelist. That is what stops a sanctioned address from receiving shares — and also what stops a
completely ordinary user who has not finished KYC. Where is the right line for a regulated fund,
and who should be able to move it?

> Your answer:
>
> A regulated fund should restrict who may receive and hold its shares, but the restriction should be proportionate to the relevant risk. Before an address receives tBILL, the operator should verify identity, sanctions status, jurisdiction, investor eligibility, and beneficial ownership. These checks support securities law and anti-money-laundering obligations. The status should also be reviewed periodically because eligibility can change.
>
> However, a single binary whitelist is too coarse. It treats an unverified applicant, a sanctioned person, an expired account, and a temporarily reviewed customer in the same way. The fund should use several states, such as pending, active, restricted, frozen, and redeem-only. A restricted holder might be prevented from receiving new shares while still being allowed to redeem through a controlled process. This reduces unnecessary deprivation of property and avoids trapping legitimate investors.
>
> No single employee or issuer wallet should control admission. Routine approval should be operated by a regulated transfer agent or compliance team. Changes should require a multisignature process with independent reviewers. An emergency officer may receive authority to impose a short temporary freeze, but the restriction should expire automatically unless a higher quorum confirms it. Permanent restrictions or seizures should require documented legal grounds.
>
> Every status change should emit an event with a reason code. Supporting evidence should be retained off-chain to protect personal data. The system should also support key rotation, periodic access reviews, and an appeal process. This structure preserves regulatory control while limiting arbitrary censorship and operational mistakes.



**D2.** A regulated fund asks for two levers: freeze an address (`setWhitelisted(addr, false)`) and
burn a balance (`MINTER_ROLE`) — both censorship tools, both justified by the product. But Ex4.3
shows the two **collide**: once an address is off the list, `burn()` reverts as well, because
`_update` guards both endpoints. So a freeze also blocks a seizure. Argue **both** sides of handing
an issuer these powers, then say what safeguards you would add and who should hold each key.

> Your answer:
>
> Giving an issuer freeze and burn powers can be justified in a regulated fund. A freeze may be required when an address is sanctioned, compromised, or subject to a court order. A forced burn or seizure may correct an erroneous issuance, reverse a legally invalid transfer, or move assets from a prohibited holder into a controlled account. These powers can protect other investors and help the fund comply with law.
>
> The same powers also create serious risks. A compromised issuer key could freeze many holders or destroy balances. An insider could target a customer without due process. Operational mistakes may become irreversible. Broad censorship powers also weaken the assurance that a token balance represents stable property rights. Ex4.3 reveals an additional design failure: removing Alice from the whitelist blocks both her transfer and the issuer’s burn. The freeze therefore prevents the enforcement action it was intended to support.
>
> The contract should separate these powers. A compliance multisig operated by the transfer agent should control temporary freezes. A different multisig, including an independent trustee or legal officer, should control seizure. The general `MINTER_ROLE` should not automatically include forced burning. A dedicated `seize` function could move shares to an escrow address after checking a case identifier and authorisation.
>
> Emergency freezes should be time-limited. Permanent seizure should require a higher approval threshold, documented legal authority, and an auditable delay unless immediate action is legally necessary. Every action should emit a reason-coded event. Affected holders should receive notice and access to an appeal process. Regular key rotation, hardware wallets, transaction limits, and independent audits would further reduce misuse.

---

## E. Tests (Tier 1 required — this is Ex2, Ex3 and Ex4)

Turn the red tests green in `test/exercises/01_BridgeTasks.t.sol` to cover the scenarios below, and
write your test function names here:

| Plank | Scenario | Your test function name |
|---|---|---|
| a | Shares exist while the custodian holds nothing | `test_Ex2_SharesExistWhileTheCustodianHoldsNothing` |
| a | `realHoldings()` moves with no cash moving | `test_Ex2_RealHoldingsIsJustANumber` |
| b | The claim value floats while the share count stays | `test_Ex3_TheClaimFloats_TheShareCountDoesNot` |
| b | One `attest` re-prices every holder at once | `test_Ex3_OneCallMovesTheWholeBook`|
| d | A non-whitelisted address cannot receive shares, even by paying | `test_Ex4_NoKyc_NoShares_EvenIfYouPay`|
| d | A whitelisted holder cannot send shares to a non-whitelisted address | `test_Ex4_TransferChecksBothEndpoints` |
| d | Removing an address freezes that holder — and blocks a burn | `test_Ex4_FreezeBeatsConfiscation`|

Then write one more scenario you consider **most likely to be exploited** on a tokenized fund, and
say which plank it breaks:

> Your answer:
>
> The most likely exploitation is the compromise or misuse of `REPORTER_ROLE`. This breaks plank (b), attestation. The system uses one reported NAV for subscription pricing, redemption pricing, and the total value of outstanding claims. It has no staleness check, deviation limit, or independent confirmation.
>
> A malicious reporter could first submit an artificially low NAV. The attacker could then subscribe with USDC and receive more tBILL shares than the correct price would permit. The reporter could next submit a much higher NAV. The attacker could enqueue those shares for redemption at the inflated value. Because the queue locks `assetsLocked` at enqueue time, a later correction would not reduce the promised payout. If settlement occurs before the manipulation is detected, the attacker extracts USDC from the fund. Remaining holders or later redeemers bear the shortfall.
>
> This scenario is credible because it does not require breaking the token contract or its invariants. The contracts may execute exactly as designed while relying on false external information. Similar harm could also arise from a stale NAV and an informed trader. Appropriate controls include threshold signatures, independent data sources, hardware-protected keys, maximum price age, deviation limits, and separate emergency controls for subscriptions and redemptions.