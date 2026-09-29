"""
agent_transfer.py — Agent-to-Agent USDC Transfer (Python CLI)

Mirrors the TypeScript backend in a single runnable script.
Uses the Circle Developer-Controlled Wallets REST API directly
(no Python SDK needed — just httpx + standard library).

Usage:
    pip install httpx python-dotenv
    cp .env.example .env        # fill in your keys
    python agent_transfer.py provision           # create Agent Alpha + Beta
    python agent_transfer.py balances            # show live USDC balances
    python agent_transfer.py transfer 0.10       # send $0.10 USDC Alpha → Beta
    python agent_transfer.py history             # print transfer history
    python agent_transfer.py status <tx_id>      # check one transaction
"""

import json
import os
import sys
import time
import uuid
from pathlib import Path
from typing import Optional

try:
    import httpx
    from dotenv import load_dotenv
except ImportError:
    sys.exit(
        "Missing dependencies. Run:  pip install httpx python-dotenv"
    )

# ─── configuration ────────────────────────────────────────────────────────────

load_dotenv()

API_KEY = os.getenv("CIRCLE_DEVELOPER_CONTROLLED_API_KEY") or os.getenv("CIRCLE_API_KEY") or ""
ENTITY_SECRET = os.getenv("CIRCLE_ENTITY_SECRET") or os.getenv("ENTITY_SECRET") or ""
BASE_URL = "https://api.circle.com/v1/w3s"
BLOCKCHAIN = "ARC-TESTNET"
ARC_USDC = "0x3600000000000000000000000000000000000000"
ARC_EXPLORER = "https://explorer.testnet.arc.io/tx"

# Local state file so wallet IDs persist across runs
STATE_FILE = Path(__file__).parent / ".agent_state.json"
TERMINAL_STATES = {"COMPLETE", "FAILED", "DENIED", "CANCELLED"}


# ─── helpers ──────────────────────────────────────────────────────────────────

def _headers() -> dict:
    if not API_KEY:
        sys.exit("ERROR: Set CIRCLE_DEVELOPER_CONTROLLED_API_KEY in your .env file.")
    return {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    }


def _client() -> httpx.Client:
    return httpx.Client(base_url=BASE_URL, headers=_headers(), timeout=30)


def _load_state() -> dict:
    if STATE_FILE.exists():
        return json.loads(STATE_FILE.read_text())
    return {"walletSetId": None, "agents": [], "history": []}


def _save_state(state: dict) -> None:
    STATE_FILE.write_text(json.dumps(state, indent=2))


def _idempotency_key() -> str:
    return str(uuid.uuid4())


def _print_agent(agent: dict, balance: Optional[str] = None) -> None:
    role = "Sender" if agent["role"] == "sender" else "Receiver"
    bal = f"  USDC: ${balance}" if balance else ""
    print(f"  [{role}] {agent['name']}")
    print(f"          id:      {agent['id']}")
    print(f"          address: {agent['address']}{bal}")


# ─── API wrappers ─────────────────────────────────────────────────────────────

def create_wallet_set(client: httpx.Client) -> str:
    """Create a wallet set and return its ID."""
    resp = client.post("/developer/walletSets", json={"name": "Agent-to-Agent Set"})
    resp.raise_for_status()
    ws_id = resp.json()["data"]["walletSet"]["id"]
    print(f"  Wallet set created: {ws_id}")
    return ws_id


def create_wallets(client: httpx.Client, wallet_set_id: str) -> list[dict]:
    """Create 2 SCA wallets on Arc Testnet."""
    resp = client.post(
        "/developer/wallets",
        json={
            "idempotencyKey": _idempotency_key(),
            "accountType": "SCA",
            "blockchains": [BLOCKCHAIN],
            "count": 2,
            "walletSetId": wallet_set_id,
        },
    )
    resp.raise_for_status()
    return resp.json()["data"]["wallets"]


def get_usdc_balance(client: httpx.Client, wallet_id: str) -> str:
    """Return USDC balance string for a wallet (e.g. '1.50')."""
    resp = client.get(f"/wallets/{wallet_id}/balances")
    resp.raise_for_status()
    balances = resp.json()["data"].get("tokenBalances", [])
    for tb in balances:
        if tb.get("token", {}).get("symbol") == "USDC":
            return tb.get("amount", "0")
    return "0"


def create_transfer(
    client: httpx.Client,
    from_wallet_id: str,
    to_address: str,
    amount: str,
) -> str:
    """Initiate a USDC transfer and return the transaction ID."""
    resp = client.post(
        "/developer/transactions/transfer",
        json={
            "idempotencyKey": _idempotency_key(),
            "walletId": from_wallet_id,
            "tokenAddress": ARC_USDC,
            "destinationAddress": to_address,
            "amounts": [amount],
            "fee": {"type": "level", "config": {"feeLevel": "MEDIUM"}},
        },
    )
    resp.raise_for_status()
    tx_id = resp.json()["data"]["id"]
    return tx_id


def get_transaction(client: httpx.Client, tx_id: str) -> dict:
    """Fetch transaction state from Circle."""
    resp = client.get(f"/transactions/{tx_id}")
    resp.raise_for_status()
    return resp.json()["data"]["transaction"]


def poll_until_terminal(client: httpx.Client, tx_id: str, interval: int = 3) -> dict:
    """Poll getTransaction every `interval` seconds until a terminal state."""
    print(f"  Polling transaction {tx_id} …")
    while True:
        tx = get_transaction(client, tx_id)
        state = tx.get("state", "UNKNOWN")
        print(f"    state: {state}")
        if state in TERMINAL_STATES:
            return tx
        time.sleep(interval)


# ─── commands ─────────────────────────────────────────────────────────────────

