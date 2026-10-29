# Invoice financing

A verified business lists an invoice, an eligible investor funds it at a discount, repayment releases the funds.

<!-- assets/rehearsal.gif goes here: the ten-step rehearsal on a local fork, 15 s -->

1 contract, two gates · 21 tests · 10-step rehearsal in 15 s · deployed on testnet in 1 block · 1,100,139 gas · 344 test RBNT · verified on Routescan · its whole life run on testnet through real adapter code

```sh
npm run dev -- --no-web --port 8548 --request-id 708   # forks testnet locally, seeds five wallets in the five credential states
scripts/rehearse-invoices.sh                            # the whole life of an invoice, ten steps, fourteen refusals, nothing signed
scripts/rehearse-live-shape.sh                          # the same against a BusinessIdentifier adapter and an Iden3 verifier adapter
cd contracts && forge script script/Deploy.s.sol --rpc-url redbelly_testnet --account <keystore-name> --sender <your-address> --broadcast
                                                        # the one place a key is used: yours, in your keystore, never read by anything here
```

**The gate is in the contract, not the wallet.** Two gates on one contract. `list` asks the business's own
identifier whether the caller is its delegate or representative, a live role read and not a proof. `fund` and
`release` ask the network's identity verifier about the investor against request 708, the AU wholesale investor
recipe. A wallet cannot talk its way past either, and neither can a front end.

**The check runs every time.** An investor whose credential is revoked after funding cannot take the
repayment. The face value waits in the contract until the credential is back, and a delegate the business has
removed is refused on its very next listing, with no window.

**Nothing deploys without the ship report.** Pre-flight, the gas report, the tests in all five credential states
and the Slither check write one file. The deploy script will not broadcast on mainnet without that file dated
today, and will not take a mock verifier or a mock stablecoin there at all. On testnet it warns and lets you learn.

## Live on testnet

Chain 153, deployed and verified on 25 October 2026, then used the same afternoon.

