---
layout: post
title: TigerBeetle as a file storage
tags: tigerbeetle, accounting, gl
---

*Could not keep it under the rug until April Fool’s Day*

> TigerBeetle is a reliable, fast, and highly available database for financial accounting. It tracks financial transactions **or anything else that can be expressed as double-entry bookkeeping**, providing three orders of magnitude more performance and **guaranteeing durability even in the face of network, machine, and storage faults.**

![Challenge accepted]({{ site.url }}/public/challengeaccepted.png)

Continuing my [if all you have is a hammer, everything looks like a nail](https://aivarsk.com/2025/12/06/tigerbeetle-without-olgp-database1/) journey, I wanted to store arbitrary binary blobs in TigerBeetle to protect them from storage faults. If I can do that, I can store anything.

The `id` field of my Accounts will contain the filename (16-byte limit). I will store the total file size in the `user_data_64` field and the filename length in the `user_data_32` field (to simplify decoding). And my Accounts will have this nice property that `credits_posted` will contain the actual number of bytes written. I can detect failed uploads and resume that (a future TODO).

```python

def create_a_file(filename, size):
    if len(filename) > 16:
        raise ValueError("Invalid filename, more than 16 bytes")
    account = tb.Account(
        id=int.from_bytes(filename.encode()),
        user_data_64=size,
        user_data_32=len(filename),
        ledger=FILE,
        code=FILE,
    )
    errors = client.create_accounts([account])
    if errors:
        raise ValueError(errors[0])
    return account

```

What makes me unhappy is that I have not found a good use for `user_data_128` on the Account record. Such a waste of resources!

I will store the actual bytes in Transfer `user_data_128`, `user_data_64`, and `user_data_32` fields. That gives a total of 28 bytes per Transfer, and the Transfer `amount` will contain the number of bytes used in the Transfer. Which will be 28 for all Transfers except the last one containing the remaining bytes.

```python

            transfers.append(
                tb.Transfer(
                    id=tb.id(),
                    debit_account_id=system_id,
                    credit_account_id=file_id,
                    amount=len(block),
                    user_data_128=int.from_bytes(block[:16]),
                    user_data_64=int.from_bytes(block[16:24]),
                    user_data_32=int.from_bytes(block[24:]),
                    ledger=FILE,
                    code=FILE,
                )
            )

```

Because TigerBeetle uses double-entry bookkeeping, I will transfer all bytes from a system file “.” (debit side) to the desired file (credit side). Which is extremely useful for audit purposes to verify that `debits_posted` on the system file Account is the same as `credits_posted` on all file Account records.

As for getting data out of TigerBeetle, I can retrieve all credit Transfers for the specific Account. They are always correctly ordered by the `timestamp` field as [guaranteed by the TigerBeetle.](https://docs.tigerbeetle.com/coding/time/#timestamps-are-totally-ordered)

```python

        timestamp_min = 0

        while True:
            transfers = client.get_account_transfers(
                tb.AccountFilter(
                    account_id=file_id, flags=tb.AccountFilterFlags.CREDITS, limit=BULK, timestamp_min=timestamp_min
                )
            )
            for transfer in transfers:
                timestamp_min = transfer.timestamp
                ...

            if len(transfers) < BULK:
                break
            timestamp_min += 1
```

All that put together, I performed tests on some of the most valuable files I did not want to ever looose:

```bash

(venv)  du -b ~/Downloads/homework.mp4
104718755       /home/aivarsk/Downloads/homework.mp4
(venv)  time ./tbcp  ~/Downloads/homework.mp4 tb:backup.mp4

real    2m3.697s
user    1m4.408s
sys     0m1.568s

```

So, you can store your files durably at speeds close to 642 kB/s. Now, let’s retrieve the file and store it on the disk:

```bash

(venv)  time ./tbcp  tb:backup.mp4 copy.mp4

real    0m47.588s
user    0m27.027s
sys     0m0.553s

```

Downloading is around four times faster at 2,228 MB/s! And of course, I verified that not a single bit was lost during the round-trip:

```bash

(venv)  sha256sum ~/Downloads/homework.mp4
4ee75486c7c65a5c158f7f6b2ca6458195aa25b155b0688173b4b52583ce4cac  /home/aivarsk/Downloads/homework.mp4
(venv)  sha256sum copy.mp4
4ee75486c7c65a5c158f7f6b2ca6458195aa25b155b0688173b4b52583ce4cac  copy.mp4
(venv) 
```

If you want to store your valuable files, guaranteeing durability even in the face of network, machine, and storage faults, [here is the full source code](https://gist.github.com/aivarsk/2b26854c956e36fdfd73349586f2b168)