def cmd_provision() -> None:
    """Create Agent Alpha (sender) and Agent Beta (receiver)."""
    state = _load_state()
    if len(state.get("agents", [])) >= 2:
        print("Agents already provisioned:")
        for a in state["agents"]:
            _print_agent(a)
        return

    if not ENTITY_SECRET:
        sys.exit(
            "ERROR: CIRCLE_ENTITY_SECRET is required to provision wallets.\n"
            "Set it in your .env file."
        )

    print("Provisioning agent wallets on Arc Testnet …")
    with _client() as c:
        ws_id = create_wallet_set(c)
        wallets = create_wallets(c, ws_id)

    if len(wallets) < 2:
        sys.exit("ERROR: Expected 2 wallets; Circle returned fewer.")

    state["walletSetId"] = ws_id
    state["agents"] = [
        {"id": wallets[0]["id"], "address": wallets[0]["address"], "name": "Agent Alpha", "role": "sender"},
        {"id": wallets[1]["id"], "address": wallets[1]["address"], "name": "Agent Beta", "role": "receiver"},
    ]
    _save_state(state)

    print("\nAgents created:")
    for a in state["agents"]:
        _print_agent(a)
    print(
        "\nNext: fund Agent Alpha with testnet USDC via https://faucet.circle.com, "
        "then run:  python agent_transfer.py transfer 0.10"
    )


def cmd_balances() -> None:
    """Show live USDC balances for both agents."""
    state = _load_state()
    agents = state.get("agents", [])
    if not agents:
        sys.exit("No agents provisioned yet. Run:  python agent_transfer.py provision")

    print("Live USDC balances on Arc Testnet:")
    with _client() as c:
        for a in agents:
            bal = get_usdc_balance(c, a["id"])
            _print_agent(a, bal)


def cmd_transfer(amount: str) -> None:
    """Send USDC from Agent Alpha to Agent Beta."""
    try:
        amt_f = float(amount)
        if amt_f <= 0:
            raise ValueError
    except ValueError:
        sys.exit(f"Invalid amount: {amount!r}. Provide a positive number, e.g. 0.10")

    state = _load_state()
    agents = state.get("agents", [])
    if len(agents) < 2:
        sys.exit("Agents not provisioned. Run:  python agent_transfer.py provision")

    sender = next((a for a in agents if a["role"] == "sender"), None)
    receiver = next((a for a in agents if a["role"] == "receiver"), None)
    if not sender or not receiver:
        sys.exit("Unexpected agent configuration in state file.")

    print(f"Sending ${amount} USDC: {sender['name']} → {receiver['name']}")
    with _client() as c:
        tx_id = create_transfer(c, sender["id"], receiver["address"], amount)
        print(f"  Transaction ID: {tx_id}")
        tx = poll_until_terminal(c, tx_id)

    state.setdefault("history", []).insert(
        0,
        {
            "id": tx_id,
            "fromAgent": sender["name"],
            "toAgent": receiver["name"],
            "amount": amount,
            "state": tx.get("state"),
            "txHash": tx.get("txHash"),
        },
    )
    _save_state(state)

    final_state = tx.get("state")
    tx_hash = tx.get("txHash")
    if final_state == "COMPLETE":
        print(f"\n  Transfer COMPLETE!")
        if tx_hash:
            print(f"  Explorer: {ARC_EXPLORER}/{tx_hash}")
    else:
        print(f"\n  Transfer ended with state: {final_state}")
        if tx.get("errorReason"):
            print(f"  Reason: {tx['errorReason']}")


def cmd_status(tx_id: str) -> None:
    """Check the current state of a transaction."""
    with _client() as c:
        tx = get_transaction(c, tx_id)
    state = tx.get("state", "UNKNOWN")
    tx_hash = tx.get("txHash")
    print(f"Transaction {tx_id}")
    print(f"  State:   {state}")
    if tx_hash:
        print(f"  Explorer: {ARC_EXPLORER}/{tx_hash}")
    if tx.get("errorReason"):
        print(f"  Error:   {tx['errorReason']}")


def cmd_history() -> None:
    """Print local transfer history."""
    state = _load_state()
    history = state.get("history", [])
    if not history:
        print("No transfers recorded yet.")
        return
    print(f"Transfer history ({len(history)} records):")
    for rec in history:
        tx_hash = rec.get("txHash", "")
        explorer = f" → {ARC_EXPLORER}/{tx_hash}" if tx_hash else ""
        print(
            f"  [{rec.get('state', '?'):10}] "
            f"{rec.get('fromAgent')} → {rec.get('toAgent')}  "
            f"${rec.get('amount')} USDC{explorer}"
        )


# ─── entry point ──────────────────────────────────────────────────────────────

COMMANDS = {
    "provision": (cmd_provision, 0),
    "balances":  (cmd_balances,  0),
    "transfer":  (cmd_transfer,  1),   # amount
    "status":    (cmd_status,    1),   # tx_id
    "history":   (cmd_history,   0),
}

USAGE = """\
Usage:
  python agent_transfer.py provision           create Agent Alpha + Beta
  python agent_transfer.py balances            show live USDC balances
  python agent_transfer.py transfer <amount>   send USDC Alpha → Beta
  python agent_transfer.py status <tx_id>      check one transaction
  python agent_transfer.py history             print transfer history
"""

if __name__ == "__main__":
    args = sys.argv[1:]
    if not args or args[0] not in COMMANDS:
        print(USAGE)
        sys.exit(0)

    cmd_name = args[0]
    fn, nargs = COMMANDS[cmd_name]
    if len(args) - 1 < nargs:
        print(f"ERROR: '{cmd_name}' requires {nargs} argument(s).\n")
        print(USAGE)
        sys.exit(1)

    fn(*args[1 : 1 + nargs])
