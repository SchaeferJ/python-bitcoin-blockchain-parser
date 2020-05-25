# bitcoin-blockchain-parser (Multicore enabled)

This Python 3 library is a fork of the [bitcoin-blockchain-parser](https://github.com/alecalve/python-bitcoin-blockchain-parser)
which provides a parser for the raw data stored by bitcoind. In contrast to the original library, this fork allows 
processes to read the blockchain data in parallel. This is particularly useful if you want to parallelize your analysis 
across multiple CPU cores. 

**NOTE:** The database used by bitcoind (LevelDB) does not support multi-process concurrency. To "solve" that problem,
a temporary directory is created and symbolically linked to the data files for every process. Thus, each process can
acquire a lock for his own temporary directory and simultaneous reads become possible (see 
[this Stackoverflow-Answer](https://stackoverflow.com/a/23540423) for details). However, this is a terrible hack that
will introduce significant instability to your code and it should **NEVER** be used in any productive environments.
In particular, any attempt to perform parallel writes will result in unexpected and unpredictable behaviour and most 
likely cause data corruption.

**This library will only work on Linux!** 

## Features
- Detects outputs types
- Detects addresses in outputs
- Interprets scripts
- Supports SegWit
- Supports ordered block parsing
- Support parallel block parsing

## Examples

Below are two basic examples for parsing the blockchain. More examples are available in the examples directory.

### Unordered Blocks

This blockchain parser parses raw blocks saved in Bitcoin Core's `.blk` file format. Bitcoin Core does not guarantee that these blocks are saved in order. If your application does not require that blocks are parsed in order, the `Blockchain.get_unordered_blocks(...)` method can be used:

```python
import os 
from blockchain_parser.blockchain import Blockchain

# Instantiate the Blockchain by giving the path to the directory 
# containing the .blk files created by bitcoind
blockchain = Blockchain(os.path.expanduser('~/.bitcoin/blocks'))
for block in blockchain.get_unordered_blocks():
    for tx in block.transactions:
        for no, output in enumerate(tx.outputs):
            print("tx=%s outputno=%d type=%s value=%s" % (tx.hash, no, output.type, output.value))
```

### Ordered Blocks

If maintaining block order is necessary for your application, you should use the `Blockchain.get_ordered_blocks(...)` method. This method uses Bitcoin Core's LevelDB index to locate ordered block data in it's `.blk` files.

```python
import os 
from blockchain_parser.blockchain import Blockchain

# To get the blocks ordered by height, you need to provide the path of the
# `index` directory (LevelDB index) being maintained by bitcoind. It contains
# .ldb files and is present inside the `blocks` directory.
for block in blockchain.get_ordered_blocks(os.path.expanduser('~/.bitcoin/blocks/index'), end=1000):
    print("height=%d block=%s" % (block.height, block.hash))
```

Blocks can be iterated in reverse by specifying a start parameter that is greater than the end parameter.

```python
for block in blockchain.get_ordered_blocks(os.path.expanduser('~/.bitcoin/blocks/index'), start=510000, end=0):
    print("height=%d block=%s" % (block.height, block.hash))
```

Building the LevelDB index can take a while which can make iterative development and debugging challenging. For this reason, `Blockchain.get_ordered_blocks(...)` supports caching the LevelDB index database using [pickle](https://docs.python.org/3.6/library/pickle.html). To use a cache simply pass `cache=filename` to the ordered blocks method. If the cached file does not exist it will be created for faster parsing the next time the method is run. If the cached file already exists it will be used instead of re-parsing the LevelDB database. 

```python
for block in blockchain.get_ordered_blocks(os.path.expanduser('~/.bitcoin/blocks/index'), cache='index-cache.pickle'):
    print("height=%d block=%s" % (block.height, block.hash))
```

**NOTE**: You must manually/programmatically delete the cache file in order to rebuild the cache. Don't forget to do this each time you would like to re-parse the blockchain with a higher block height than the first time you saved the cache file as the new blocks will not be included in the cache.

## Installing

Requirements : python-bitcoinlib, plyvel, coverage for tests

plyvel requires leveldb development libraries for LevelDB >1.2.X

On Linux, install libleveldb-dev

```
sudo apt-get install libleveldb-dev
```

Then, just run
```
python setup.py install
```

## Tests

Run the test suite by lauching
```
./tests.sh
```



