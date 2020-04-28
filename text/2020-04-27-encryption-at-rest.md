# Encryption-At-Rest

An earlier version of this proposal can be find at: https://docs.google.com/document/d/14u0YKHXxjE0a9wAd3tsu0Hqjmknjr0bYU_EMzZSbzkA/edit?usp=sharing

## Summary

This is a design proposal to support encryption-at-rest in TiKV. In this document we will discuss design and implementation details of:
* How to config TiKV to enable encryption and how TiKV handle encryption key rotation.
* How TiKV encrypt data files through underlying storage engine (i.e. RocksDB)
* How TiKV handle temporary files (e.g. snapshot files) when encryption is enabled.
* How TiKV interact with other TiDB tools when encryption is enabled.

## Motivation

Encryption at rest means that data is encrypted when it is stored. As opposed to encryption in flight (TLS) or encryption in use (rarely used). Different things could be doing encryption at rest (SSD drive, file system, cloud vendor, etc), but if the database does  encryption by itself this helps ensure that attackers must authenticate with the database . For example, even when an attacker gains access to the physical machine, he cannot access data by copying files on disk.

A block cipher is an algorithm, that given a symmetry key, encrypts a fixed-sized data block (plaintext) into a block of same size (ciphertext). AES is a block cipher of fixed 128 bits (16 bytes) block. The key size could be 128, 192, 256 bits, referred to as AES128, AES192, AES256 respectively. 

Encrypting data block by block using the same key is insecure because it leaks out patterns of the data. For example, an attacker would know if two blocks have the same ciphertext, it means they have the same plaintext, even if the key is not compromised. Block cipher mode of operation is a kind of algorithm to encrypt each block of data differently so that different blocks with the same plaintext yield different ciphertext. CTR (also called counter mode) is one such algorithm which is commonly used and suitable to support random reads. Each block is encrypted together with its block index, making sure different blocks are encrypted differently. This Wiki page has a very good explanation of the matter and the CTR algorithm: https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation.

RocksDB provides an API called Env which abstracts file system APIs. In particular, it provides EncryptedEnv implementation which encrypts all RocksDB files (SST, WAL, blob files for Titan, etc) transparently when it is enabled. It implements CTR, and requires the caller to provide a block cipher implementation. TiKV currently supports using EncryptedEnv, but supplies it with a XOR block cipher (simply XOR the key with data block), which is insecure.

Both CockroachDB and MongoDB support encryption at rest. Key rotation with CockroachDB requires restarting the process. CockroachDB also doesn’t support encrypting backups. MongoDB supports encrypting backups, and also supports key management through KMIP. Both CockroachDB and MongoDB don't encrypt their info log, though MongoDB provides an option to avoid logging sensitive data in info log.

## Detailed Design

### Outline

We will be using envelope encryption in order to support key rotation. This is the same approach other databases (CockroachDB, MongoDB) we studied uses. There are two kinds of keys being used when encryption is enabled:

* master key: User provided key. The user supplies TiKV with the master key on TiKV startup. Management of the master key is external to TiKV.
* data key: TiKV generates the data key to encrypt actual data files. The list of data keys used is persisted with TiKV data and encrypted using the master key. The data key is rotated by TiKV automatically every week by default (configurable).

We support the following encryption methods: AES128-CTR, AES192-CTR and AES256-CTR. Users can specify the data file encryption method in TiKV config. Encryption is disabled by default. 

TiKV implements a key manager module to handle master key and data key management. The module is used to handle:

* Persist encrypted data keys and encryption info (encryption key, method and IV used) for each data file.
* Dectect and handle master key rotation.
* Interface with storage engine (i.e. RocksDB) to pass encryption key, method and IV used to encrypt or decrypt a file.

The actual encryption happens both in storage engine (i.e. RocksDB) and out side of storage engine (for temporary files used by TiKV, like snapshot files and imported backup files). In either of the cases, OpenSSL is used to encrypt the files and the encryption format are compatible with each other. This means, for example, SST files generated and encrypted outside of RocksDB by TiKV are able to ingest into RocksDB as is, as long as the key manager module is able to pass encryption info along correctly.

### Providing and Rotating Master Key

Management of master key is external to TiKV. Master key can be provided to TiKV in one of the following ways. We need to update config format accordingly.

