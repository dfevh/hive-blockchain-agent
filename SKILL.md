---
name: hive-blockchain-agent
description: >
  Use this skill whenever an agent needs to interact with the HIVE blockchain
  ecosystem. Covers posting content, voting and curation, HBD savings management,
  Resource Credit (RC) monitoring and delegation, HP power-up/power-down, token
  transfers, custom_json operations, account queries, streaming live blockchain
  data, and Lightning Network bridging via v4vapp. Trigger on any mention of
  HIVE, HBD, Hive Power, RC credits, Peakd, Ecency, Keychain, v4vapp, or
  any HIVE-native dApp interaction.
version: 1.0.0
author: dfevh
license: MIT
tags:
  - hive
  - blockchain
  - web3
  - social
  - defi
  - hbd
  - content
  - curation
  - agent
compatible_skills:
  - alby-bitcoin-payments-agent-skill
  - crypto-ta-analyzer
  - trading-strategist
  - analyst
  - writer
  - creditor
  - sniper
  - hunter
---

# HIVE Blockchain Agent Skill

## What HIVE Is — Agent Context

HIVE is a Delegated Proof-of-Stake (DPoS) blockchain purpose-built for
social content and Web3 applications. Key facts every agent must know:

- **Block time**: 3 seconds. Transactions confirm fast — no gas fees.
- **Resource Credits (RC)**: Every account has an RC pool that replenishes
  over 5 days. RC is proportional to Hive Power (HP = staked HIVE). Agents
  spend RC for every on-chain action (post, vote, transfer, custom_json).
  Never let RC drop to zero — monitor it before every action batch.
- **HBD (Hive Backed Dollar)**: HIVE's algorithmic stablecoin. HBD savings
  earn a protocol-set APR (currently ~15–20%). Putting HBD into savings is
  the most risk-free yield action available on the chain.
- **Hive Power (HP)**: Staked HIVE. More HP = more RC budget = more
  on-chain actions per day. Also governs vote weight — higher HP = more
  valuable curation votes.
- **Reward Pool**: Every 7 days, posts pay out from a shared community
  reward pool. Votes within the first 5 minutes of a post earn maximum
  curation rewards for the voter. This is the Sniper window.
- **Keychain / HiveSigner / HiveAuth**: Authentication layers. Agents should
  use posting keys for social operations (post, vote, follow, custom_json)
  and active keys only for financial operations (transfer, power up/down,
  savings). NEVER expose or log private keys.

---

## API Nodes — Always Use These

```
Primary:   https://api.hive.blog
Backup 1:  https://api.deathwing.me
Backup 2:  https://hived.emre.sh
Backup 3:  https://api.openhive.network
```

If the primary node fails or returns an error, rotate to the next. All
nodes accept JSON-RPC 2.0 over HTTPS POST.

---

## Core Libraries

### JavaScript / TypeScript
```bash
npm install @hiveio/dhive
# or
npm install hive-tx
```

### Python
```bash
pip install beem
# beem is the most complete Python library for HIVE
```

---

## Operation 1 — Read Account Info

```javascript
// JS: Get full account data including HP, RC, HBD balances
import { Client } from '@hiveio/dhive';
const client = new Client(['https://api.hive.blog', 'https://api.deathwing.me']);

async function getAccount(username) {
  const [account] = await client.database.getAccounts([username]);
  return {
    name: account.name,
    hive_balance: account.balance,           // liquid HIVE
    hbd_balance: account.hbd_balance,        // liquid HBD
    hbd_savings: account.savings_hbd_balance,// HBD in savings (earning APR)
    hive_power: account.vesting_shares,      // raw VESTS (convert to HP below)
    reputation: account.reputation,
    post_count: account.post_count,
    voting_manabar: account.voting_manabar,  // current vote power (max 10000)
  };
}
```

```python
# Python: Get account with beem
from beem import Hive
from beem.account import Account

hive = Hive(node="https://api.hive.blog")
account = Account("username", blockchain_instance=hive)

print(account.get_balances())       # all token balances
print(account.get_resource_credits()) # RC status
print(account.rep)                  # reputation score
```

---

## Operation 2 — Check & Monitor Resource Credits

**Check RC before every batch of operations.** If RC is below 20%, pause
and wait for recharge or delegate from a higher-HP account.

```python
from beem import Hive
from beem.rc import RC

hive = Hive()
rc = RC(hive_instance=hive)

# Current RC costs per operation type
costs = {
  "custom_json": rc.custom_json(),
  "vote":        rc.vote(),
  "comment":     rc.comment(),   # same cost as a new post
}

# Check account's current RC
from beem.account import Account
account = Account("your_username", blockchain_instance=hive)
rc_info = account.get_resource_credits()

current_rc    = rc_info["rc_manabar"]["current_mana"]
max_rc        = rc_info["max_rc"]
rc_percentage = (current_rc / max_rc) * 100

print(f"RC: {rc_percentage:.1f}% remaining")
print(f"Can vote: {int(current_rc / costs['vote'])} more times")
print(f"Can post: {int(current_rc / costs['comment'])} more times")
```

