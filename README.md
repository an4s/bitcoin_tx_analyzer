# bitcoin_tx_analyzer
A collection of forks of different versions of the official Bitcoin repo that allow fast retrieval of data relevant to transactions.

## Motivation

Have you ever needed to look up fee or size of Bitcoin transactions? Different websites such as [Blockchain](https://www.blockchain.com/) and [coinbase](https://developers.coinbase.com/) provide APIs [[1](https://www.blockchain.com/api), [2](https://developers.coinbase.com/api/v2)] which can be used by developers to query such information. However, more often than not, these websites place a limit on the number of consecutive queries that can be made through the aforementioned APIs [[3](https://www.blockchain.com/api/q), [4](https://developers.coinbase.com/api/v2#rate-limiting)]. Therefore, it becomes very time consuming if you want to retrieve data for a large number of transactions.

If you run your own Bitcoin node with `txindex` [[5](https://bitcoin.stackexchange.com/a/35708)] enabled, you could query data relevant to transactions by invoking the Bitcoin RPC APIs [[6](https://developer.bitcoin.org/reference/rpc/)] through `bitcoin-cli`. Note, however, that such requests are relayed and responded over an HTTP connection with the RPC server. Not only can this be slow for a number of reasons, too many requests can cause the node to crash [7](https://github.com/bitcoin/bitcoin/blob/master/doc/JSON-RPC-interface.md#limitations). Hence, relying on this process can become cumbersome very quickly.

I ran into the issues mentioned above for one of my research projects where I needed to query data for more than _**4 million**_ transactions. I quickly realized that unless I had a whole lot of time on my hand (which I obviously didn't), I'd  have to come up with a different, much faster solution. I experimented with querying data directly from the Bitcoin Core software and to my delight, it turned out to be much faster than the methods described above. I basically inserted a new thread in the software that read transaction hashes from a file, queries relevant data from within the software and writes the results to files. So far, I have ported this functionality in different versions of Bitcoin Core that I have used in my research work over the last few years.

### Supported versions:

- [`v0.15.2`](https://github.com/an4s/bitcoin_tx_analyzer_v0.15.2)
- [`v0.18.1`](https://github.com/an4s/bitcoin_tx_analyzer_v0.18.1)
- [`v0.20.1`](https://github.com/an4s/bitcoin_tx_analyzer_v0.20.1)

### Requirements:

- Fully synchronized Bitcoin node
- `txindex` should be set to `1` when running the Bitcoin node
- `bitcoin-qt` or `bitcoind` must be run in the foreground on a terminal (i.e., `daemon` should not be enabled)

## Usage

The analysis tools works in the following fashion: it reads data from files and writes data to files. The tool must be enabled by passing the `-enable-tx-analysis` command line argument to `bitcoin-qt` or `bitcoind`, i.e.,

```> bitcoind -enable-tx-analysis```

or

```> bitcoin-qt -enable-tx-analysis```

By default, the tool reads from a file named `ta-input-file` which can be changed by providing the `-ta-input-filename` command line argument to `bitcoin-qt` or `bitcoind`, i.e.,

```> bitcoind -enable-tx-analysis -ta-input-filename=<path/to/file>```

or

```> bitcoin-qt -enable-tx-analysis -ta-input-filename=<path/to/file>```

Note that _all_ files must be placed in a directory named `tx-analysis-files` that should itself be created in the Bitcoin data directory. For example, in Linux, this could be

```/home/ubuntu/.bitcoin/tx-analysis-files/ta-input-file```

The input file contains a list of names of files that contain one or more transaction hashes. This is useful if you have several data sets that you want to process in a single pass.

**Example:**

```
   > less ta-input-file
     txids_ds_1
     txids_ds_2
```

```
   > less txids_ds_1
     aef0bec0979acb7ec7f15de7be67c6eb86c30a8a5666ef26a9fc95109af447e9
     2f4b94e03496c593134a5e697cb64611f96e479bd6b1fe0f2878962ae583f334
     4345e17ae0dac1656cc89c95241a1d855bcdb7ae59c04f5a8f9ecfa9e3335de4
     65412e6abfc448ea0f42207da74c72297bfbe8a17a5cad602414f703b2a3d6c4
     5e47b0bde0a57a40aa98a9af381184613e9ca4d6f6547d63ccef9d162846d2d7
```

```
   > less txids_ds_2
     0546cb2f471d0ec97ed70becb48a5efb929623f4bc4063d2257f4675394935c3
     be0f4813e321162dfe239a07e0c39e66c545f987638c3e62415b7a809ca27cce
     66da8d4b5a59761c39ed6c71a9fee687c0a2d44a85d04bdb6acd0d41d4601271
     29da6f9f071199cdbcfea56712106a6e39362172dff20ba653056d7c4400557c
     cf49bc40cf5adae09b962d29f1266e50d0d618a3d945894dd22f9da3951c6650
```

The tool queries three properties of each transaction
- Fee of transaction
- Size of transaction
- Parents of transaction

For each file `f` in the `ta-input-file`, the tool creates two files named `f_out` which contains the hash of a transaction, its fee (in satoshis) and its size (in bytes) separated by commas, and `f_unknown` which contains a list of transaction hashes that were not known to the Bitcoin node. In addition, it creates a directory named `f_parents` (if it does not already exist) which contains a file for _each_ transaction in the file `f` within which is a list of hashes of its parent transactions.

### Sample 1

The following run shows the working of the tool with the default arguments (input/output files in the `tx-analysis-files` directory in this repo). Transactions were processed at a rate between 30-50 transactions per second; processing may be stalled for a few seconds if the node is synchronizing with the network and utilizes higher CPU usage. The tool shows the progress of the analysis at runtime as the number of transactions that have been analyzed, this number as a fraction of the total number of transactions, and the total time that has been elapsed so far for the current data set. Note that once the tool finishes performing analysis, it simply terminates the thread created for this purpose. It does _**not**_ affect the Bitcoin software or cause it to shut down.

```
an4s@ubuntu:~$ bitcoin-qt -enable-tx-analysis
> TX analysis enabled
> Initializing TX analyzer
> INFO - file </home/an4s/.bitcoin/tx-analysis-files/ex-1> added to processing queue
> INFO - file </home/an4s/.bitcoin/tx-analysis-files/ex-2> added to processing queue
> TX analyzer initialized successfully
> Running TX analysis
>> INFO - Reading hashes from file: <ex-1>
>> INFO - Successfully read hashes from file: <ex-1>
>> INFO - Beginning transaction analysis...
Progress:	10000/10000 (100.0000%) [elapsed 00:04:42.689]
UKN: 2    <-- number of transactions unknown to the node
EXC: 0    <-- number of transactions that are invalid or could cause exceptions at runtime
>> INFO - writing transaction fees and sizes to file: <ex-1_out>
>> INFO - writing transaction parents to directory: <ex-1_parents>
>> INFO - writing hashes of unknown transactions to file: <ex-1_unknown>
>> INFO - Reading hashes from file: <ex-2>
>> INFO - Successfully read hashes from file: <ex-2>
>> INFO - Beginning transaction analysis...
Progress:	10000/10000 (100.0000%) [elapsed 00:03:09.282]
UKN: 1
EXC: 0
>> INFO - writing transaction fees and sizes to file: <ex-2_out>
>> INFO - writing transaction parents to directory: <ex-2_parents>
>> INFO - writing hashes of unknown transactions to file: <ex-2_unknown>
> TX analysis complete
```

### Sample 2

The following run shows the working of the tool with non-default input file name. Note that this file contains an invalid entry which is reported by the tool as a warning. In this case, the tool prompts the user whether they want to abort (e.g., to fix the issues) or continue with the valid entries. The number of transactions processed per second is quite consistent.

```
an4s@ubuntu:~$ bitcoin-qt -enable-tx-analysis -ta-input-filename=MyInputFile
> TX analysis enabled
> Initializing TX analyzer
> INFO - file </home/an4s/.bitcoin/tx-analysis-files/ex-3> added to processing queue
> WARN - file path: </home/an4s/.bitcoin/tx-analysis-files/ex-9> doesn't exist
> INFO - file </home/an4s/.bitcoin/tx-analysis-files/ex-4> added to processing queue
> TX analyzer initialized successfully
> Running TX analysis
> some files failed to be opened successfully. continue? [y/n]
y
>> INFO - Reading hashes from file: <ex-3>
>> INFO - Successfully read hashes from file: <ex-3>
>> INFO - Beginning transaction analysis...
Progress:	10000/10000 (100.0000%) [elapsed 00:03:36.834]
UKN: 4
EXC: 0
>> INFO - writing transaction fees and sizes to file: <ex-3_out>
>> INFO - writing transaction parents to directory: <ex-3_parents>
>> INFO - writing hashes of unknown transactions to file: <ex-3_unknown>
>> INFO - Reading hashes from file: <ex-4>
>> INFO - Successfully read hashes from file: <ex-4>
>> INFO - Beginning transaction analysis...
Progress:	10000/10000 (100.0000%) [elapsed 00:03:02.748]
UKN: 4
EXC: 0
>> INFO - writing transaction fees and sizes to file: <ex-4_out>
>> INFO - writing transaction parents to directory: <ex-4_parents>
>> INFO - writing hashes of unknown transactions to file: <ex-4_unknown>
> TX analysis complete
```

### Sample 3

The following run shows the working of the tool with invalid value provided to the `-ta-input-filename` argument (i.e., this file does not exist). The tool detects this while initializing and terminates right away.

```
an4s@ubuntu:~$ bitcoin-qt -enable-tx-analysis -ta-input-filename=InvalidFile
Warning: Ignoring XDG_SESSION_TYPE=wayland on Gnome. Use QT_QPA_PLATFORM=wayland to run on Wayland anyway.
> TX analysis enabled
> Initializing TX analyzer
> ERROR - input path: </home/an4s/.bitcoin/tx-analysis-files/InvalidFile> doesn't exist
> TX analyzer initializing failed
```

## Epilogue

Please provide feedback on the tool and report issues, if you find any. I intend to port the tool forward to future versions of Bitcoin, time permitting. Feel free to use the tool, port it forward and create PRs. Your contributions are welcome!

My research works that have used results from this tool can be found [here](https://github.com/nislab/bitcoin-releases).
