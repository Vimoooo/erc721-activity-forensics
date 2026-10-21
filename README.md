import os
import json
from collections import defaultdict
from web3 import Web3

RPC_URL = os.getenv("RPC_URL")
NFT_ADDRESS = os.getenv("NFT_ADDRESS")
BLOCK_RANGE = int(os.getenv("BLOCK_RANGE", "5000"))

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

def decode_address(topic):
    return Web3.to_checksum_address(
        "0x" + topic.hex()[-40:]
    )

def main():
    if not RPC_URL or not NFT_ADDRESS:
        raise ValueError(
            "Set RPC_URL and NFT_ADDRESS environment variables."
        )

    w3 = Web3(Web3.HTTPProvider(RPC_URL, timeout=30))

    if not w3.is_connected():
        raise ConnectionError("Unable to connect to RPC.")

    contract = Web3.to_checksum_address(NFT_ADDRESS)
    latest_block = w3.eth.block_number
    start_block = max(0, latest_block - BLOCK_RANGE)

    # Query in chunks to avoid RPC providers' log-range limits.
    chunk_size = 500
    activity = defaultdict(lambda: {
        "transfers": 0,
        "participants": set(),
        "last_block": 0,
        "last_tx": None,
    })

    print(f"Scanning NFT contract: {contract}")
    print(f"Block range: {start_block} - {latest_block}")

    for start in range(start_block, latest_block + 1, chunk_size):
        end = min(start + chunk_size - 1, latest_block)

        logs = w3.eth.get_logs({
            "address": contract,
            "fromBlock": start,
            "toBlock": end,
            "topics": [TRANSFER_TOPIC],
        })

        for log in logs:
            if len(log["topics"]) < 4:
                continue

            sender = decode_address(log["topics"][1])
            receiver = decode_address(log["topics"][2])
            token_id = int.from_bytes(
                log["topics"][3],
                byteorder="big"
            )

            item = activity[token_id]
            item["transfers"] += 1
            item["participants"].update([sender, receiver])

            if log["blockNumber"] >= item["last_block"]:
                item["last_block"] = log["blockNumber"]
                item["last_tx"] = log["transactionHash"].hex()

    report = {
        "contract": contract,
        "chain_id": w3.eth.chain_id,
        "scanned_from": start_block,
        "scanned_to": latest_block,
        "tokens_seen": len(activity),
        "tokens": [],
    }

    for token_id, item in sorted(activity.items()):
        report["tokens"].append({
            "token_id": token_id,
            "transfer_count": item["transfers"],
            "unique_addresses": len(item["participants"]),
            "last_activity_block": item["last_block"],
            "last_transaction": item["last_tx"],
        })

    filename = "nft_activity_report.json"

    with open(filename, "w", encoding="utf-8") as file:
        json.dump(report, file, indent=2)

    print(f"\nNFTs observed: {len(activity)}")
    print(f"Report saved to: {filename}")

    for token in report["tokens"][:10]:
        print(
            f"Token #{token['token_id']} | "
            f"Transfers: {token['transfer_count']} | "
            f"Addresses: {token['unique_addresses']}"
        )

if __name__ == "__main__":
    main()
