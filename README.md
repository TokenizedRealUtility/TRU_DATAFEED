# TRU DataFeed: a beginner’s guide

**Put a small observation on TRU, or put a fingerprint of a larger batch on TRU. Retrieve it later and check that the data matches.**

This guide matches **TRU_DATAFEED_05_DUAL_CORE_PINNED**, the datafeed implementation prepared for Core 0.08.1. It covers HAM radio, science experiments, individual records, batches, retrieval and search. All example observations are synthetic test data.

## What happens, in plain English?

1. **Queue:** put observations in a local waiting list. Free; no transaction.
2. **Preview:** see what would be published. Free; no transaction.
3. **Publish:** your Core wallet signs and submits one transaction. This spends a fee.
4. **Wait:** a miner includes the transaction in a block.
5. **Get and verify:** retrieve the record and check its data fingerprint and current confirmation.

A **feed** is a label such as `ham-demo` or `lab-cooling`. A **record** is one observation. A **batch** is one or more records published together. A **TXID** is the transaction’s lookup number. A **hash** is a fingerprint calculated from the data.

## Are these OP_RETURNs or TRUScripts?

**New DataFeed publications use compact OP_RETURN outputs in ordinary signed TRU transactions. They are not wrapped as TRUScripts.** Their application payload begins with `TRUDF1:`.

The earlier publisher used a TRUScript JSON wrapper, itself carried in OP_RETURN. The new publisher removes that extra wrapper so small feeds fit the existing relay size limit. Its reader can also retrieve older compatible TRUScript-wrapped feed records.

Use the DataFeed commands to retrieve new records; do not expect them to appear as newly created TRUScript assets in a script vault. Neither a new token nor an AI agent is required. This publication format does not change consensus rules.

## Pick your storage mode

| Mode | What goes on-chain? | Can someone retrieve the observations from TRU? | Best for |
|---|---|---|---|
| `inline` | The observations and their batch hash | Yes, for the published inline records | A short measurement or very small batch |
| `hash` | The feed label, record count and batch hash | Only the commitment; original records must be supplied separately | Larger batches or data you want to keep off-chain |

The complete payload, including its envelope, must fit **254 UTF-8 bytes**. That fits the existing 257-byte generic OP_RETURN script limit. This is not 254 bytes per record. Short feed names leave more room for data.

A hash is not an encrypted copy and cannot reconstruct lost observations. The tool hashes an ordered JSON array of records; it does not build a Merkle tree or hash the original CSV file’s raw bytes.

## 1. What you need

- The DataFeed 05 patch installed in your Core tree, rebuilt, and the upgraded Core running.
- Python 3.9 or later and a working authenticated `tru-cli` connection to your local node.
- For publishing: an unlocked Core wallet and a funded wallet address with a mature, spendable ordinary TRU output.
- For reading: the publishing wallet and its keys are not needed.

If installing the patch for the first time, follow the package’s installer README. It requires its matching source baseline. Do not bypass a drift error or install the earlier DataFeed repair 04 alongside it.

To permit explicit wallet publication, add or update this setting in the **existing** `[network]` section of the configuration your node uses:

```ini
[network]
TRU_RPC_WALLET_SEND_ENABLE=1
```

Do not duplicate the section or replace your other settings. Restart Core after changing it; this configuration change alone does not require another rebuild. Publication uses privileged authenticated local RPC, not the public explorer endpoint.

## 2. Set up your terminal

Choose your actual Core directory on the first line. For Burton’s running node, use `$HOME/NEW_TRU`; another installation may use `$HOME/TRU-Core`.

```bash
export CORE_ROOT="$HOME/NEW_TRU"
export CORE_BUILD="$CORE_ROOT/build-native"
export TRUCLI="$CORE_BUILD/bin/tru-cli"
export DF="$CORE_ROOT/tools/datafeed/tru_datafeed.py"
export DF_DB="$HOME/.local/share/tru/datafeed-v1.sqlite3"
export DF_INDEX="$HOME/.local/share/tru/datafeed-v2-index.sqlite3"

df() {
  python3 "$DF" --cli "$TRUCLI" \
    --db "$DF_DB" --index-db "$DF_INDEX" "$@"
}

mkdir -p "$HOME/tru-datafeed-examples"
cd "$HOME/tru-datafeed-examples"

"$TRUCLI" raw getdatafeedinfo '{}'
```

Adjust `CORE_BUILD` if your configured build directory is named differently. Run subsequent commands in this same shell; repeat setup in a new terminal. Here, `df` is a shell shortcut for the Python tool and temporarily takes precedence over Linux’s disk-space `df` command. `/usr/bin/df` still accesses that system command.

Expected capabilities:

