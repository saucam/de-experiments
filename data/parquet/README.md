# Parquet For Data Processing

This is a demo of parquet data format and its capabilities for big data processing.

## Setting up the environment

For this demo you need:

- conda (can follow the steps [from here](https://docs.conda.io/projects/conda/en/latest/user-guide/install/index.html) to install it)

After installing conda, execute the below command to setup all required dependencies

```
conda env create -f environment.yml
```
This will create a new conda env named ```parquet_exp``` with all required dependencies installed.
Then, activate the env

```
conda activate parquet_exp
```

Now you are ready to follow along, fire up jupyter-lab and navigate to ```parquet.ipynb``` notebook to start executing the experiment.

```
jupyter-lab
```

## Background

How the data lives in the storage layer determines the query types that can be supported as well as the speed of retrieval. Therefore, datastore should be carefully chosen by considering the underlying format and if that matches the query requirements of the system to be designed.
[Apache parquet](https://parquet.apache.org/) is an open source, column-oriented data file format designed for efficient data storage and retrieval. It provides efficient data compression and encoding schemes with enhanced performance to handle complex data in bulk.

## Storage Format

So how exactly does parquet store the tabular data into files for efficient storage and retrieval?

A single parquet file stores the data in the following format:

The rows of the table are split into RowGroups. Each RowGroup will have data for a fixed number of rows. For each RowGroup, parquet will first store the Column1 values together (called a Chunk), followed by something called Column Metadata. The metadata has certain additional information about the column like the data type (string, int, bool etc), number of values stored, codec, encodings etc. Then it will store the next Column2 values (Chunk) together, followed by the column 2 metadata and so on for each of the N columns of the table. This completes the RowGroup 1 data. Then parquet repeats the same structure for RowGroup2 and so on till all the M RowGroups of the data are stored.
After all RowGroups are stored, parquet will add file metadata for the file which has offsets of each of the Column Chunks written before along with RowGroup Metadata. Readers are expected to first read the file metadata to find all the column chunks they are interested in. The columns chunks should then be read sequentially.

This looks like below:

```
    4-byte magic number "PAR1"
    <Column 1 Chunk 1 + Column Metadata>
    <Column 2 Chunk 1 + Column Metadata>
    ...
    <Column N Chunk 1 + Column Metadata>
    <Column 1 Chunk 2 + Column Metadata>
    <Column 2 Chunk 2 + Column Metadata>
    ...
    <Column N Chunk 2 + Column Metadata>
    ...
    <Column 1 Chunk M + Column Metadata>
    <Column 2 Chunk M + Column Metadata>
    ...
    <Column N Chunk M + Column Metadata>
    File Metadata (Includes Column Chunk Offsets + RowGroup Metadata)
    4-byte length in bytes of file metadata (little endian)
    4-byte magic number "PAR1"
```

Here

```
Column 1 Chunk 1 + Column 2 Chunk 1 ... Column N Chunk 1 = RowGroup 1
```

And so on.

## Why is it important to know these details?

Because of this specific way of storing data, we get several advantages:

- Since column values are stored one after the other and each Column Chunk Offset is maintained as metadata in the file metadata, we can skip reading chunks of columns we are not interested in. For example if a query select only 2 out of 7 columns in the schema, parquet reader can skip reading Column chunks of 5 other columns, which translates to huge savings in I/O and therefore making queries over big datasets very fast. This is called **Columnar projection**. In contrast, if we use a standard row oriented RDBS, it will scan the data for all columns regardless of what columns are selected in the query.
- Since column values are stored together which are of same type, it is easier to encode and compress the Column Chunks, leading to impressive reductions in storage size.
- Parquet readers can be supplied filters while reading the data (which are part of the queries to the table). This is also called **Predicate pushdown**. Once readers scan the FileMetadata, they can read RowGroup Metadata that can get them statistics like Min and Max of each Column Chunk in the RowGroup. This can effectively eliminate the need to read a RowGroup in case the range of values inside that RowGroup fall outside the filter range, thereby reducing I/O. This has huge implications when querying very large datasets, where range filters when passed on to the parquet readers can reduce I/O significantly, thereby making queries a lot faster.

For example, if we execute the following query in spark on a table t that has its data stored in parquet:
```
select * from t where c1 > 1000;
```
In this case, the filter c1 > 1000 can be pushed down to the parquet reader, which can eliminate reading RowGroups where column c1 values never exceed 1000, just by reading the metadata.
- Parquet supports nested columns, using **[Dremel encoding](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/36632.pdf)**. This means we can use parquet even if our requirements warrant deeply nested schemas.
- Parquet also supports creating specialized filters like **Bloom Filter** for specific use cases.
- Parquet support data encryption, while allowing for a regular Parquet functionality (columnar projection, predicate pushdown, encoding and compression).