Lab A1: Transaction History Exploration
===

Introduction
---

Etherscan (https://etherscan.io/) provides a web service to explore Ethereum transactions and blocks. In this lab, you will retrieve and analyze Ethereum transaction history on etherscan by interacting with this website. Particularly, you will extract insights on transaction fees.

| Exercises | Points | CS student |
| --- | --- | --- | 
|  1  | 10 |  Required | 
|  2  | 10 | Required |
|  3  | 10 | Required | 
|  4  | 20 | Required | 
|  5  | 30 | Required | 
|  6  | 20 | Required | 

Exercise 1. Manually explore three transactions
---

Suppose the following Etherscan page shows details of a particular transaction (hash 0x84ae):

https://etherscan.io/tx/0x84aee3793659afeebfb89b86e6a8ffd3b9f143b3719c9b358905a83dbd71cb79

You are asked to report the average fees of three transactions, that is, transaction 0x84ae, its predecessor transaction and successor transaction. Transaction tx1 is the predecessor of tx2, if tx1 is ordered right before tx2 in the same block.

Hint: You can find ordered transaction history related to block `15479087` on the following web page: https://etherscan.io/txs?block=15479087

Exercise 2. Manually explore one block
---

Find the transaction that transfers the highest Ether "value" in block `15479087`. Report the transaction hash. 

Exercise 3. Manually explore two blocks
---

Find the last transaction in block `15479087` and the first transaction in block `15479088`. Report the average fees of these two transactions.

- Hint: Assume the first transaction in a block is listed as the first row on the first page under that block on etherscan.io. Likewise, the last transaction in a block is listed as the last row on the last page under that block on etherscan.io.

Exercise 4. Automatically explore 50 transactions in one block
---

```python
import requests
from time import sleep
from bs4 import BeautifulSoup
import re

# Regexes for ids
ADDR40 = re.compile(r'0x[a-fA-F0-9]{40}')
HASH64 = re.compile(r'0x[a-fA-F0-9]{64}')

def first_href(tr, startswith):
    """Return (anchor, href) of the first <a> whose href startswith(...) in this row."""
    for a in tr.find_all('a', href=True):
        href = a['href']
        if href.startswith(startswith):
            return a, href
    return None, None

def all_hrefs_after(tr, marker_a, startswith):
    """Yield (anchor, href) for anchors AFTER marker_a whose href startswith(...)."""
    found_marker = False
    for a in tr.find_all('a', href=True):
        if a is marker_a:
            found_marker = True
            continue
        if not found_marker:
            continue
        if a['href'].startswith(startswith):
            yield a, a['href']

def text_or_none(el):
    return el.get_text(strip=True) if el else None

def clean_number(el):
    """Join text nodes without inserting spaces (fixes '0 . 0001' -> '0.0001')."""
    if not el:
        return None
    return "".join(el.stripped_strings).replace(" ", "")

def extract_transaction_info(tr):
    # Tx hash from /tx/ link
    tx_a, tx_href = first_href(tr, '/tx/')
    tx_hash = None
    if tx_href:
        m = HASH64.search(tx_href)
        tx_hash = m.group(0) if m else None

    # Block number from /block/ link text
    blk_a, _ = first_href(tr, '/block/')
    block = blk_a.get_text(strip=True) if blk_a else None

    # From/To: first two /address/ links after the block link
    from_addr = to_addr = None
    if blk_a:
        addrs = []
        for a, href in all_hrefs_after(tr, blk_a, '/address/'):
            m = ADDR40.search(href)
            if m:
                addrs.append(m.group(0))
            if len(addrs) == 2:
                break
        if addrs:
            from_addr = addrs[0]
            if len(addrs) > 1:
                to_addr = addrs[1]

    # Value: Etherscan usually tags it with .td_showAmount (includes " ETH")
    value = text_or_none(tr.select_one('.td_showAmount'))

    # Clean fee/gas strings (remove spaces introduced by nested spans)
    fee = clean_number(tr.select_one('.showTxnFee'))          # number string (ETH, no unit)
    gas_price = clean_number(tr.select_one('.showGasPrice'))  # number string (Gwei, no unit)

    # Always return a dict (option 2 only; never skip rows)
    return {
        'hash': tx_hash,
        'block': block,
        'from': from_addr,
        'to': to_addr,
        'value': value,       # e.g., "0.124942 ETH"
        'fee': fee,           # e.g., "0.00019191" (ETH)
        'gas_price': gas_price,  # e.g., "9.13902086" (Gwei)
    }

def scrape_block(blocknumber, page):
    url = f"https://etherscan.io/txs?block={blocknumber}&p={page}"
    headers = {
        'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15; rv:120.0) Gecko/20100101 Firefox/120.0',
        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
        'Accept-Language': 'en-US,en;q=0.8',
    }
    sleep(0.5)
    resp = requests.get(url, headers=headers, timeout=20)
    resp.raise_for_status()

    soup = BeautifulSoup(resp.content, 'html.parser')
    rows = soup.select('table.table-hover tbody tr')
    if not rows:
        snippet = soup.get_text(" ", strip=True)[:200]
        print("[warn] No rows parsed; anti-bot or layout change? Snippet:", snippet)

    for row in rows:
        tx = extract_transaction_info(row)
        print(
            "transaction of ID:", tx.get('hash'),
            "block:", tx.get('block'),
            "from address", tx.get('from'),
            "toaddress", tx.get('to'),
            "transaction fee", tx.get('fee'),
            "gas price", tx.get('gas_price'),
            "value", tx.get('value')
        )

if __name__ == "__main__":
    scrape_block(15479087, 1)

```

In this exercise, you will run a python code to crawl data from the etherscan website automatically. The example code above crawls the etherscan web page  (i.e., https://etherscan.io/txs?block=15479087) to read the first 50 transactions in block `15479087`.

To run the python code, you will need a Python runtime and some libraries. If your computer does not support Python (yet), you can find installation instructions on
https://www.python.org/downloads/ for both Windows and Mac machines. In addition, the Python libraries can be installed in a Python console: 

```bash
>>> pip3 install requests
>>> pip3 install beautifulsoup4
```

After installation, copy the above python code to a file and run the file in a python runtime (e.g., your favorite python IDE). After running the code, you can observe transaction attributes printed on the terminal or Python console.

Exercise 5. Automatically explore all transactions in one block
---

In this exercise, you are required to report the average fee of all transactions in block `15479087`. You can modify the given code.

- Hint: transactions in block `15479087` are listed on three pages.

Exercise 6. Automatically explore transactions across two blocks
---

In this exercise, you are required to report the average fees of 100 transactions, which are the first 50 transactions in block `15479087` and the first 50 transactions in block `15479088`. You can modify the given code.


Deliverable
---

1. Report the transaction fee required for each exercise.
2. For exercise 4, submit the screenshot that runs the crawler code on your computer.
    - If there are too many results that cannot fit into a single screen, you can randomly choose two screens and do two screenshots. 
3. For exercise 5/6/7, submit your modified Python file and the screenshot that runs the code on your computer. The Python programs need to be stored in plaintext format and in separate files from your report. 

FAQ
---

- Question: How to verify your code is correct?
    - Answer: Let's say your modified Python code needs to scan 100 transactions and calculate the average transaction fee. To verify your code is correct, you can change the number 100 in your program to a smaller one, say 3, and manually calculate the average fee of the 3 transactions. If the manual calculation result equals your program result, it shows that your program is likely to be correct.
- Question: Can I do lab exercises 4/5/6 without installing anything on my computer?
    - Answer: Yes, it is possible. You could use Google's colab platform that supports running python code in a web browser:  https://colab.research.google.com/?utm_source=scs-index .
- Question: How to install a Python IDE?
    - Answer: It is not required to install a Python IDE (Python runtime is enough). But if you want, you can install the Pycharm for Python IDE (the community version) by following the instruction here: https://www.jetbrains.com/help/pycharm/installation-guide.html#toolbox. You will need to configure Python interpreter in Pycharm: https://www.jetbrains.com/help/pycharm/configuring-local-python-interpreters.html.