```json
{
  "automatic_broadcast": false,
  "fee_cap_enforced": true,
  "format": "TRU_DATAFEED_RPC_V1",
  "max_payload_bytes": 254,
  "wallet_publish_enabled": true
}
```

`automatic_broadcast: false` is normal. Explicit publication still works. If `wallet_publish_enabled` is false, correct the running Core’s configuration before attempting a publication.

## 3. Example A: publish one HAM radio observation

### A. Give this test a fresh feed label

```bash
export HAM_FEED="ham-$(date +%s)"
echo "$HAM_FEED"
```

Keep this value for the rest of the test. Do not create a new feed merely to retry an uncertain submission.

### B. Queue a synthetic reception report

```bash
df queue --feed "$HAM_FEED" \
  --record '{"test":true,"station":"HAM-01","frequency_mhz":145.5,"signal_dbm":-46.7}'

df list --feed "$HAM_FEED"
```

`QUEUED=1` means it is in your local waiting list. No funds have been spent. Running `queue` again adds another observation; it does not replace the first one.

Replace the field names and values with your own data when you are ready. Use explicit units in names, as shown above. For real time-series data, include your own measurement timestamp; a confirming block’s time is not the sensor’s measurement time.

### C. Preview it

```bash
df publish --feed "$HAM_FEED" --mode inline --batch-size 1
```

Expect `DRY_RUN=YES; BROADCAST=NO`. If the payload is too large, use hash mode for the already queued record or reduce your intended data before queueing a new record. Do not increase the limit beyond the implementation’s maximum.

### D. Select the paying address

```bash
"$TRUCLI" raw listaddresses '[]'
read -r -p 'Paste your FULL funded wallet address beginning with T: ' DF_OWNER
printf 'Selected address: %s\n' "$DF_OWNER"
```

**Paste the whole address, not `0`, `1`, or an address-list number.** Listing an address shows wallet ownership, not spendable balance at that address. A token-control output cannot fund this fee.

### E. Publish once

This is the step that signs and submits a transaction:

```bash
df publish --feed "$HAM_FEED" --mode inline --batch-size 1 \
  --owner "$DF_OWNER" --max-fee-atoms 100000 --broadcast
```

`100000` atoms is a cap of **0.001 TRU**, not the fee requested. Core calculates the fee and refuses an amount above the cap before signing. The supplied wallet has a 10000-atom base fee; the actual requirement depends on its policy calculation.

Successful submission prints a `BATCH_ID` and `TXID`, with `STATUS=OBSERVED`. Save both. `OBSERVED` is not confirmation.

**If submission errors, stop and see Troubleshooting. Do not keep repeating `--broadcast`.**

### F. Retrieve and verify

Enter the values printed by the successful publication:

```bash
read -r -p 'Paste the successful publication TXID: ' HAM_TXID
read -r -p 'Enter its BATCH_ID number: ' HAM_BATCH_ID

df get "$HAM_TXID"
df verify "$HAM_TXID"
```

Before mining, `verify` reports incomplete verification and exits nonzero. Wait and repeat only `df verify`. After confirmation, expect:

```json
"inline_hash_valid": true,
"chain_confirmed": true,
"verified": true
```

Now associate that confirmed transaction with your journal batch:

```bash
df reconcile --feed "$HAM_FEED" --batch-id "$HAM_BATCH_ID" --txid "$HAM_TXID"
```

Reconciliation updates your local journal. It does not submit another transaction.

### G. Pull out only the original observations

```bash
df get "$HAM_TXID" |
python3 -c 'import json,sys; print(json.dumps(json.load(sys.stdin)["records"], indent=2))'
```

Save the full retrieval report to a new file:

```bash
df get "$HAM_TXID" --output "$HAM_FEED-receipt.json"
```

Output commands refuse to overwrite an existing file. Choose a new filename when saving another report.

## 4. Example B: publish a batch of science measurements

Imagine recording how a beaker of water cools. These are **synthetic demonstration values**, not an actual experiment. We will publish one hash for three observations and keep the original measurements locally.

### A. Create and queue the dataset

```bash
export SCI_FEED="lab-$(date +%s)"
export SCI_INPUT="$SCI_FEED-input.json"

cat > "$SCI_INPUT" <<'JSON'
[
  {"test":true,"experiment":"cooling-demo","sample":1,"elapsed_s":0,"temperature_c":65.2},
  {"test":true,"experiment":"cooling-demo","sample":2,"elapsed_s":60,"temperature_c":61.8},
  {"test":true,"experiment":"cooling-demo","sample":3,"elapsed_s":120,"temperature_c":58.9}
]
JSON

df queue --feed "$SCI_FEED" --file "$SCI_INPUT"
df list --feed "$SCI_FEED"
```

Expect three queued records. A top-level JSON array is imported as separate records; a single JSON object is one record.

