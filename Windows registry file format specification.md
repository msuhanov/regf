# Windows registry file format specification
-------

## Table of contents
  * [Types of files](#types-of-files)
    * [Examples of hives and supporting files](#examples-of-hives-and-supporting-files)
      * [Windows 8.1: System hive](#windows-81-system-hive)
      * [Windows 8.1: BCD hive](#windows-81-bcd-hive)
      * [Windows Vista: System hive](#windows-vista-system-hive)
      * [Windows Vista: SAM hive](#windows-vista-sam-hive)
      * [Windows XP: System hive](#windows-xp-system-hive)
  * [Transactional registry (TxR)](#transactional-registry-txr)
  * [Format of primary files](#format-of-primary-files)
    * [Base block](#base-block)
      * [Notes](#notes)
    * [Hive bin](#hive-bin)
      * [Notes](#notes-1)
    * [Cell](#cell)
      * [Index leaf](#index-leaf)
      * [Fast leaf](#fast-leaf)
      * [Hash leaf](#hash-leaf)
        * [Note](#note)
      * [Index root](#index-root)
        * [Notes](#notes-2)
      * [Key node](#key-node)
        * [Flags](#flags)
        * [Virtualization control flags](#virtualization-control-flags)
        * [Debug](#debug)
      * [Key values list](#key-values-list)
      * [Key value](#key-value)
        * [Data size](#data-size)
        * [Data types](#data-types)
        * [Flags](#flags-1)
      * [Key security](#key-security)
        * [Notes](#notes-3)
      * [Big data](#big-data)
        * [Data segment](#data-segment)
    * [Summary](#summary)
  * [Format of transaction log files](#format-of-transaction-log-files)
    * [Old format](#old-format)
      * [Base block](#base-block-1)
        * [Notes](#notes-4)
      * [Dirty vector](#dirty-vector)
        * [Notes](#notes-5)
      * [Dirty pages](#dirty-pages)
        * [Notes](#notes-6)
    * [New format](#new-format)
      * [Base block](#base-block-2)
      * [Log entries](#log-entries)
        * [Notes](#notes-7)
  * [Dirty state of a hive](#dirty-state-of-a-hive)
  * [Multiple transaction log files](#multiple-transaction-log-files)
    * [Old format](#old-format-1)
    * [New format](#new-format-1)
  * [Additional sources of information](#additional-sources-of-information)

## Types of files
Registry hives consist of primary files, transaction log files, and backup copies of primary files. Primary files and their backup copies share the same format to hold an actual data making up a Windows registry, transaction log files are used to perform fault-tolerant writes to primary files.

### Examples of hives and supporting files

#### Windows 8.1: System hive
* Primary file:
  * \Windows\System32\config\SYSTEM
* Transaction log files:
  * \Windows\System32\config\SYSTEM.LOG
  * \Windows\System32\config\SYSTEM.LOG1
  * \Windows\System32\config\SYSTEM.LOG2

#### Windows 8.1: BCD hive
* Primary file:
  * \Boot\BCD
* Transaction log files:
  * \Boot\BCD.LOG
  * \Boot\BCD.LOG1
  * \Boot\BCD.LOG2

#### Windows Vista: System hive
* Primary file:
  * \Windows\System32\config\SYSTEM
* Transaction log files:
  * \Windows\System32\config\SYSTEM.LOG
  * \Windows\System32\config\SYSTEM.LOG1
  * \Windows\System32\config\SYSTEM.LOG2
* Backup copy of a primary file:
  * \Windows\System32\config\SYSTEM.SAV

#### Windows Vista: SAM hive
* Primary file:
  * \Windows\System32\config\SAM
* Transaction log files:
  * \Windows\System32\config\SAM.LOG
  * \Windows\System32\config\SAM.LOG1
  * \Windows\System32\config\SAM.LOG2

#### Windows XP: System hive
* Primary file:
  * \WINDOWS\system32\config\system
* Transaction log file:
  * \WINDOWS\system32\config\system.LOG
* Backup copy of a primary file:
  * \WINDOWS\system32\config\system.sav

## Transactional registry (TxR)
Transactional registry, introduced in Windows Vista, is a feature that allows a program to accumulate multiple modifications of a registry hive within a transaction, which can be committed or rolled back as a single unit (like a database transaction). Such a transaction is written to storage files of the Common Log File System (CLFS).

Transactional registry and CLFS are out of the scope of this document.

## Format of primary files
A primary file consists of a base block, also known as a file header, and a hive bins data. A hive bins data consists of hive bins. Each hive bin includes a header and cells, a cell is the actual storage of high-level registry entities (like keys, values, etc.). Primary files may contain padding of an arbitrary size after the end of the last hive bin.

![Primary file layout](https://raw.githubusercontent.com/msuhanov/regf/master/images/primary.png "Primary file layout")

### Base block
The base block is 4096 bytes in length, it contains the following structure in Windows XP (*hereinafter, all numbers and bit masks are in the little-endian form, all data units are bytes, unless otherwise mentioned*):

Offset|Length|Field|Value(s)|Description
---|---|---|---|---
0|4|Signature|regf|ASCII string
4|4|Primary sequence number||This number is incremented by 1 in the beginning of a write operation on the hive
8|4|Secondary sequence number||This number is incremented by 1 at the end of a write operation on the hive, primary sequence number and secondary sequence number should be equal after a successful write operation
12|8|Last written timestamp||FILETIME (UTC)
20|4|Major version|1|Major version of a hive writer
24|4|Minor version|3, 4, or 5|Minor version of a hive writer
28|4|File type|0|0 means *primary file*
32|4|File format|1|1 means *direct memory load*
36|4|Root cell offset||Offset of a root cell in bytes, relative from the start of the hive bins data
40|4|Hive bins data size||Size of the hive bins data in bytes
44|4|Clustering factor||Sector size of the underlying disk in bytes divided by 512
48|64|Filename||UTF-16LE string (contains a partial file path to the primary file, or a file name of the primary file), used for debugging purposes
112|396|Reserved||
508|4|Checksum||XOR-32 checksum of the previous 508 bytes
512|3576|Reserved||
4088|4|Boot type||This field has no meaning on a disk
4092|4|Boot recover||This field has no meaning on a disk

In Windows 10, the following fields are found to be allocated in the previously reserved areas:

Offset|Length|Field|Value|Description
---|---|---|---|---
112|16|RmId||GUID, see below
128|16|LogId||GUID, see below
144|4|Flags||Bit mask, see below
148|16|TmId||GUID, see below
164|4|GUID signature|rmtm|ASCII string
168|8|Last reorganized timestamp||FILETIME (UTC)
|||
4040|16|ThawTmId||GUID, this field has no meaning on a disk
4056|16|ThawRmId||GUID, this field has no meaning on a disk
4072|16|ThawLogId||GUID, this field has no meaning on a disk

The *RmId* and *LogId* fields contain a GUID of the Resource Manager (RM), and the *TmId* field contains a GUID used to generate a file name of a log file for the Transaction Manager (TM), see the *ZwCreateResourceManager()* and *ZwCreateTransactionManager()* routines. The *GUID signature* field is always set to the "rmtm" ASCII string.

The *Flags* field is used to record the state of the Kernel Transaction Manager (KTM), possible flags are:

Mask|Description
---|---
0x00000000|KTM released the hive
0x00000001|KTM locked the hive (there are pending transactions)

#### Notes
1. *File offset of a root cell = 4096 + Root cell offset*. This formula also applies to any other offset relative to the start of the hive bins data.
2. The XOR-32 checksum is calculated using the following algorithm:
   * let C be a 32-bit value, initial value is zero;
   * let D be a data containing 508 bytes;
   * split D into non-overlapping groups of 32 bits, and for each group, G[i], do the following: C = C xor G[i];
   * C is the checksum.
3. The *Boot type* and *Boot recover* fields are used for in-memory hive recovery management by a boot loader and a kernel, they are not written to a disk in most cases (when *Clustering factor* is 8, these fields may be written to a disk, but they have no meaning there).
4. New fields, except the *Last reorganized timestamp*, were introduced in Windows Vista as a part of the CLFS. The *Last reorganized timestamp* was introduced in Windows 8 and Windows Server 2012.
5. The *Last reorganized timestamp* field contains a timestamp of the latest hive defragmentation (it happens once a week when a hive isn't locked).
6. The *ThawTmId*, *ThawRmId*, and *ThawLogId* fields are used to restore the state of the *TmId*, *RmId*, and *LogId* fields respectively when thawing a hive (after it was frozen in order to create a shadow copy).

### Hive bin
The *hive bin* is variable in size and consists of a header and cells. A hive bin's header is 32 bytes in length, it contains the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|4|Signature|hbin|ASCII string
4|4|Offset||Offset of a current hive bin in bytes, relative from the start of the hive bins data
8|4|Size||Size of a current hive bin in bytes
12|8|Reserved||
20|8|Timestamp||FILETIME (UTC), defined for the first hive bin only (see below)
28|4|Spare||This field has no meaning on a disk

#### Notes
1. A hive bin's size is multiple of 4096 bytes.
2. The *Spare* field is used when shifting hive bins and cells in-memory.
3. A *Timestamp* in the header of the first hive bin acts as a backup copy of a *Last written timestamp* in the base block.

### Cell
*Cells* fill the remaining space of a hive bin, each cell is variable in size and has the following structure:

Offset|Length|Field|Description
---|---|---|---
0|4|Size|Size of a current cell in bytes, including this field (aligned to 8 bytes): the size is positive if a cell is unallocated or negative if a cell is allocated (use absolute values for calculations)
4|...|Cell data|

*Cell data* may contain one of the following records:

Record|Description
---|---
Index leaf (li)|Subkeys list
Fast leaf (lf)|Subkeys list with name hints
Hash leaf (lh)|Subkeys list with name hashes
Index root (ri)|List of subkeys lists (used to subdivide subkeys lists)
Key node (nk)|Registry key node
Key value (vk)|Registry key value
Key security (sk)|Security descriptor item
Big data (db)|List of data segments

Also, *Cell data* may contain a raw data (e.g. *Key value* data) and any lists defined below. A padding may be present at the end of a cell.

#### Index leaf
The *Index leaf* has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|2|Signature|li|ASCII string
2|2|Number of elements||
4|...|List elements||

A list element has the following structure (a list element is repeated *Number of elements* times):

Offset|Length|Field|Description
---|---|---|---
0|4|Key node offset|In bytes, relative from the start of the hive bins data

All list elements are required to be sorted ascending by an uppercase version of a key name string (comparison should be based on character codes).

#### Fast leaf
The *Fast leaf* has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|2|Signature|lf|ASCII string
2|2|Number of elements||
4|...|List elements||

A list element has the following structure:

Offset|Length|Field|Description
---|---|---|---
0|4|Key node offset|In bytes, relative from the start of the hive bins data
4|4|Name hint|The first 4 ASCII characters of a key name string (used to speed up lookups)

If a key name string is less than 4 characters in length, it is stored in the beginning of the *Name hint* field, unused bytes of this field are set to null. UTF-16LE characters are converted to ASCII, if possible (if it isn't, the first byte of the *Name hint* field is null).

All list elements are required to be sorted (as described above).

#### Hash leaf
The *Hash leaf* has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|2|Signature|lh|ASCII string
2|2|Number of elements||
4|...|List elements||

A list element has the following structure:

Offset|Length|Field|Description
---|---|---|---
0|4|Key node offset|In bytes, relative from the start of the hive bins data
4|4|Hash|Hash of a key name string, see below (used to speed up lookups)

All list elements are required to be sorted (as described above).

##### Note
The hash is calculated using the following algorithm:
   * let H be a 32-bit value, initial value is zero;
   * let N be an *uppercase* name of a key;
   * split N into characters, and for each character (two bytes in the case of UTF-16LE), treated as a number (character code), C[i], do the following: H = 37 * H + C[i];
   * H is the hash value.

#### Index root
The *Index root* has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|2|Signature|ri|ASCII string
2|2|Number of elements||
4|...|List elements||

A list element has the following structure:

Offset|Length|Field|Description
---|---|---|---
0|4|Subkeys list offset|In bytes, relative from the start of the hive bins data

##### Notes
1. An *Index root* can't point to another *Index root*.
2. A *Subkeys list* can't point to an *Index root*.
3. List elements within subkeys lists referenced by a single *Index root* must be sorted as a whole (i.e. the first list element of the second subkeys list must be greater than the last list element of the first subkeys list).

#### Key node
The *Key node* has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|2|Signature|nk|ASCII string
2|2|Flags||Bit mask, see below
4|8|Last written timestamp||FILETIME (UTC)
12|4|Spare||Probably not used
16|4|Parent||Offset of a parent key node in bytes, relative from the start of the hive bins data (this field has no meaning on a disk for a root key node)
20|4|Number of subkeys||
24|4|Number of volatile subkeys||
28|4|Subkeys list offset||In bytes, relative from the start of the hive bins data
32|4|Volatile subkeys list offset||This field has no meaning on a disk (volatile keys are not written to a file)
36|4|Number of key values||
40|4|Key values list offset||In bytes, relative from the start of the hive bins data
44|4|Key security offset||In bytes, relative from the start of the hive bins data
48|4|Class name offset||In bytes, relative from the start of the hive bins data
52|4|Largest subkey name length||In bytes, a subkey name is treated as a UTF-16LE string (see below)
56|4|Largest subkey class name length||In bytes
60|4|Largest value name length||In bytes, a value name is treated as a UTF-16LE string
64|4|Largest value data size||In bytes
68|4|WorkVar||The meaning of this field is unknown
72|2|Key name length||In bytes
74|2|Class name length||In bytes
76|...|Key name string||ASCII string or UTF-16LE string

Starting from Windows Vista, Windows Server 2003 SP2, and Windows XP SP3, the *Largest subkey name length* field has been split into 4 fields:

Offset (bits)|Length (bits)|Field|Description
---|---|---|---
0|16|Largest subkey name length|
16|4|Virtualization control flags|Bit mask, see below
20|4|User flags (Wow64 flags)|Bit mask
24|8|Debug|See below

**Warning**

When implementing the structure defined above in a program, keep in mind that a compiler may pack the *Virtualization control flags* and *User flags* bit fields in a different way. In C, two or more bit fields inside an integer may be packed right-to-left, so the first bit field defined in an integer may reside in the less significant (right) bits. In debug symbols for Windows, the *UserFlags* field is defined before the *VirtControlFlags* field exactly for this reason (however, these fields are written to a file in the order indicated in the table above).

##### Flags
In Windows XP and Windows Server 2003, the first 4 bits are reserved for user flags (set via the *NtSetInformationKey()* call, read via the *NtQueryKey()* call). Other bits have the following meaning:

Mask|Name|Description
---|---|---
0x0001|KEY_VOLATILE|Is volatile
0x0002|KEY_HIVE_EXIT|Is the mount point of another hive
0x0004|KEY_HIVE_ENTRY|Is the root key for this hive
0x0008|KEY_NO_DELETE|This key can't be deleted
0x0010|KEY_SYM_LINK|This key is a symlink (a target key is specified as a string (REG_LINK) in a value named "SymbolicLinkValue")
0x0020|KEY_COMP_NAME|Name is an ASCII string (otherwise it is a UTF-16LE string)
0x0040|KEY_PREDEF_HANDLE|Is a predefined handle (a handle is stored in the *Number of key values* field)

The following bits are also used as of Windows Vista:

Mask|Name|Description
---|---|---
0x0080|VirtualSource|This key was virtualized at least once
0x0100|VirtualTarget|Is virtual
0x0200|VirtualStore|Is a part of a virtual store path

It is plausible that a registry key virtualization (when registry writes to sensitive locations are redirected to per-user locations in order to protect a Windows registry against corruption) required more space than 4 bits in the beginning of this field can provide, that is why the *Largest subkey name length* field was split and the new fields were introduced.

Starting from Windows Vista, user flags were moved away from the first 4 bits of the *Flags* field to the new *User flags* bit field (see above). These user flags in the new location are also called *Wow64 flags*. In Windows XP and Windows Server 2003, user flags are stored in the old location anyway.

It is unclear whether the first 4 bits of the *Flags* field are still reserved or not in recent versions of Windows. For example, in Windows Vista and Windows 7, the 4th bit of the *Flags* field is set to 1 in many key nodes belonging to different hives; however, this bit can't be read through the *NtQueryKey()* call.

##### Virtualization control flags
The *Virtualization control flags* field is set according to the following bit masks:

Mask|Name|Description
---|---|---
0x2|REG_KEY_DONT_VIRTUALIZE|Disable registry write virtualization
0x4|REG_KEY_DONT_SILENT_FAIL|Disable registry open virtualization
0x8|REG_KEY_RECURSE_FLAG|Propagate virtualization flags to new child keys

##### Debug
When the *Debug* field of a key node and the *CmpRegDebugBreakEnabled* kernel variable are set to 1, a checked Windows kernel will execute the *int 3* instruction on an access to the registry key. A retail Windows kernel has this feature disabled.

#### Key values list
The *Key values list* has the following structure:

Offset|Length|Field
---|---|---
0|...|List elements

A list element has the following structure:

Offset|Length|Field|Description
---|---|---|---
0|4|Key value offset|In bytes, relative from the start of the hive bins data

#### Key value
The *Key value* has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|2|Signature|vk|ASCII string
2|2|Name length||In bytes, can be 0 (name isn't set)
4|4|Data size||In bytes, can be 0 (value isn't set), the most significant bit has a special meaning (see below)
8|4|Data offset||In bytes, relative from the start of the hive bins data (or a data itself, see below)
12|4|Data type||See below
16|2|Flags||Bit mask, see below
18|2|Spare||Probably not used
20|...|Name||ASCII string or UTF-16LE string

##### Data size
When the most significant bit is 1, a data (4 bytes or less) is stored in the *Data offset* field directly (when a data contains less than 4 bytes, it is being stored as is in the beginning of the *Data offset* field). The most significant bit (when set to 1) should be ignored when calculating the data size.

When the most significant bit is 0, a data is stored in the *Cell data* field of another cell (pointed by the *Data offset* field) or in the *Cell data* fields of multiple cells (referenced in the *Big data* structure stored in a cell pointed by the *Data offset* field).

##### Data types

Value|Name(s)
---|---
0x00000000|REG_NONE
0x00000001|REG_SZ
0x00000002|REG_EXPAND_SZ
0x00000003|REG_BINARY
0x00000004|REG_DWORD, REG_DWORD_LITTLE_ENDIAN
0x00000005|REG_DWORD_BIG_ENDIAN
0x00000006|REG_LINK
0x00000007|REG_MULTI_SZ
0x00000008|REG_RESOURCE_LIST
0x00000009|REG_FULL_RESOURCE_DESCRIPTOR
0x0000000a|REG_RESOURCE_REQUIREMENTS_LIST
0x0000000b|REG_QWORD, REG_QWORD_LITTLE_ENDIAN

##### Flags

Mask|Description
---|---
0x0001|Name is an ASCII string (otherwise it is a UTF-16LE string)

#### Key security
The *Key security* item has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|2|Signature|sk|ASCII string
2|2|Reserved||
4|4|Flink||See below
8|4|Blink||See below
12|4|Reference count||
16|4|Security descriptor size||In bytes
20|...|Security descriptor||

##### Notes
1. Flink and blink are offsets in bytes, relative from the start of the hive bins data.
2. Key security items form a doubly linked list. A key security item may act as a list header or a list entry (the only difference here is the meaning of flink and blink fields).
4. When a key security item acts as a list header, flink and blink point to the first and the last entries of this list respectively. If a list is empty, flink and blink point to a list header (i.e. to a current cell).
5. When a key security item acts as a list entry, flink and blink point to the next and the previous entries of this list respectively. If there is no next entry in a list, flink points to a list header. If there is no previous entry in a list, blink points to a list header.

#### Big data
The *Big data* is used to reference a data larger than 16344 bytes (when the *Minor version* field of the base block is greater than 3), it has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|2|Signature|db|ASCII string
2|2|Number of segments||
4|4|Offset of a list of segments||In bytes, relative from the start of the hive bins data

The list of segments has the following structure:

Offset|Length|Field
---|---|---
0|...|List elements

A list element has the following structure:

Offset|Length|Field|Description
---|---|---|---
0|4|Data segment offset|In bytes, relative from the start of the hive bins data

##### Data segment
A *data segment* is stored in the *Cell data* field of a cell pointed by the *Data segment offset* field. A data segment has the maximum size of 16344 bytes.

Data segments of a *Big data* record, except the last one, have the maximum size.

### Summary
1. A *Base block* points to a root cell, which contains a *Key node*.
2. A *Key node* points to a parent *Key node*, to a *Subkeys list* (a subkey is a *Key node* too), to a *Key values list*, to a *Key security* item.
3. A *Subkeys list* can be subdivided with the help of the *Index root* structure.
4. A *Key value* points to a data. A data may be stored in the *Data offset* field of a *Key value* structure, in a separate cell, or in a bunch of cells. In the latter case, a *Key value* points to the *Big data* structure in a cell.

## Format of transaction log files

### Old format
A transaction log file (old format) consists of a base block, a dirty vector, and dirty pages.

![Transaction log file layout (old format)](https://raw.githubusercontent.com/msuhanov/regf/master/images/old-log.png "Transaction log file layout (old format)")

#### Base block
A partial backup copy of a base block is stored in the first sector of a transaction log file, only the first *Clustering factor * 512* bytes of a base block are written there.

A backup copy of a base block isn't an exact copy anyway, the following modifications are performed on it by a hive writer:

1. *File type* field is set to 1 (1 means *transaction log*).
2. *Primary sequence number* is incremented by 1 before writing a log of dirty data.
3. *Secondary sequence number* is incremented by 1 after a log of dirty data was written.
4. *Checksum* is recalculated with the modified data.

##### Notes
1. A partial backup copy of a base block is made using a data from memory, not from a primary file.
2. A transaction log file is considered to be valid when it has an expected base block (including the modifications mentioned above), and its primary sequence number is equal to its secondary sequence number.
3. A transaction log can be applied when a *Last written timestamp* in its base block is equal to a *Last written timestamp* in a base block of a primary file (when a base block of a primary file is invalid, a *Timestamp* from the first hive bin is used instead).
4. If a base block of a primary file has a wrong *Checksum*, it is being recovered using a base block from a transaction log file (and the *File type* field is set back to 0).
5. In Windows XP SP2, the same memory region is used to write a base block both to a transaction log file and to a primary file, that is why, after a successful write operation, a base block of a primary file will contain two sequence numbers equal to *N*, and a base block of a transaction log file will contain two sequence numbers equal to *N - 1* (sequence numbers are incremented by 1 for a transaction log file first, and then they are incremented by 1 again for a primary file). In Windows Vista SP2, a copy of a base block is used when writing to a transaction log file, that is why a transaction log file and a primary file will contain the same sequence numbers in their base blocks after a successful write operation.

#### Dirty vector
The *Dirty vector* is stored starting from the beginning of the second sector of a transaction log file, it has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|4|Signature|DIRT|ASCII string
4|...|Bitmap||Bitmap of dirty pages

Each bit of a bitmap corresponds to the state of a specific *512-byte* page within a hive bins data to be written to a primary file from memory, regardless of a sector size of an underlying disk (these pages don't overlap, there are no gaps between them):
* the first bit, bit #1, corresponds to the state of the first *512-byte* page within a hive bins data;
* bit #2 corresponds to the state of the second *512-byte* page within a hive bins data, etc.

The meaning of each bit in a bitmap is the following:

Bit|Description
---|---
0|A corresponding page is clean
1|A corresponding page is dirty

Bitmap length (*in bits*) is calculated using the following formula: *Bitmap length = Hive bins data size / 512*.

##### Notes
1. The state of a base block isn't recorded in a dirty vector.
2. Dirty vector is stored in-memory as a *RTL_BITMAP* structure. However, only the *Buffer* field of this structure is written to a transaction log file.
3. Windows tracks changes to a hive bins data in 4096-byte blocks. This means that a dirty vector's bitmap is *expected* to contain only the following *bytes*: 0x00 and 0xFF.
4. A padding is likely to be present after the end of a bitmap (up to a sector boundary).

#### Dirty pages
*Dirty pages* are stored starting from the beginning of the sector following the last sector of a dirty vector. Each dirty page is stored at an offset divisible by 512 bytes and has a length of 512 bytes.

The first dirty page corresponds to the first bit set to 1 in the bitmap of a dirty vector, the second dirty page corresponds to the second bit set to 1 in the bitmap of a dirty vector, etc. During recovery, contiguous dirty pages belonging to the same hive bin in a primary file are processed together, and a dirty hive bin is verified for correctness (its *Signature* must be correct, its *Size* must not be less than 4096 bytes, its *Offset* must match the *Offset* of a corresponding hive bin in a primary file); recovery stops if a dirty hive bin is invalid, an invalid dirty hive bin is ignored.

##### Notes
1. The number of dirty pages is equal to the number of bits set to 1 in the bitmap of a dirty vector. Remnant dirty pages may be present after the end of the last dirty page.
2. A dirty page will be written to a primary file at the following offset: *File offset = 4096 + 512 * Bit position*, where *Bit position* is the index of a corresponding bit in the bitmap of a dirty vector.

### New format
A transaction log file (new format) consists of a base block and log entries. This format was introduced in Windows 8.1 and Windows Server 2012 R2.

![Transaction log file layout (new format)](https://raw.githubusercontent.com/msuhanov/regf/master/images/new-log.png "Transaction log file layout (new format)")

#### Base block
A modified partial backup copy of a base block is stored in the first sector of a transaction log file in the same way as in the old format and for the same purpose. However, the *File type* field is set to 6.

#### Log entries
*Log entries* are stored starting from the beginning of the second sector. Each log entry is stored at an offset divisible by 512 bytes and has a variable size.

A log entry has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|4|Signature|HvLE|ASCII string
4|4|Size||Size of a current log entry in bytes
8|4|Flags||Copy of the *Flags* field of the base block at the time of creation of a current log entry (see below)
12|4|Sequence number||This number will be written both to the primary sequence number and to the secondary sequence number of the base block when applying a current log entry
16|4|Hive bins data size||Copy of the *Hive bins data size* field of the base block at the time of creation of a current log entry
20|4|Dirty pages count||Number of dirty pages attached to a log entry
24|8|Hash-1||See below
32|8|Hash-2||See below
40|...|Dirty pages references||
...|...|Dirty pages||

A dirty page reference describes a single page to be written to a primary file, and it has the following structure:

Offset|Length|Field|Description
---|---|---|---
0|4|Offset|Offset of a page in a primary file, relative from the start of the hive bins data
4|4|Size|Size of a page in bytes

*Dirty pages* are attached to a log entry in the same order as in the *Dirty pages references* without an alignment or gaps.

##### Notes
1. *Hash-1* is the Marvin32 hash of the data starting from the beginning of the first page reference of a current log entry with the length of *Size - 40* bytes.
2. *Hash-2* is the Marvin32 hash of the first 32 bytes of a current log entry (including the *Hash-1* calculated before).
3. The following seed is used for calculating the *Hash-1* and *Hash-2* (hexadecimal bytes): 82 EF 4D 88 7A 4E 55 C5.
4. A transaction log file may contain multiple log entries written at once, as well as old (already applied) log entries.
5. If a primary file is dirty and has a valid *Checksum* (in the base block), only subsequent log entries are applied. A subsequent log entry is a log entry with a sequence number equal to or greater than a secondary sequence number of the base block in a primary file.
6. If a primary file is dirty and has a wrong *Checksum*, its base block is recovered from a transaction log file. Then subsequent log entries are applied.
7. If a log entry with a sequence number *N* is *not* followed by a log entry with a sequence number *N + 1*, recovery stops after applying a log entry with a sequence number *N*. If the first log entry doesn't contain an expected sequence number (equal to a secondary sequence number of the base block in a primary file), recovery stops.
8. If a log entry has a wrong value in the field *Hash-1*, *Hash-2*, or *Hive bins data size* (i.e. it is not multiple of 4096 bytes), recovery stops, only previous log entries (preceding a bogus one) are applied.
9. A primary file is grown according to the *Hive bins data size* field of a log entry being applied.
10. Dirty hive bins are verified for correctness during recovery (but recovery doesn't stop on a bad hive bin, a bad hive bin is healed instead).
11. The *Flags* field of a log entry is set to a value of the *Flags* field of the base block. During recovery, the *Flags* field of the base block is set to a value taken from a log entry being applied.

## Dirty state of a hive
A hive is considered to be dirty when a base block in a primary file contains a wrong checksum, or its primary sequence number doesn't match its secondary sequence number. If a hive isn't dirty, but a transaction log file (new format) contains subsequent log entries, they are ignored.

## Multiple transaction log files
A primary file may have either one transaction log file (\*.LOG) or two transaction log files (\*.LOG1 and \*.LOG2). In the latter case, an empty third file (\*.LOG) may be present for backward compatibility.

### Old format
If multiple transaction log files are present and a hive is dirty, an applicable one (i.e. having the same *Last written timestamp* in a base block as in a base block of a primary file, see above) is used to recover a hive.

A switch to a new transaction log file may happen after an error occurs when applying a transaction log to a primary file.

### New format
If multiple transaction log files are present and a hive is dirty, the first transaction log file (\*.LOG1) and then the second transaction log file (\*.LOG2) are used to recover a hive.

A switch to a new transaction log file may be used to split log entries.

## Additional sources of information
1. http://www.sentinelchicken.com/data/TheWindowsNTRegistryFileFormat.pdf
2. https://github.com/libyal/libregf/blob/master/documentation/Windows%20NT%20Registry%20File%20(REGF)%20format.asciidoc
3. http://amnesia.gtisc.gatech.edu/~moyix/suzibandit.ltd.uk/MSc/

-------
© 2015 Maxim Suhanov