**Rule**: If RC < 20%, do NOT post or vote. Only read. Wait for recharge
or request RC delegation from a higher-HP account.

---

## Operation 3 — Post Content (Writer Skill)

```python
from beem import Hive
from beem.account import Account
import json, time

def post_to_hive(
    author,        # HIVE username string
    title,         # Post title string
    body,          # Markdown body string
    tags,          # list of strings, first tag = category
    posting_key,   # WIF posting private key — keep in env var, never hardcode
    beneficiaries=None  # optional: [{"account": "username", "weight": 1000}] (10%)
):
    hive = Hive(keys=[posting_key])
    permlink = title.lower().replace(" ", "-")[:255]  # max 255 chars
    permlink = ''.join(c for c in permlink if c.isalnum() or c == '-')
    
    # Deduplicate permlink with timestamp if needed
    permlink = f"{permlink}-{int(time.time())}"
    
    metadata = json.dumps({
        "tags": tags,
        "app": "maikers-hive-agent/1.0",
        "format": "markdown"
    })
    
    post_args = {
        "author": author,
        "permlink": permlink,
        "title": title,
        "body": body,
        "json_metadata": metadata,
        "parent_permlink": tags[0],  # community/category
        "parent_author": "",         # empty = top-level post
    }
    
    if beneficiaries:
        # Set beneficiaries (revenue sharing via Creditor skill)
        from beem.comment import Comment
        from beem.transactionbuilder import TransactionBuilder
        from beembase import operations
        # Use comment_options to add beneficiaries
        pass  # see beneficiary section below
    
    account = Account(author, blockchain_instance=hive)
    account.post(**post_args)
    return permlink
```

**Critical post rules:**
- Wait at least 5 minutes between posts to avoid spam flags
- Tags: max 8 tags, first tag = main community (e.g. "hive-167922")
- Body must be valid Markdown — images as `![alt](url)` format
- Permlink must be unique per author — add timestamp suffix if unsure
- App tag in json_metadata identifies your agent to the community

---

## Operation 4 — Vote / Curate (Sniper Skill)

```python
from beem import Hive
from beem.comment import Comment

def vote_post(
    voter,        # your HIVE username
    author,       # post author username  
    permlink,     # post permlink
    weight,       # vote weight: 1–10000 (10000 = 100% upvote)
    posting_key
):
    """
    SNIPER TIMING: Vote between 5 min and 24 hours after post creation
    for optimal curation reward. Votes in the first 5 minutes return
    curation rewards to the author, not the voter.
    
    Vote power (mana) regenerates 20% per day.
    At 100% weight, you drain 2% of mana per vote.
    At 10% weight (1000), you drain 0.2% per vote — use for micro-curation.
    
    Recommended strategy:
    - Full votes (10000): max 5 per day to preserve mana above 80%
    - Micro votes (500-1000): for engagement without draining mana
    """
    hive = Hive(keys=[posting_key])
    post = Comment(f"@{author}/{permlink}", blockchain_instance=hive)
    post.vote(weight=weight, voter=voter)
```

**Check post age before voting:**
```python
from datetime import datetime, timezone

def get_post_age_minutes(author, permlink):
    hive = Hive()
    post = Comment(f"@{author}/{permlink}", blockchain_instance=hive)
    created = post.json()["created"].replace("Z", "+00:00")
    created_dt = datetime.fromisoformat(created)
    now = datetime.now(timezone.utc)
    return (now - created_dt).total_seconds() / 60

# Only vote if 5 < age < 6.5 days (before payout)
age = get_post_age_minutes(author, permlink)
if 5 < age < 9360:  # 9360 minutes = 6.5 days
    vote_post(voter, author, permlink, 10000, key)
```

---

## Operation 5 — HBD Savings (Creditor Skill)

```python
from beem import Hive
from beem.account import Account

def deposit_hbd_to_savings(username, amount_hbd, active_key):
    """
    Move HBD into savings to earn protocol APR (~15-20%).
    Requires ACTIVE key (financial operation).
    Withdrawal from savings has a 3-day waiting period.
    This is the safest yield action on HIVE — no smart contract risk.
    """
    hive = Hive(keys=[active_key])
    account = Account(username, blockchain_instance=hive)
    account.transfer_to_savings(
        amount=f"{float(amount_hbd):.3f} HBD",
        to=username,
        memo="maikers-agent-savings-deposit"
    )

def withdraw_hbd_from_savings(username, amount_hbd, active_key):
    """Initiates withdrawal — funds available after 3 days."""
    hive = Hive(keys=[active_key])
    account = Account(username, blockchain_instance=hive)
    account.transfer_from_savings(
        amount=f"{float(amount_hbd):.3f} HBD",
        to=username,
        request_id=int(time.time()),  # unique ID per withdrawal request
        memo="maikers-agent-savings-withdrawal"
    )
```