### B. Preview, then publish one batch

```bash
df publish --feed "$SCI_FEED" --mode hash --batch-size 3
```

If the preview succeeds, publish deliberately:

```bash
df publish --feed "$SCI_FEED" --mode hash --batch-size 3 \
  --owner "$DF_OWNER" --max-fee-atoms 100000 --broadcast
```

This selects the three oldest queued records in that feed and submits **one transaction**, not three. Save its identifiers:

```bash
read -r -p 'Paste the science batch TXID: ' SCI_TXID
read -r -p 'Enter the science BATCH_ID number: ' SCI_BATCH_ID
```

### C. Export the exact batch for safekeeping

```bash
export SCI_BATCH_FILE="$SCI_FEED-batch-$SCI_BATCH_ID.json"

df export-batch "$SCI_BATCH_ID" --output "$SCI_BATCH_FILE"
```

Keep the exported array and TXID together. This export captures exactly the selected records, in the selected order—even when the original input file contained more records than the publication selected.

### D. Verify after mining

```bash
df get "$SCI_TXID"
df verify "$SCI_TXID" --file "$SCI_BATCH_FILE"
```

For hash mode, `get` normally shows `records: null`: the measurements were not published inline. Verification with the original batch should show:

```json
"provided_data_hash_valid": true,
"chain_confirmed": true,
"verified": true
```

Once confirmed:

```bash
df reconcile --feed "$SCI_FEED" --batch-id "$SCI_BATCH_ID" --txid "$SCI_TXID"
```

### E. Prove that changing a reading is detected

Create a separate altered copy, preserving the original:

```bash
python3 - "$SCI_BATCH_FILE" "$SCI_FEED-altered.json" <<'PY'
import json, sys
from pathlib import Path
records = json.loads(Path(sys.argv[1]).read_text())
records[0]["temperature_c"] = 999.0
with Path(sys.argv[2]).open("x") as f:
    json.dump(records, f)
PY

df verify "$SCI_TXID" --file "$SCI_FEED-altered.json"
```

**This verification should fail.** It demonstrates that the changed dataset does not match the recorded commitment. No transaction or fee is involved in this test.

## 5. Other ways to supply data

### Import CSV

```bash
cat > sensor-demo.csv <<'CSV'
station,temperature_c,humidity_percent
LAB-01,21.4,55
LAB-01,21.6,54
CSV

df queue --feed csv-demo --file sensor-demo.csv
```

Each CSV row becomes one record. CSV values remain strings; `"21.4"` and JSON number `21.4` are different data. Use `export-batch` after publication rather than manually recreating the verification array.

### Read JSON lines from a file or pipe

A `.jsonl` file contains one JSON record on each line:

```bash
printf '%s\n' \
  '{"test":true,"station":"HAM-02","signal_dbm":-51.2}' \
  '{"test":true,"station":"HAM-02","signal_dbm":-50.8}' |
  df queue --feed receiver-demo --stdin
```

A finite receiver log can be supplied with `--file receiver.jsonl`. An endless stream is not a continuously committed ingestion service: this implementation commits a queue import when that invocation finishes reading its input. Use finite chunks.

You can represent laboratory results, weather readings, equipment events or other observations as JSON records. This tool does not directly upload arbitrary PDFs, photographs or binary files. To commit to a binary file, calculate its hash separately, put that hash and relevant information in a JSON record, and retain the binary file separately.

## 6. Single submission versus batch submission

| Choice | Command options | What one successful publish does |
|---|---|---|
| One observation | `--batch-size 1` | Selects one queued record for one transaction |
| Up to 100 observations | `--batch-size 100 --mode hash` | Selects up to 100 queued records for one commitment transaction |
| Tiny on-chain batch | `--batch-size 2 --mode inline` | Publishes two records only if the whole envelope fits 254 bytes |

The default mode is `hash`; use `--mode inline` explicitly when you want the observations retrievable from chain data. A batch may select fewer records if fewer are queued. The maximum selection is 10000 records. Every invocation publishes at most one batch; it does not drain the queue automatically.

To publish another batch, first check the previous submission’s outcome and your remaining queue. Re-running `queue` adds data again; it is not a retry command.

## 7. Find records when you know the TXID

The easiest method:

```bash
df get "$HAM_TXID"
```

You can also ask Core directly:

```bash
"$TRUCLI" raw getdatafeedrecord "{\"txid\":\"$HAM_TXID\"}"
```

Core returns the raw feed payload and current transaction-location evidence. The Python tool presents the decoded observations and performs the batch-hash comparison.

A hash-mode reader still needs the separately retained batch array. No lookup command can recover missing data from its hash.

## 8. Discover records by feed label

First scan the block range containing the publication. Scanning means “read these blocks and build a local searchable list.” It does not spend funds.