* Managed in AWS Key Management Service (KMS, https://aws.amazon.com/kms/). Currently we only support AWS KMS.
* Pass through a file, whose path is specified in TiKV config.

KMS is the preferred way for master key management in production. To use it, user would need to specify the KMS CMK to use in TiKV config. There are two APIs that TiKV uses from KMS service: `GenerateDataKey` and `Decrypt`. The `GenerateDataKey` is capable of generating a new encryption key and return both of its plaintext and ciphertext form. When encrypted TiKV data keys, TiKV will call `GenerateDataKey` to generate a KMS data key (not to confuse with TiKV data key) and use it to encrypt TiKV data keys, and store the ciphertext key along with the TiKV data keys. When TiKV need to load TiKV data keys from disk, TiKV will call the `Decrypt` API the decrypt the ciphertext key, and use the result as the key to decrypt the TiKV data keys.

Note the extra layer of envolope encryption used here (KMS CMK -> KMS data key -> TiKV data key). We don't use KMS data key directly as TiKV data key is because AWS KMS currently doesn't provide a way for bulk decrypt. If we accumulate a number of data keys in the system (think of by default we rotate data key every week, after a year there will be 50+ data keys co-existing), on TiKV restart we don't want to send multiple requests to AWS to decrypt them one by one.

Alternatively passing master key stored in a file is also supported, but it is not recommended to use in production. In that case the plaintext key is used directly to encrypt and decrypt (TiKV) data keys.

In both of the cases, AES256-GCM is used to encrypt the (TiKV) data keys. The main reason to use GCM mode instead of CTR mode is not to verify authenticity of the persisted data keys, but rather provide a way to verify if the right master key is provided to decrypt the key. Without using an authentication encryption method like GCM, we don't have reliable way to tell if user provide a different key than the one use previously to encrypt the data keys. The use of AES256 here is non-configurable, in order to reduce configurable knobs that could confuse user.

To rotate master key, user will need to specify both of the new master key and old master key in config. On restart, TiKV will first try using the current (new) master key to decrypt its data keys. If it finds out the master key is not the right key, it try fallback to use the old master key. If it success, data keys are re-encrypted using the new master key and get persisted again. After that, the key rotation is finished and the new master key will be used and the old master key will be ignored and safe to be destroyed or disabled. The various cases on TiKV restart will be discuss at the end of next section.

### Data Key Management

The key manager module stores two additional files:

* encryption key dictionary (key.dict): Maps data key id to data keys, together with metadata. Also stores the data key id used to encrypt the encryption file dictionary. The content is encrypted using the master key.
* encryption file dictionary (file.dict): Maps data file paths to key id and metadata. This file is stored as plaintext.

Except for the encryption key dictionary, TiKV never persists any keys in plaintext or ciphertext.

Both of the files share the same file format, as depicted as follows:

+----------------------------------------------+---------------------+
|                   header                     |                     |
+----------+-----------+-----------+-----------+ serialized content, |
| version  |   flags   |    crc    | file size | could be encrypted  |
| (1 byte) | (3 bytes) | (4 bytes) | (8 bytes) |  (variable length)  |
+----------+-----------+-----------+-----------+---------------------+

Version and flags are reserved for possible future update of file format. CRC and file size is used to validate content. On update of each dictionary, content is dumped from memory and written to file as a whole. No delta update for now. We need to fsync directory each time we update the files.

If file dictionary becomes too large and it is too expensive to rewrite the whole file for each data file creation, we may need to compress it or even shard it. Let’s keep it simple for now.

Content of both of key dictionary and file dictionary are encoded using protobuf. The following shows the structure of content of the file dictionary. Files not tracked in file dictionary are assumed to be stored in plaintext. Path is absolute path in the file system. 

```
message FileInfo {
    EncryptionMethod method = 1;
    uint64 key_id = 2;
    bytes iv = 3;
}

message FileDictionary {
    map<string, FileInfo> files = 1;
}
```

The following shows the structure of content of the key dictionary. Key id is a unique random number generated together with the key. The flag was_exposed is to track whether the key has ever been persisted as plaintext (by turning off encryption, in which case key dictionary will be stored as plaintext). File dictionary encryption info is also stored in key dictionary.

```
enum EncryptionMethod {
    UNKNOWN = 0;
    PLAINTEXT = 1;
    AES128_CTR = 2;
    AES192_CTR = 3;
    AES256_CTR = 4;
}
 
message DataKey {
    bytes key = 1;
    EncryptionMethod method = 2;
    uint64 creation_time = 3;
    bool was_exposed = 4;
}

message KeyDictionary {
    map<uint64, DataKey> keys = 1;
    FileInfo file_dict = 2;
    uint64 current_data_key = 3;
}
```

The `KeyDictionary` and `FileDictionary` struct are serialized, possibly encrypted (only for key dictionary), and stored in the `EncryptedContent` struct and get serialized again before write to their corresponding files.

```
message EncryptedContent {
    map<string, bytes> metadata = 1;
    bytes content = 2;
}
```

The `metadata` field may contain the following keys:

* "method": The method used to encrypt the content, value could be "plaintext" or "aes256-gcm".
* "iv": The IV used to encrypt the content, if applicable.
* "aes_gcm_tag": When GCM mode is used to encrypt the content, stored the tag being generated along with encryption.
* "kms_vendor": If KMS is used to encrypt the content, the vendor for the KMS service. Currently only support "AWS".
* "kms_ciphertext_key": If KMS is used to encrypt the content, the data key generated from the KMS service (not to confuse with data key generated by TiKV) in ciphertext.

On restart, the key manager module checks the following as part of initialization:

* If encryption is disabled in config, and both of the key dictionary and file dictionary are not presented, that means no data files are encrypted.
* If one and only one of key dictionary and file dictionary is presented, TiKV panics since this is a data corruption.
* If encryption is enabled, TiKV will load data keys from key dictionary by using the current master key to decrypt it. If it find the master key is not the right key when decrypting key dictionary (happens when either 1. KMS service return an error indicating a wrong key is given when decrypting KMS data key, or 2. GCM tag mismatch when decrypting key dictionary), and user have provided a previous master key as alternative, TiKV fallback to use the previous master key to decrypt the key dictionary. If that success, the new master key is used to re-encrypt and persist the key dictionary, after which master key rotation is considered success. If again the previous master key is not the right key, TiKV will panic on restart.
* If encryption is disabled in config, but both of the dictionaries presents, TiKV will use the master key andprevious master key in config in order to decrypt the key dictionary, and if success, rewrite the key dictionary as plaintext and set the `was_exposed` flag. Panic if failed.
* On any other errors (checksum mismatch for either of the dictionaries, other KMS service error, other encryption related errors, etc), TiKV will panic on restart.

### Encrypting Data Files in Storage Engine

For each file to encrypt, we randomly generate an IV for each file. They will be stored in the encryption file dictionary. Together with the current data key, they are passed to EncryptedEnv to construct a file object (rocksdb::WritableFile, rocksdb::RandomAccessFile, etc).

There are several things we need to change in EncrptedEnv. Currently EncryptedEnv stores a 4k header in each of the files, which include the nonce and initial counter. (The rest of the header is unused.) The header is stored in plaintext. CockroachDB changed that to store the nonce and initial counter in file map, since the file map needs to exists anyway to track encryption status of files. It also simplifies the logic by making encrypted file size the same as plaintext.

A KeyManager API needs to be added for EncyptedEnv to callback to TiKV to:
* query existing file encryption metadata.
* update file dictionary when creating a file, rename a file or create a hard link.
* remove file from file dictionary when deleting a file

### Encrypting Temporary Files

By far, TiKV will create temporary files that may contain user data in two cases: creating a region snapshot and send over to a peer TiKV node, or restoring a backup using the BR tool.

Current flow of creating a region snapshot:

1. The sender reads region data from storage engine and generate temporary files and store locally before sending. For the data read from default CF or write CF, RocksDB SST writer is used to generate RocksDB SST files. For data read from lock CF, the data is written in a custom format called plain file.
2. The files are send over network via gRPC.
3. The receiver saves the files locally. If there are additional updates in the region, the files are rewritten.
4. In case of SST files, they are ingested into RocksDB directly. The plain files are scanned and the data is also written into RocksDB.

In step 1, TiKV should encrypt the temporary files before writing them to disk, decrypt again when reading the file for sending. In step 3, TiKV should encrypt the file before writing them to local storage. SST files can be ingested into RocksDB directly, provided key manager will pass the same encryption key, method and IV along to RocksDB. The plain files are decrypted as they are being read. The files are transfer over network as plaintext from the view of TiKV raftstore. It is adviced to enable TLS on TiKV to keep network transfer encrypted.

For restoring a backup using BR, the current flow is:
1. Copy the files from storage backend (local, S3 or GCS) to local storage
2. Ingest the files to RocksDB accordingly.

In step 1, TiKV should encrypt the file before writing the file to local storage, and make sure key manager pass the same encryption key, method and IV to RocksDB when requested.

For both of the flow above, SST file ingestion is used. TiKV currently have a specially logic to make an extra copy of the file, so if the ingestion fails, it can use the extra copy to retry. The key manager should keep track of the extra copy as well.

As far as we aware of, no other tooling (Backup using BR, TiDB Ligntening, TiCDC, etc) will create temporary files in TiKV. TiKV directly read or write to storage engine for those toolings, where encryption should be handled by the `EncryptedEnv` layer.

## Drawbacks

* * When deploying a TiDB cluster, the majority of user data are stored in TiKV nodes, and those data are encrypted-at-rest when encryption is enabled. However there are small amount of user data stored in PD nodes as metadata. As of v4.0.0, PD doesn't support encryption-at-rest. As a workaround, it is recommended to use storage level encryption (e.g. file system encryption) to protect sensitive data stored in PD.
* TiKV currently does not exclude encryption keys and user data from core dump. It is highly adviced to disable core dump for TiKV process when using encryption-at-rest. This is not currently handled by TiKV itself.
* TiKV keeps track of encryption key and method used for each data files, using absolute path of the files. As a result, once encryption is turned on for a TiKV node, user should not change data file paths config such as `storage.data-dir`, `raftstore.raftdb-path`, `rocksdb.wal-dir` and `raftdb.wal-dir`.
* Currently in many places TiKV will log user data to info log, which leaks user data. The issue is not addressed in current approach.

## Alternatives

Using storage level encryption (file system encryption, etc). As discussed previously, it doesn't prevent attacker that gain access to the TiKV host to access senstive user data.

## Unresolved Questions

See the drawbacks section, otherwise there's no unresolved questions so far.