| | |
|---|---|
| `InvoiceFinancing` | [`0xcd7F22Ddc8d22224A98f1c98c601F890cb3F3d90`](https://redbelly.testnet.routescan.io/address/0xcd7F22Ddc8d22224A98f1c98c601F890cb3F3d90/contract/153/code) |
| deploy transaction | `0x8dbf4ec2576b9cabb616e8ddc75a18e4ffbeb20419442ef9ba691c324eceb455`, block 3,056,722 |
| gas | 1,100,139 at a base fee of 312,461 gwei, 343.75 test RBNT, about 52 US cents at that day's feed price |
| its stablecoin | [`0x9cF4E639C35141cF4C36E5c38dc88151FB89E6Db`](https://redbelly.testnet.routescan.io/address/0x9cF4E639C35141cF4C36E5c38dc88151FB89E6Db/contract/153/code), a `MockStablecoin` (mUSD, six decimals, anyone mints), 508,568 gas in the same block |
| lister gate | [`0x6a9d3ed5A4082ff9C0287C211AD227657D0AF839`](https://redbelly.testnet.routescan.io/address/0x6a9d3ed5A4082ff9C0287C211AD227657D0AF839/contract/153/code), a `BusinessDelegateAdapter` over a [`MockBusinessIdentifier`](https://redbelly.testnet.routescan.io/address/0x87B7094051d38d00a150E394312c08C7700b6BF1/contract/153/code), request 0 (delegate or representative) |
| investor gate | [`0x3D1920190f2D60571eC9ccdA945447fAeF7EFc20`](https://redbelly.testnet.routescan.io/address/0x3D1920190f2D60571eC9ccdA945447fAeF7EFc20/contract/153/code), the kit's `Iden3VerifierAdapter` over a [`MockZKPVerifierV2`](https://redbelly.testnet.routescan.io/address/0x68E07D5a913F9988240582a431acc1AEA4212461/contract/153/code), request 708 |
| record and report | `contracts/deployments/153-InvoiceFinancing.json`, `contracts/deployments/ship-153-2026-10-25.md` |

The identifier and the ZKP verifier are mocks, because no business of mine is verified on the network yet and no
request 708 has been issued. The two adapters are the code a real deployment would run, and the contract reads
through them. It was deployed with a shared `ReceptorMock` on both gates, then the gates were
repointed at the adapters in one broadcast, which is the owner's right and nobody else's; the transaction that
proves a stranger cannot is in the transcript. Then one invoice went through its whole life on testnet in nineteen
transactions: listed at 48,250.00 mUSD and 250 bps, funding refused while the investor had no proof, funded once
the proof was recorded, repaid, release refused while the proof was withdrawn with the face value held in the
contract, released when the proof came back, and a second delegate added and removed on the identifier with the
gate's answer read in between. Every refusal was read against the live network, not a fork. Nothing here is on
mainnet, and I will not put a mock there; the deploy script refuses it for me.

## Install

| You need | Why |
|---|---|
| Node 22 or later | The scripts and the web app; `npm run doctor` checks it |
| [Foundry](https://getfoundry.sh) | `forge`, `anvil` and `cast`: build, test, fork, deploy |
| A wallet verified on the network, for step 4 only | The network refuses any other sender, and so does the script's own pre-flight |

`npm run doctor` checks the first two and tells you what is missing, one line each with the fix.

1. **Install**, once per clone:

   ```sh
   npm install
   npm run contracts:install
   ```

2. **Test**:

   ```sh
   npm run test        # 89 tests: the scaffold's 68 and 21 for the invoices, one a fuzz over face values and discounts
   ```

3. **Rehearse** on a local fork, two shells, nothing signed:

   ```sh
   npm run dev -- --no-web --port 8548 --request-id 708
   ```

   ```sh
   scripts/rehearse-invoices.sh
   ```

   A test stablecoin, the contract against the seeded verifier, a delegate and a removed delegate on the lister
   gate, a 10,000 mUSD invoice at 300 bps listed, funded with the advance landing in the business's wallet, repaid
   and released, and fourteen refusals, each the error the contract names for that wallet and that request.
   `scripts/rehearse-live-shape.sh` runs the same life against the two adapter shapes above, with realistic
   values, the due date passing and a delegate removed mid-flight. Save the output outside the project: a
   document hash looks like a key to the secret scanner.

4. **Deploy** to testnet with your own keystore. Make it once with `cast wallet import <keystore-name>
   --interactive`, verify that wallet on the network, fund it from the faucet, then:

   ```sh
   npm run preflight
   npm run ship -- --chain 153 --account <keystore-name>
   cd contracts && forge script script/Deploy.s.sol --rpc-url redbelly_testnet --account <keystore-name> --sender <your-address> --broadcast
   ```

   `VERIFIER` is the investor gate's verifier, `BUSINESS_VERIFIER` the lister gate's and `STABLECOIN` the
   token invoices settle in. Set any of them to a contract you already have and the script uses it. Leave one
   unset on testnet and the script deploys a mock for it and says so in capitals; on mainnet it refuses.
   `REQUEST_ID` defaults to 708 and `LISTER_REQUEST_ID` to 0.

## The flow

| When | Command | What you get |
|---|---|---|
| once per machine | `npm run doctor` | What is missing, with the fix |
| once per clone | `npm install && npm run contracts:install` | The contracts and the web app ready to build |
| every change | `npm run test` | 89 tests, both gates in all five credential states |
| before a deploy | `npm run dev` and `scripts/rehearse-invoices.sh` | The whole life of an invoice on a fork, in fifteen seconds |
| before a deploy | `npm run ship -- --chain 153 --account <keystore-name>` | One report: pre-flight, gas, tests, analysers. On mainnet, the gate |
| the deploy | `forge script ... --broadcast` | Here: one block, 1,100,139 gas for the contract, 343.75 test RBNT |
| after the deploy | `forge verify-contract ...` | Source on Routescan; the record carries the arguments already encoded |
| in use | `list`, `fund`, `repay`, `release` | An invoice whose every step is an event on chain |

<details>
<summary>What the contract does, action by action</summary>

- **`list(faceValue, discountBps, dueDate, documentHash)`**, a delegate or representative of the business only,
  through the lister gate. Face value in the stablecoin's units, discount from 1 to 9,999 basis points, a due date
  in the future, the keccak256 of the invoice document. Nothing about the debtor goes on chain.
- **`fund(id)`**, an eligible investor only, through the investor gate. Pays the discounted advance straight to the
  wallet that listed, in the stablecoin, and becomes the invoice's investor. Approve the advance first.
- **`repay(id)`**, anyone: the business, its debtor, a guarantor. Moves the face value into the contract. No gate,
  because a credential is a condition on receiving money here, not on paying it.
- **`release(id)`**, the recorded investor only, still eligible. Takes the face value out.

Every invoice, status and amount can be read back, and every action emits an event. Two choices I have made and
not yet changed. The advance goes to the wallet that listed; on the live shape that is a delegate, a service
account, and a payee set by a representative is the change I would make before this held real money. The due date
is recorded for the investor's judgement and not enforced; a late repayment is accepted, and overdue handling with
a cancel for an unfunded listing is the second change. The discount rounds in the business's favour.

</details>

<details>
<summary>Why this network, and what the contract would be anywhere else</summary>

The contract checks both sides, with no back office in between. The seller is a delegate of a verified
business, read live from the business's own identifier contract, and the funder passes the wholesale investor gate.

| The same contract | Who checks the two sides | What a removed delegate or a revoked credential does |
|---|---|---|
| on a chain with no identity layer | nobody, or a front end anyone can bypass | nothing |
| with allow-lists the platform keeps | the platform, by hand, off chain | whatever the platform remembers to do |
| here | the contract, against the identifier and the network's verifier, on every call | the next listing, funding or release is refused, on chain |

</details>

## Docs

| | |
|---|---|
| 01. The run record | `docs/01-run-record.md`: what was built, what the fork run found, the live-shape rehearsal and its parameter table, what was not done |
| 02. The deploy | `contracts/deployments/ship-153-2026-10-25.md`: the report that gated it, every check and figure |
| 03. The rehearsal | `docs/03-rehearsal.txt`: the ten steps as they ran, gas and status per transaction |
| 04. The life on testnet | `docs/04-live-on-testnet.txt`: the nineteen transactions, every hash, every refusal read live |
| 05. The threat model | `THREAT-MODEL.md` |
| 06. The recipes | `recipes/README.md`: the eligibility recipes the kit ships, and why this contract uses `business-delegate` and `au-wholesale-investor` |

## Development

```sh
git clone https://github.com/gatedpath/invoice-financing && cd invoice-financing
npm install && npm run contracts:install
npm run test
```

Issues and pull requests are welcome; `CONTRIBUTING.md` is short. If you have found a security problem,
please read `SECURITY.md` first. This is built on my kit at `npm create redbelly-dapp` and it is an
independent project: Redbelly did not make it and is not associated with it. Their developer portal is
[Vine](https://vine.redbelly.network), and that is where the network's own facts live. MIT.

<details>
<summary>The scaffold's own README, as generated</summary>

<!-- the template README goes here, unchanged except the title -->

</details>