After your HAM transaction is confirmed, obtain its exact height:

```bash
export HAM_HEIGHT="$(
  "$TRUCLI" raw gettransaction "{\"txid\":\"$HAM_TXID\"}" |
  python3 -c '
import json, sys
r = json.load(sys.stdin)
r = r.get("result", r)
if r.get("txState") != "CONFIRMED" or r.get("active") is not True:
    raise SystemExit("Not actively confirmed yet")
print(r["blockheight"])
'
)"

echo "Confirmed height: $HAM_HEIGHT"
```

Only proceed if a numeric height was printed:

```bash
df scan --from-height "$HAM_HEIGHT" --to-height "$HAM_HEIGHT" --max-blocks 1 --rescan

df search --feed "$HAM_FEED" --live
```

For a larger range, explicitly choose the heights. TRU’s genesis height is 1. The default maximum range is 100 blocks per invocation; `--max-blocks` can be increased up to 5000. A scan indexes all feeds in the chosen range. `search --feed` filters those results.

Save results:

```bash
df search --feed "$HAM_FEED" --live --format jsonl --output "$HAM_FEED-search.jsonl"
df search --feed "$HAM_FEED" --live --format csv --output "$HAM_FEED-search.csv"
```

Search without `--live` labels rows `CACHED_NOT_RECHECKED`; it is not a current-confirmation claim. The scanner prunes detected stale chain branches. A subsequent reorg can change status, so recheck important results.

On another upgraded, synced node, perform terminal setup with that node’s paths, set the same feed label and numeric block height, then scan/search. That node does not need the publisher’s wallet or journal. For hash mode it does need the original batch array to verify the observations themselves.

On the first use of an old discovery database, the upgraded tool resets its rebuildable legacy discovery cache. Rescan the required ranges. It does not reset your publication journal.

## 9. Is publishing automatic?

**Not in this version.** Each publication requires an explicit `publish ... --broadcast` command.

```bash
df watch --feed "$HAM_FEED" --interval 300 --batch-size 100
```

This checks and previews the queue every five minutes. It does not sign, submit or spend. Ctrl+C stops it; `--once` performs one pass. Agent memory-checkpoint automation is separate and does not grant datafeed spending permission.

## 10. Troubleshooting

| Output | Meaning and next step |
|---|---|
| `wallet_publish_enabled: false` | Enable the setting in the running Core’s configuration and restart. |
| `Owner address must belong to the loaded wallet` | Check that you supplied a complete address, not a list number, and that its wallet is loaded. The tool may still mark the batch uncertain; do not blindly retry it. |
| `Payload ... exceeds` / `Payload is ... limit ...` | Preview is too large. Hash mode commits the already queued data without including all observations on-chain. |
| `OBSERVED (not confirmed)` | Submission returned a TXID. Wait for mining and use read-only verification. |
| `Verification incomplete or failed` | Inspect the report: it may be unconfirmed, orphaned, missing original hash-mode data, or mismatched. Do not republish to fix verification. |
| `Exact confirmed batch match required` | Reconciliation requires a matching payload and active confirmation. Wait or investigate the mismatch. |
| `SUBMISSION_UNCERTAIN` | Preserve the journal and inspect Core. This message currently covers definite RPC refusals as well as genuinely ambiguous outcomes. Do not reset rows or automatically re-broadcast. |
| `Output exists` | Use a new export filename. |

For a candidate TXID, inspect without changing journal status:

```bash
df reconcile --feed YOUR_FEED_LABEL --txid YOUR_CANDIDATE_TXID
```

For a new-format journal batch, after review and confirmation, add its `--batch-id` to associate the exact matching commitment. Older uncertain submissions may lack enough stored batch information for this association. Do not copy a recovery SQL script intended for someone else’s specific batch IDs.

## 11. What to keep and what verification proves

Keep the TXID, feed label, confirming block height, exported batch array for hash mode, and a backup of your publication journal. Back up the SQLite journal when no datafeed process is writing to it. Its default location is `~/.local/share/tru/datafeed-v1.sqlite3`.

Inline observations are public. Hash-mode observations remain in your files unless you share them separately; a hash is not encryption and is not a guarantee that predictable data cannot be guessed. Do not publish secrets or private credentials inline.

Verification checks that the retrieved or supplied records match the commitment and that your Core currently reports the transaction confirmed. It does not establish measurement accuracy, exclusive ownership of a feed label, or independent verification without trusting your node. A reported sensor timestamp is supplied by the publisher, not certified by the blockchain.

For the operator’s current workflow, a single inline HAM test has already been submitted, retrieved and successfully reconciled after confirmation on the running node. This does not replace separate second-node, hash-batch, recovery and reorg acceptance tests.