---

## Operation 6 — Power Up HIVE (HP Growth)

```python
def power_up(username, amount_hive, active_key):
    """
    Convert liquid HIVE → Hive Power (staked).
    More HP = more RC (more agent actions per day) + stronger votes.
    Irreversible immediately — power down takes 13 weeks to complete.
    """
    hive = Hive(keys=[active_key])
    account = Account(username, blockchain_instance=hive)
    account.transfer_to_vesting(
        amount=f"{float(amount_hive):.3f} HIVE",
        to=username
    )
```

---

## Operation 7 — RC Delegation (Creditor Skill)

```python
def delegate_rc(from_account, to_account, rc_amount, active_key):
    """
    Delegate Resource Credits to another account (HF26+).
    Useful for: onboarding new agents that don't yet have enough HP.
    rc_amount is in raw RC units (very large numbers — use 10**9 scale).
    Example: 10_000_000_000 RC ≈ enough for ~60 votes/day for a new account.
    """
    from beem.transactionbuilder import TransactionBuilder
    from beembase import operations
    
    hive = Hive(keys=[active_key])
    op = operations.Delegate_rc({
        "from": from_account,
        "delegatees": [to_account],
        "max_rc": rc_amount
    })
    tb = TransactionBuilder(blockchain_instance=hive)
    tb.appendOps([op])
    tb.appendWif(active_key)
    tb.sign()
    tb.broadcast()
```

---

## Operation 8 — Custom JSON (App State On-Chain)

```python
def broadcast_custom_json(account, app_id, json_data, posting_key):
    """
    Write arbitrary JSON to the HIVE blockchain permanently.
    Use this for: agent action logs, skill execution records,
    on-chain coordination between Maikers agents, dApp state.
    Max 8KB per operation. Costs RC (much cheaper than a post).
    
    app_id examples:
      "maikers_agent_v1"   — agent action log
      "maikers_skill_run"  — skill execution record
      "hive_hunt_result"   — Hunter skill discovery log
    """
    from beem import Hive
    from beem.account import Account
    import json
    
    hive = Hive(keys=[posting_key])
    account_obj = Account(account, blockchain_instance=hive)
    account_obj.custom_json(
        id=app_id,
        json_data=json_data,
        required_posting_auths=[account]
    )
```

---

## Operation 9 — Stream Live Blockchain Data

```python
from beem import Hive
from beem.blockchain import Blockchain

def stream_hive_operations(op_types=None, account_filter=None):
    """
    Stream real-time HIVE operations as they happen.
    op_types: list of operation types to filter
    ["vote", "comment", "transfer", "custom_json", "claim_reward_balance"]
    
    Use cases:
    - Hunter skill: watch for new posts by target accounts
    - Sniper skill: detect posts that just passed the 5-min window
    - Analyst skill: track vote patterns and trending content
    - Creditor skill: monitor incoming HBD transfers
    """
    hive = Hive(node="https://api.hive.blog")
    blockchain = Blockchain(blockchain_instance=hive, mode="head")
    
    for operation in blockchain.stream(opNames=op_types):
        # Filter by account if specified
        if account_filter:
            involved = operation.get("author", "") or operation.get("voter", "") or \
                       operation.get("from", "") or operation.get("to", "")
            if involved not in account_filter:
                continue
        
        yield operation  # process in your agent loop
```

---

## Operation 10 — Lightning Bridge via v4vapp

```
Every HIVE account has a Lightning address: username@v4v.app

To receive BTC sats from HIVE earnings:
1. Send HBD to the v4vapp conversion gateway: @v4vapp
2. Memo: your Lightning address or invoice
3. v4vapp converts HBD → BTC and pays out via Lightning

To receive HIVE from Lightning payments:
1. Generate a v4vapp invoice via https://v4v.app
2. Pay the Lightning invoice from any NWC wallet (Alby Hub etc.)
3. HIVE/HBD arrives in your HIVE account

Combined with alby-bitcoin-payments-agent-skill, this creates a
fully autonomous earn-on-HIVE → receive-in-BTC loop.
```

---

## Maikers Agent Mapping

