# Network-Aware BWA 0.7.19 - Build Instructions

This is a port of the network-aware BWA features from version 0.5.10 to BWA 0.7.19.

## Features

- **bam2bam command**: Direct BAM-to-BAM alignment workflow
- **fastq2bam command**: Single-end FASTQ/FASTA-to-BAM alignment (same engine as bam2bam)
- **Network distribution**: ZeroMQ-based distributed alignment across compute nodes
- **Ancient DNA optimized**: Improved settings for aDNA analysis
- **Single-end and paired-end support**: Handles both read types automatically
- **64-bit compatible**: Full support for large genomes

## Prerequisites

### Required:
- GCC or compatible C compiler
- zlib development libraries
- make

### Optional (for network features):
- ZeroMQ development libraries (libzmq)
- pkg-config

## Quick Build

For local-only use (no network features):

```bash
make clean
make
```

This will build the `bwa` binary with bam2bam support but without network distribution.

## Build with Network Support

To enable network-aware features (requires ZeroMQ):

```bash
# Install ZeroMQ first (example for Ubuntu/Debian)
sudo apt-get install libzmq3-dev

# Then build with network support
make clean
make NETWORK=1
```

Or for systems with pkg-config:

```bash
make clean
make NETWORK=1 CFLAGS="-g -Wall -O2 $(pkg-config --cflags libzmq)" LIBS="-lm -lz -lpthread $(pkg-config --libs libzmq)"
```

### macOS (Homebrew)

ZeroMQ is always linked, so install it and point `make` at Homebrew's paths:

```bash
brew install zeromq
make clean
make INCLUDES="-I$(brew --prefix)/include" LDFLAGS="-L$(brew --prefix)/lib"
```

(`-lrt` is automatically omitted on non-Linux, so no other changes are needed.)

## Installation

```bash
sudo make install
```

Or copy the `bwa` binary to your preferred location:

```bash
cp bwa /usr/local/bin/
```

## Testing

### Test bam2bam with paired-end data:
```bash
# Create test index
./bwa index reference.fasta

# Convert FASTQ to unaligned BAM
samtools import -@ 4 -1 reads_1.fq -2 reads_2.fq -o unaligned.bam

# Run bam2bam
./bwa bam2bam -g reference.fasta -f aligned.bam unaligned.bam
```

### Test bam2bam with single-end data:
```bash
# Create unpaired BAM (reads must have unpaired flags)
samtools import -@ 4 -0 reads.fq -o unpaired.bam

# Run bam2bam
./bwa bam2bam -g reference.fasta -f aligned.bam unpaired.bam
```

### Test fastq2bam with single-end data:
```bash
# No BAM conversion needed -- align straight from FASTQ/FASTA
./bwa fastq2bam -g reference.fasta -f aligned.bam reads.fq.gz
```

### Ancient DNA settings:
```bash
./bwa bam2bam   -g reference.fasta -n 0.01 -o 2 -l 1024 -t 16 -f aligned.bam unaligned.bam
./bwa fastq2bam -g reference.fasta -n 0.01 -o 2 -l 1024 -t 16 -f aligned.bam reads.fq.gz
```

## Usage

### bam2bam Command

```bash
bwa bam2bam [options] <in.bam>

Options:
  -g, --genome PREFIX               prefix of genome index files (required)
  -f, --output FILE                 output BAM file [stdout]
  
  -n, --num-diff NUM                max #diff or missing prob [0.04]
  -o, --max-gap-open INT            maximum gap opens [1]
  -e, --max-gap-extensions INT      maximum gap extensions [-1]
  -l, --seed-length INT             seed length [32]
  -k, --seed-mismatches INT         max differences in seed [2]
  
  -t, --num-threads INT             number of threads [1]
  -p, --listen-port PORT            listen for workers on PORT [0]
  
  --only-aligned                    output only aligned reads
  --drop-aligned                    skip already aligned reads
  --broken-input                    handle BAM with mismatched pairs
```

### fastq2bam Command

```bash
bwa fastq2bam [options] <in.fastq>     # single-end FASTQ/FASTA (use '-' for stdin)
```

Accepts the same alignment options as `bam2bam` (`-g -f -n -o -e -l -k -t -p ...`).
Paired-end is not supported; the BAM-pairing options (`-0/-1/-2`, `--broken-input`)
do not apply.

### Network Distribution

Add `-p PORT` to the master to accept remote workers. The master binds three ports:
`PORT` (config), `PORT+1` (work) and `PORT+2` (insert-size broadcast) -- open all three
in any firewall.

**On the master node** (also using 16 local threads here):
```bash
./bwa fastq2bam -g reference.fasta -l 1024 -n 0.01 -o 2 -t 16 -p 5555 -f aligned.bam reads.fq.gz
# (bam2bam works the same way with an unaligned BAM as input)
```

**On each worker node:**
```bash
./bwa worker -h <master-host> -p 5555 -t <N>
```

The master sends the worker the alignment parameters **and its own genome-prefix
path**, which the worker loads from its *local* filesystem. If the worker doesn't have
the index at that path (e.g. a different OS/mount), use `-g` to load a local copy --
parameters still come from the master, so results are identical:

```bash
./bwa worker -g /local/path/reference.fasta -h <master-host> -p 5555 -t <N>
```

Each worker needs the index files (`.amb .ann .bwt .pac .sa`) locally. To copy them to
another machine over Tailscale:

```bash
sudo tailscale set --operator=$USER      # one-time, so tailscale runs without sudo
tailscale file cp reference.fasta.amb reference.fasta.ann reference.fasta.bwt \
                  reference.fasta.pac reference.fasta.sa <peer-name>:
# then on the peer:  tailscale file get <dir>
```

## Key Improvements Over 0.5.10

1. **Better mapping rates**: 64-bit BWA 0.7.19 algorithms provide 4-20% higher mapping
2. **Perfect mate rescue**: Smith-Waterman rescue eliminates singletons in paired-end data
3. **No memory corruption**: All 32-bit/64-bit serialization issues fixed
4. **Larger genomes**: Full 64-bit support for genomes >4GB

## Troubleshooting

### "cannot find -lzmq" error
Install ZeroMQ development libraries or build without network support (don't use NETWORK=1)

### "got a pair, but the flags are wrong"
Your BAM file has paired-end flags but mismatched read names. Use `--broken-input` flag.

### "got two reads, but the names don't match"
For single-end data, ensure reads have unpaired flags (flag 4 for unmapped, not flag 77).

## Changes from Original BWA 0.7.19

- Added `bam2bam.c` - Main bam2bam implementation
- Added `fastq2bam` subcommand - single-end FASTQ/FASTA input into the bam2bam engine
  (`read_fastq_single` in `bwaseqio.c`)
- Added `bwa worker -g/--genome` - load the index from a local path instead of the
  master-supplied one (for workers on a different host/mount)
- Added `insert_size.c` - Insert size distribution tracking
- Added `bgzf.c/h` - BGZF compression support
- Added `bwape.h` - Paired-end header definitions
- Added `main.h` - Command dispatch header
- Modified pairing and mate rescue code for network serialization
- Fixed 32-bit/64-bit structure compatibility

## License

Same as BWA: GPLv3

## Citation

If you use this software, please cite:
- Li H. and Durbin R. (2009) Fast and accurate short read alignment with Burrows-Wheeler Transform. Bioinformatics, 25:1754-60.
- Original network-aware BWA implementation

## Contact

For issues specific to this port, please report at the repository where you obtained this code.
