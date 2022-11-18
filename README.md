# BUSZ-format

The BUSZ format is a compressed format of the BUS format. This repository details the specification of the format.


## Format specification

A BUSZ file is a binary file consisting of a header, followed by zero or more compressed blocks of BUS records, ending with an empty block. The BUSZ header includes all information from the BUS header, along with compression parameters. BUSZ files have a different magic number than BUS files.

An additional index file can be generated during compression, indexing the barcodes in the BUSZ file.

### BUSZ header

A BUSZ header contains the following elements in order:

|Field name | Description | Type | Value |
|-----------|-------------|------|-------|
| magic | fixed magic string | char[4] | BUS\1 |
| version | BUS format version | uint32_t | |
| bc_len | Barcode length [1-32] | uint32_t | |
| umi_len | UMI length [1-32] | uint32_t | |
| tlen   | Length of plain text header | uint32_t | |
| text | Plain text header | char[tlen] |  |
| block_size | Number of rows in each block | uint32_t | |
| pfd_block_size* | Size of a NewPFD block | uint32_t | |
| lossy_umi** | Indicate whether a lossy compression was used on UMIs | uint32_t | |

*The equivalence classes are compressed using the NewPFD compression scheme. It subdivides the input into blocks of `pfd_block_size`.

**Lossy compression of UMIs is not implemented. Thus, this value is always false until implemented.


### BUSZ compressed block

Each compressed block contains a block header and a block of compressed BUS records. The block header indicates the size of the succeeding block. It is a single `uint64_t` where the 34 most significant bits denote the size of the compressed block in bytes. The 30 least significant bits denote the number of BUS records in the block.

```c++
uint64_t block_size_bytes = ...;
uint64_t block_size block_size_records = ...;
uint64_t block_header = (block_size_bytes << 30) | block_size_records;
```
The block header for the empty block is 0 and denotes the EOF.

The compressed block contains transposed records from the BUS file, such that all the barcodes in the block are stored in a row, then all the UMIs in the block, etc. These are stored in the order they come in the BUS file.

### Index file

The index file is a binary file consisting of a header, followed by zero or more indices into the BUSZ file.

The header contains the following elements in order:

| Field name| Description  | Type | Value |
|------------|-------------|------|-------|
| magic |fixed magic string|char[4]| BZI\0|
| n_blocks| Number of compressed blocks in the BUSZ file| uint64_t | |
| block_size| The block_size in the corresponding BUSZ header | uint64_t | |
| last_block_size| The number of BUS records in the last non-empty block | uint64_t | |

The number of BUS records can be computed using the header values: `n_records = (n_blocks-1) * block_size + last_block_size`.

Following the header come `block_size` indices.
The indices are in the order of the compressed blocks in the BUSZ file. Each index entry consists of the following elements in order:

| Field name| Description  | Type |
|------------|-------------|------|
| barcode | The first barcode in the compressed block | uint64_t |
| byte_offset | The offset in bytes of the compressed block | uint64_t |