| Maikers Skill NFT | HIVE Operations to Run |
|---|---|
| **Writer** | Post content, edit posts, comment replies, publish schedules |
| **Analyst** | Stream blockchain data, parse reward curves, trending analysis |
| **Creditor** | HBD savings deposit/withdraw, RC delegation, token transfers |
| **Sniper** | Vote at 5-min window, monitor post age, execute timed actions |
| **Hunter** | Stream new posts, filter by tag/author/community, surface opportunities |
| **Entertainer** | Cross-post content, generate engagement threads, community replies |
| **Automaton** | Scheduled job runner for all above, RC budget scheduler |
| **Validator** | Verify transaction broadcasts, confirm RC sufficiency before ops |

---

## Agent Safety Rules

1. **NEVER store private keys in code.** Use environment variables or
   a secrets manager. Posting key is enough for social ops.
2. **NEVER use active key unless required.** Only for transfers, power up,
   savings, and RC delegation.
3. **Always check RC before a batch.** A failed transaction still costs RC.
4. **Rate-limit posts.** Max 4–5 posts per day per account. HIVE community
   flags spam aggressively.
5. **Always set a unique permlink.** Duplicate permlinks silently overwrite
   the existing post — dangerous for scheduled agents.
6. **Monitor your voting mana.** Keep above 80% for effective curation.
   Below 20%, your votes have negligible effect.
7. **Test on testnet first.** Use https://testnet.openhive.network for
   dry runs before mainnet deployment.

---

## Quick Reference: API Endpoints

```
Account info:       condenser_api.get_accounts
RC status:          rc_api.find_rc_accounts
Trending posts:     condenser_api.get_discussions_by_trending
New posts:          condenser_api.get_discussions_by_created
Post content:       condenser_api.get_content
Account history:    condenser_api.get_account_history
Dynamic globals:    condenser_api.get_dynamic_global_properties
HBD savings APR:    condenser_api.get_dynamic_global_properties → hbd_interest_rate
Block stream:       condenser_api.get_block (poll) or use beem Blockchain.stream
```

---

## Example: Full Autonomous Agent Loop (NFT #1906 — Writer + Analyst + Creditor)

```python
"""
This loop represents what NFT #1906 does when fully activated:
1. Analyst reads trending HIVE data
2. Writer drafts and posts content
3. Creditor manages the HBD rewards that come in 7 days later
"""
import os, time
from beem import Hive
from beem.account import Account
from beem.rc import RC
from beem.blockchain import Blockchain

ACCOUNT  = os.environ["HIVE_ACCOUNT"]
POST_KEY = os.environ["HIVE_POSTING_KEY"]
ACT_KEY  = os.environ["HIVE_ACTIVE_KEY"]

hive = Hive(keys=[POST_KEY, ACT_KEY])
account = Account(ACCOUNT, blockchain_instance=hive)
rc = RC(hive_instance=hive)

# Step 1: Analyst — check RC budget
rc_info = account.get_resource_credits()
rc_pct = rc_info["rc_manabar"]["current_mana"] / rc_info["max_rc"] * 100
if rc_pct < 20:
    print(f"RC at {rc_pct:.1f}% — standing by for recharge")
    time.sleep(3600)  # wait 1 hour

# Step 2: Analyst — get trending tags + content opportunities
trending = hive.rpc.get_trending_tags("", 20)
top_tags = [t["name"] for t in trending[:5]]

# Step 3: Writer — compose and post (content generated by AI agent)
# [AI generates content here based on trending context]
content = generate_hive_post(top_tags)  # your AI generation function

permlink = post_to_hive(
    author=ACCOUNT,
    title=content["title"],
    body=content["body"],
    tags=content["tags"],
    posting_key=POST_KEY
)

# Step 4: Creditor — log the action on-chain via custom_json
broadcast_custom_json(
    account=ACCOUNT,
    app_id="maikers_agent_v1",
    json_data={"action": "post", "permlink": permlink, "cycle": int(time.time())},
    posting_key=POST_KEY
)

# Step 5: Creditor — check HBD balance and auto-deposit excess to savings
balances = account.get_balances()
liquid_hbd = float(balances["HBD"])
if liquid_hbd > 10.0:
    keep = 5.0  # keep 5 HBD liquid
    deposit_hbd_to_savings(ACCOUNT, liquid_hbd - keep, ACT_KEY)
    print(f"Deposited {liquid_hbd - keep:.3f} HBD to savings")

print("Agent cycle complete.")
```

---

## Resources

- HIVE Developer Docs: https://developers.hive.io
- beem Python library: https://github.com/holgern/beem
- dhive JS library: https://github.com/openhive-network/dhive
- HIVE API explorer: https://hive.hivesigner.com
- v4vapp Lightning bridge: https://v4v.app
- RC system deep dive: https://developers.hive.io/tutorials-python/rcdemo.html
- HiveAuth (keyless auth): https://hiveauth.com
- Testnet: https://testnet.openhive.network
- Maikers Skills Registry: https://skills.maikers.com
