BFF -- Forked Version for the Personal Projects
===


### Installation
1. To read and write dataset in the `jsonl` format (e.g., the Pile), run `cargo build --release --bin jsonl` to compile and generate the binary executable file. Alternatively, run `cargo run --bin jsonl [args]` to compile and immediately run the executable.
2. To read and write dataset in the `json.gz` format, use the default format or pass `--bin json-gz` instead.
3. Accelerate the deduplication for multiple files by passing `threads`.

Trouble-shooting: 
1. if `cargo` isn't installed in your system, run 
```bash
sudo snap install rustup --classic  # install
rustup default stable  # setup default configuration
```
2. if `cmake` isn't installed, run
```bash
sudo apt update
sudo apt install cmake
```

### Usage
Comands for the Pile deduplication:
```bash
target/release/jsonl \
  --bloom-filter-file filter.bff \
  --bloom-filter-size 2147483648 \
  --expected-ngram-count 1000000000 \
  --output-directory deduped_Github/ \
  --threads 32 \
  $PATH_TO_PILE/Github/*.jsonl
```
Note: The suggested bloom filter size for the 1000000000 expected ngram is 2147483648. I am not sure about how to properly set it to achieve a good accuracy-efficiency trade-off.

### Choosing Hyperparameters

#### How does Bloom filter work?

A Bloom filter is a probabilistic data structure designed to test whether an element is a member of a set. It is very efficient in terms of space and time, but it allows for some possibility of false positives (indicating an element is in the set when it is not).

A bloom filter starts as an array of bits, all set to 0, and several hash functions. When you add an item, it is processed by each hash function, each of which outputs an index to the bit array. The bits at these indices are set to 1. To check if an item is in the set, process it with the same hash functions. If all the bits at the resulting indices are 1, the items is **presumed** to be in the set. If any bit is 0, the item is **definitely not** in the set.

#### Choosing the size of a Bloom filter.
The key is to balance the trade-off between the space requirement and the false positive rate. There are two main factors:

* `n`: The number of ngrams expected to be stored.
* `m`: The size of the Bloom filter in bits.
* `P`: The false positive probability, usually 0.01-0.02 is acceptable. 

The number of hash functions `k` and the false possitive probability can be calculated as follows:

$$
k = \lceil \frac{m}{n} \ln 2 \rceil
$$

$$
P = \left( 1 - \left( 1 - \frac{1}{m} \right)^{kn} \right)^k \
% \approx \left( 1 - \frac{kn}{8m} \right)^k \
% \approx \left( 1 - \frac{\ln 2}{n} \right)^k = 1
$$


The optimal filter size `m` is

$$
m = - \frac{n \ln P}{(\ln 2)^2}
$$

Here is a quick lookup table used for experiments in the Scaling experiments:

| Domain | Num. Ngrams | False Possibility Rate | Bloom Filter Size in Bits| Bloom Filter Size in GB |
|------------|------------|------------|------------|------------|
| Full | 1400000000000 | 0.02 | 11399309000000 | 1424 |
| PubMed | 6500000000 | 0.02 | 52925361687 | 6.6 |


Example script:
```bash
SECONDS=0

mkdir deduped_pubmed
target/release/jsonl \
  --bloom-filter-file filter.bff \
  --bloom-filter-size 52925361687 \
  --expected-ngram-count 6500000000 \
  --output-directory deduped_pubmed/ \
  --threads 64 \
  /mnt/md-256k/massive_ds_data/full/pubmed/*.jsonl

echo "The script took $SECONDS seconds to execute."
```

BFF
=== 

The big friendly filter 😁

Getting started
---------------

1. Install Rust on your machine.
    1. `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`
    2. Add `~/.cargo/bin` to your `PATH` environment variable.
2. Run `cargo build --release`. It places the binary at `target/release/bff`.
3. Run `./target/release/bff --help` to see the available options.

Examples
--------

### Deduplicating a file against itself

This is how you deduplicate a file against itself:
```bash
target/release/bff \
  --bloom-filter-file filter.bff \
  --bloom-filter-size 274877906944 \
  --expected-ngram-count 1000000000 \
  --output-directory deduped/ \
  input.json.gz
```

This creates the filter at `filter.bff`, with a size of 256 GB.
This size should be a little smaller than the amount of main memory you have.
It calculates the optimal setup for the filter based on the expected number of ngrams.
Getting that number right is very important.
If in doubt, guess high.
It's safer to guess a higher number than a lower number.
The filter will be created in memory, and only written to disk at the end of the job.

### Deduplicating multiple files

To get a lot of speed out of `bff`, you have to process multiple files at once:
```bash
target/release/bff \
  --bloom-filter-file filter.bff \
  --bloom-filter-size 274877906944 \
  --expected-ngram-count 1000000000 \
  --output-directory deduped/ \
  *.json.gz
```

Each input file will run in its own thread, and the filter will be shared between them.
In the end, as before the filter will be written to disk.

### Pre-load the filter

You can stick ngrams into the filter ahead of time, for example if you want to decontaminate your dataset:
```bash
target/release/bff \
  --bloom-filter-file decontaminating_filter.bff \
  --bloom-filter-size 274877906944 \
  --expected-ngram-count 1000000000 \
  --output-directory deduped/ \
  --filtering-threshold 1.0 \
  my_test_set.json.gz
```

This will copy the output unchanged to the `deduped/` directory, but it will also produce a filter that you can use afterwards.
It is important that you still take a good guess at the ngram count you expect to see when you do the actual
deduplication.
The parameters of the bloom filter are baked in when you first create the file, so you have to guess right the
first time.

### Only decontaminate

If you only want to decontaminate, but not deduplicate against itself, you can do that by using the filter
you just created in the previous step:
```bash
target/release/bff \
  --bloom-filter-file decontaminating_filter.bff \
  --bloom-filter-size 274877906944 \
  --expected-ngram-count 1000000000 \
  --output-directory deduped/ \
  --update-bloom-filter false \
  *.json.gz
```

If you are using the filter this way, you can use the number of ngrams in the decontamination set for the
`--expected-ngram-count` parameter.
Since this is usually much smaller, it might make the filter run faster.
