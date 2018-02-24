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
        * [Access bits](#access-bits)
        * [Class name](#class-name)
        * [WorkVar](#workvar)
        * [Flags](#flags)
        * [User flags](#user-flags)
        * [Virtualization control flags](#virtualization-control-flags)
        * [Debug](#debug)
        * [Layered keys](#layered-keys)
      * [Key values list](#key-values-list)
      * [Key value](#key-value)
        * [Data size](#data-size)
        * [Data types](#data-types)
        * [Flags](#flags-1)
      * [Key security](#key-security)
        * [Notes](#notes-3)
      * [Big data](#big-data)
        * [Data segment](#data-segment)
    * [Unallocated cell](#unallocated-cell)
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
  * [Flush strategies](#flush-strategies)
  * [Sector size and clustering factor](#sector-size-and-clustering-factor)
  * [Very old versions of Windows](#very-old-versions-of-windows)
  * [Additional sources of information](#additional-sources-of-information)

## Types of files
Stable registry hives consist of primary files, transaction log files, and backup copies of primary files. Primary files and their backup copies share the same format to hold actual data making up a Windows registry, transaction log files are used to perform fault-tolerant writes to primary files. Before writing modified (dirty) data to a primary file, a hive writer will store this data in a transaction log file. If an error (like a system crash) occurs when writing to a transaction log file, a primary file will remain consistent; if an error occurs when writing to a primary file, a transaction log file will contain enough data to recover a primary file and bring it back to the consistent state.

### Examples of hives and supporting files

#### Windows 8.1: System hive
* Primary file:
  * \Windows\System32\config\SYSTEM
* Transaction log files:
  * \Windows\System32\config\SYSTEM.LOG
  * \Windows\System32\config\SYSTEM.LOG1
  * \Windows\System32\config\SYSTEM.LOG2
* Backup copy of a primary file:
  * \Windows\System32\config\RegBack\SYSTEM

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
* Backup copies of a primary file:
  * \Windows\System32\config\SYSTEM.SAV
  * \Windows\System32\config\RegBack\SYSTEM
  * \Windows\System32\config\RegBack\SYSTEM.OLD

#### Windows Vista: SAM hive
* Primary file:
  * \Windows\System32\config\SAM
* Transaction log files:
  * \Windows\System32\config\SAM.LOG
  * \Windows\System32\config\SAM.LOG1
  * \Windows\System32\config\SAM.LOG2
* Backup copies of a primary file:
  * \Windows\System32\config\RegBack\SAM
  * \Windows\System32\config\RegBack\SAM.OLD

#### Windows XP: System hive
* Primary file:
  * \WINDOWS\system32\config\system
* Transaction log file:
  * \WINDOWS\system32\config\system.LOG
* Backup copy of a primary file:
  * \WINDOWS\system32\config\system.sav

## Transactional registry (TxR)
Transactional registry, introduced in Windows Vista, is a feature that allows a program to accumulate multiple modifications of a registry hive within a transaction, which can be committed or rolled back as a single unit (like a database transaction). Such a transaction is written to storage files of the Common Log File System (CLFS).

Transactional registry and storage files of the CLFS are out of the scope of this document.

## Format of primary files
A primary file consists of a base block, also known as a file header, and hive bins data. Hive bins data consists of hive bins. Each hive bin includes a header and cells, cells are the actual storage of high-level registry entities (like keys, values, etc.). Primary files may contain a padding or remnant data of an arbitrary size after the end of the last hive bin.

![Primary file layout](https://raw.githubusercontent.com/msuhanov/regf/master/images/primary.png "Primary file layout")

### Base block
The base block is 4096 bytes in length, it contains the following structure in Windows XP (*hereinafter, all numbers and bit masks are in the little-endian form, all data units are bytes, unless otherwise mentioned*):

Offset|Length|Field|Value(s)|Description
---|---|---|---|---
0|4|Signature|regf|ASCII string
4|4|Primary sequence number| |This number is incremented by 1 in the beginning of a write operation on the primary file
8|4|Secondary sequence number| |This number is incremented by 1 at the end of a write operation on the primary file, a *primary sequence number* and a *secondary sequence number* should be equal after a successful write operation
12|8|Last written timestamp| |FILETIME (UTC)
20|4|Major version|1|Major version of a hive writer
24|4|Minor version|3, 4, 5, or 6|Minor version of a hive writer
28|4|File type|0|0 means *primary file*
32|4|File format|1|1 means *direct memory load*
36|4|Root cell offset| |Offset of a root cell in bytes, relative from the start of the hive bins data
40|4|Hive bins data size| |Size of the hive bins data in bytes
44|4|Clustering factor| |Logical sector size of the underlying disk in bytes divided by 512
48|64|File name| |UTF-16LE string (contains a partial file path to the primary file, or a file name of the primary file), used for debugging purposes
112|396|Reserved| |
508|4|Checksum| |XOR-32 checksum of the previous 508 bytes
512|3576|Reserved| |
4088|4|Boot type| |This field has no meaning on a disk
4092|4|Boot recover| |This field has no meaning on a disk

In Windows 10, the following fields are found to be allocated in the previously reserved areas:

Offset|Length|Field|Value|Description
---|---|---|---|---
112|16|RmId| |GUID, see below
128|16|LogId| |GUID, see below
144|4|Flags| |Bit mask, see below
148|16|TmId| |GUID, see below
164|4|GUID signature|rmtm|ASCII string
168|8|Last reorganized timestamp| |FILETIME (UTC), see below
...|...| | |
4040|16|ThawTmId| |GUID, this field has no meaning on a disk
4056|16|ThawRmId| |GUID, this field has no meaning on a disk
4072|16|ThawLogId| |GUID, this field has no meaning on a disk

The *RmId* field contains a GUID of the Resource Manager (RM), and the *TmId* field contains a GUID used to generate a file name of a log file stream for the Transaction Manager (TM), see the *ZwCreateResourceManager()* and *ZwCreateTransactionManager()* routines; the *LogId* field contains a GUID used to generate a file name of a physical log file, see the *ClfsCreateLogFile()* routine. The *GUID signature* field is always set to the "rmtm" ASCII string.

The *Flags* field is used to record the state of the Kernel Transaction Manager (KTM) and the hive, possible flags are:

Mask|Description
---|---
0x00000001|KTM locked the hive (there are pending or anticipated transactions)
0x00000002|The hive has been defragmented (all its pages are dirty therefore) and it is being written to a disk (Windows 8 and Windows Server 2012 only, this flag is used to speed up hive recovery by reading a transaction log file instead of a primary file); this hive supports the layered keys feature (starting from Insider Preview builds of Windows 10 "Redstone 1")

#### Notes
1. *File offset of a root cell = 4096 + Root cell offset*. This formula also applies to any other offset relative from the start of the hive bins data (however, if such a relative offset is equal to 0xFFFFFFFF, it doesn't point anywhere).
2. The XOR-32 checksum is calculated using the following algorithm:
   * let C be a 32-bit value, initial value is zero;
   * let D be data containing 508 bytes;
   * split D into non-overlapping groups of 32 bits, and for each group, G[i], do the following: C = C xor G[i];
   * if C is equal to -1, set C to -2;
   * if C is equal to 0, set C to 1;
   * C is the checksum.
3. The *Boot type* and *Boot recover* fields are used for in-memory hive recovery management by a boot loader and a kernel, they are not written to a disk in most cases (when *Clustering factor* is 8, these fields may be written to a disk, but they have no meaning there).
4. New fields, except the *Last reorganized timestamp*, were introduced in Windows Vista as a part of the CLFS. The *Last reorganized timestamp* field was introduced in Windows 8 and Windows Server 2012.
5. The *Last reorganized timestamp* field contains a timestamp of the latest hive reorganization (it happens once a week when a hive isn't locked). The reorganization process may either defragment the hive on a disk by storing allocated data compactly in a primary file or clear the access history of key nodes in the hive (see the *Access bits* field of a key node). The defragmentation process implies clearing the access history of key nodes in the hive. The first 2 bits of the *Last reorganized timestamp* field, counting from the least significant bit, have a special meaning: when the first bit is set to 1, the hive was defragmented during the latest reorganization; when the second bit is set to 1, the access history of key nodes in the hive was cleared during the latest reorganization. When the *Last reorganized timestamp* field is equal to 1, the next reorganization will defragment the hive; when the *Last reorganized timestamp* field is equal to 2, the next reorganization will clear the access history of key nodes in the hive (these are special values to enforce the specific operation).
6. The *LogId* field usually contains the same value as the *RmId* field.
7. When the *RmId* field is null, the *LogId* and *TmId* fields may contain garbage data.
8. The *ThawTmId*, *ThawRmId*, and *ThawLogId* fields are used to restore the state of the *TmId*, *RmId*, and *LogId* fields respectively when thawing a hive (after it was frozen in order to create a shadow copy).
9. The *Last written timestamp* field isn't updated as of Windows 8.1 and Windows Server 2012 R2 (but it can be set when a hive is created).

### Hive bin
The *Hive bin* is variable in size and consists of a header and cells. A hive bin's header is 32 bytes in length, it contains the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|4|Signature|hbin|ASCII string
4|4|Offset| |Offset of a current hive bin in bytes, relative from the start of the hive bins data
8|4|Size| |Size of a current hive bin in bytes
12|8|Reserved| |
20|8|Timestamp| |FILETIME (UTC), defined for the first hive bin only (see below)
28|4|Spare (*or* MemAlloc)| |This field has no meaning on a disk (see below)

#### Notes
1. A hive bin size is multiple of 4096 bytes.
2. The *Spare* field is used when shifting hive bins and cells in memory. In Windows 2000, the same field is called *MemAlloc*, it is used to track memory allocations for hive bins.
3. A *Timestamp* in the header of the first hive bin acts as a backup copy of a *Last written timestamp* in the base block.

### Cell
*Cells* fill the remaining space of a hive bin (without gaps between them), each cell is variable in size and has the following structure:

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
Key security (sk)|Security descriptor
Big data (db)|List of data segments

Also, *Cell data* may contain raw data (e.g. *Key value* data) and any list defined below. A padding may be present at the end of a cell.

When a record (entity) contains an offset field pointing to another record (entity), this offset points to a cell containing the latter record (entity). As already mentioned above, an offset relative from the start of the hive bins data doesn't point anywhere when it is equal to 0xFFFFFFFF.

#### Index leaf
The *Index leaf* has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|2|Signature|li|ASCII string
2|2|Number of elements| |
4|...|List elements| |

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
2|2|Number of elements| |
4|...|List elements| |

A list element has the following structure:

Offset|Length|Field|Description
---|---|---|---
0|4|Key node offset|In bytes, relative from the start of the hive bins data
4|4|Name hint|The first 4 ASCII characters of a key name string (used to speed up lookups)

If a key name string is less than 4 characters in length, it is stored in the beginning of the *Name hint* field (hereinafter, *the beginning of the field* means *the byte at the lowest address or the first few bytes at lower addresses in the field*), unused bytes of this field are set to null. UTF-16LE characters are converted to ASCII (extended), if possible (if it isn't, the first byte of the *Name hint* field is null).

Hereinafter, an extended ASCII string means a string made from UTF-16LE characters with codes less than 256 by removing the null byte (at the highest address) from each character.

All list elements are required to be sorted (as described above).

#### Hash leaf
The *Hash leaf* has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|2|Signature|lh|ASCII string
2|2|Number of elements| |
4|...|List elements| |

A list element has the following structure:

Offset|Length|Field|Description
---|---|---|---
0|4|Key node offset|In bytes, relative from the start of the hive bins data
4|4|Name hash|Hash of a key name string, see below (used to speed up lookups)

All list elements are required to be sorted (as described above). The *Hash leaf* is used when the *Minor version* field of the base block is greater than 4.

##### Note
The hash is calculated using the following algorithm:
   * let H be a 32-bit value, initial value is zero;
   * let N be an *uppercase* name of a key;
   * split N into characters, and for each character (exactly two bytes in the case of UTF-16LE, also known as a *wide character*), treated as a number (character code), C[i], do the following: H = 37 * H + C[i];
   * H is the hash value.

#### Index root
The *Index root* has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|2|Signature|ri|ASCII string
2|2|Number of elements| |
4|...|List elements| |

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
2|2|Flags| |Bit mask, see below
4|8|Last written timestamp| |FILETIME (UTC)
12|4|Access bits| |Bit mask, see below (this field is used as of Windows 8 and Windows Server 2012; in previous versions of Windows, this field is reserved and called *Spare*)
16|4|Parent| |Offset of a parent key node in bytes, relative from the start of the hive bins data (this field has no meaning on a disk for a root key node)
20|4|Number of subkeys| |
24|4|Number of volatile subkeys| |
28|4|Subkeys list offset| |In bytes, relative from the start of the hive bins data (also, this field may point to an *Index root*)
32|4|Volatile subkeys list offset| |This field has no meaning on a disk (volatile keys are not written to a file)
36|4|Number of key values| |
40|4|Key values list offset| |In bytes, relative from the start of the hive bins data
44|4|Key security offset| |In bytes, relative from the start of the hive bins data
48|4|Class name offset| |In bytes, relative from the start of the hive bins data
52|4|Largest subkey name length| |In bytes, a subkey name is treated as a UTF-16LE string (see below)
56|4|Largest subkey class name length| |In bytes
60|4|Largest value name length| |In bytes, a value name is treated as a UTF-16LE string
64|4|Largest value data size| |In bytes
68|4|WorkVar| |Cached index (see below)
72|2|Key name length| |In bytes
74|2|Class name length| |In bytes
76|...|Key name string| |ASCII (extended) string or UTF-16LE string

Starting from Windows Vista, Windows Server 2003 SP2, and Windows XP SP3, the *Largest subkey name length* field has been split into 4 bit fields (the offsets below are relative from the beginning of the old *Largest subkey name length* field, i.e. the first bit field starts within the byte at the lowest address):

Offset (bits)|Length (bits)|Field|Description
---|---|---|---
0|16|Largest subkey name length|
16|4|Virtualization control flags|Bit mask, see below
20|4|User flags (Wow64 flags)|Bit mask, see below
24|8|Debug|See below

**Warning**

When implementing the structure defined above in a program, keep in mind that a compiler may pack the *Virtualization control flags* and *User flags* bit fields in a different way. In C, two or more bit fields inside an integer may be packed right-to-left, so the first bit field defined in an integer may reside in the less significant (right) bits. In debug symbols for Windows, the *UserFlags* field is defined before the *VirtControlFlags* field exactly for this reason (however, these fields are written to a file in the order indicated in the table above).

**End of warning**

Starting from Insider Preview builds of Windows 10 "Redstone 1", the *Access bits* field has been split into the following fields:

Offset|Length|Field|Description
---|---|---|---
0|1|Access bits|
1|1|Layered key bit fields|See below
2|2|Spare|Not used

The *Layered key bit fields* have the following structure (the offsets below are relative from the most significant bit):

Offset (bits)|Length (bits)|Field|Value(s)|Description
---|---|---|---|---
0|1|Inherit class|0 or 1|See below
1|5|Spare| |Not used
6|2|Layer semantics|0, 1, 2, or 3|See below

##### Access bits
Access bits are used to track when key nodes are being accessed (e.g. via the *RegCreateKeyEx()* and *RegOpenKeyEx()* calls), the field is set according to the following bit masks:

Mask|Description
---|---
0x1|This key was accessed before a Windows registry was initialized with the *NtInitializeRegistry()* routine during the boot
0x2|This key was accessed after a Windows registry was initialized with the *NtInitializeRegistry()* routine during the boot

When the *Access bits* field is equal to 0, the access history of a key is clear.

##### Class name
Typically, a class name is a UTF-16LE string assigned to a key node; it doesn't have any particular meaning. For example, Microsoft was using class names to store keys for the Syskey encryption. Originally, class names were intended to be used to define key node types (similar to data types of the *Key value*).

##### WorkVar
In Windows 2000, the *WorkVar* field may contain a zero-based index of a subkey in a subkeys list found at the last successful lookup (used to speed up lookups when the same subkey is accessed many times consecutively: a list element with the cached index is always tried first); such a cached index may point to a list element either in a subkeys list or in a volatile subkeys list, the latter is examined last.

This field isn't used as of Windows XP.

##### Flags
In Windows XP and Windows Server 2003, the first 4 bits, counting from the most significant bit, are reserved for user flags (set via the *NtSetInformationKey()* call, read via the *NtQueryKey()* call). Other bits have the following meaning:

Mask|Name|Description
---|---|---
0x0001|KEY_VOLATILE|Is volatile (not used, a key node on a disk isn't expected to have this flag set)
0x0002|KEY_HIVE_EXIT|Is the mount point of another hive (a key node on a disk isn't expected to have this flag set)
0x0004|KEY_HIVE_ENTRY|Is the root key for this hive
0x0008|KEY_NO_DELETE|This key can't be deleted
0x0010|KEY_SYM_LINK|This key is a symlink (a target key is specified as a UTF-16LE string (REG_LINK) in a value named "SymbolicLinkValue", example: *\REGISTRY\MACHINE\SOFTWARE\Classes\Wow6432Node*)
0x0020|KEY_COMP_NAME|Key name is an ASCII string, possibly an extended ASCII string (otherwise it is a UTF-16LE string)
0x0040|KEY_PREDEF_HANDLE|Is a predefined handle (a handle is stored in the *Number of key values* field)

The following bits are also used as of Windows Vista:

Mask|Name|Description
---|---|---
0x0080|VirtualSource|This key was virtualized at least once
0x0100|VirtualTarget|Is virtual
0x0200|VirtualStore|Is a part of a virtual store path

It is plausible that both a registry key virtualization (when registry writes to sensitive locations are redirected to per-user locations in order to protect a Windows registry against corruption) and a registry key reflection (when registry changes are synchronized between keys in 32-bit and 64-bit views; this feature was removed in Windows 7 and Windows Server 2008 R2) required more space than this field can provide, that is why the *Largest subkey name length* field was split and the new fields were introduced.

Starting from Windows Vista, user flags were moved away from the first 4 bits of the *Flags* field to the new *User flags* bit field (see above). These user flags in the new location are also called *Wow64 flags*. In Windows XP and Windows Server 2003, user flags are stored in the old location anyway.

It should be noted that, in Windows Vista and Windows 7, the 4th bit (counting from the most significant bit) of the *Flags* field is set to 1 in many key nodes belonging to different hives; this bit, however, can't be read through the *NtQueryKey()* call. Such key nodes are present in initial primary files within an installation image (*install.wim*), and their number doesn't increase after the installation. A possible explanation for this oddity is that initial primary files were generated on Windows XP or Windows Server 2003 using the Wow64 subsystem (see below).

##### User flags
The *User flags* field (in the appropriate location for a version of Windows being used) is set according to the following bit masks:

Mask|Description
---|---
0x1|Is a 32-bit key: this key was created through the Wow64 subsystem or this key shall not be used by a 64-bit program (e.g. by a 64-bit driver during the boot)
0x2|This key was created by the reflection process (when reflecting a key from another view)
0x4|Disable registry reflection for this key
0x8|In the old location of the *User flags* field: execute the *int 3* instruction on an access to this key (both retail and checked Windows kernels), this bit was superseded by the *Debug* field (see below); in the new location of the *User flags* field: disable registry reflection for this key if a corresponding key exists in another view and it wasn't created by a caller (see below)

In Windows 7, Windows Server 2008 R2, and more recent versions of Windows, the bit mask 0x1 isn't used to mark 32-bit keys created by userspace programs.

The bit mask 0x8 was supported in the new location of the *User flags* field only in pre-release versions of Windows Vista, e.g. beta 2 [[1](https://web.archive.org/web/20051230061259/http://msdn.microsoft.com/library/en-us/sysinfo/base/regsetkeyflags.asp)].

##### Virtualization control flags
The *Virtualization control flags* field is set according to the following bit masks:

Mask|Name|Description
---|---|---
0x2|REG_KEY_DONT_VIRTUALIZE|Disable registry write virtualization for this key
0x4|REG_KEY_DONT_SILENT_FAIL|Disable registry open virtualization for this key
0x8|REG_KEY_RECURSE_FLAG|Propagate virtualization flags to new child keys (subkeys)

##### Debug
When the *CmpRegDebugBreakEnabled* kernel variable is set to 1, a checked Windows kernel will execute the *int 3* instruction on various events according to the bit mask in the *Debug* field. A retail Windows kernel has this feature disabled.

The following bit masks are supported:

Mask|Name|Event description
---|---|---
0x01|BREAK_ON_OPEN|This key is opened
0x02|BREAK_ON_DELETE|This key is deleted
0x04|BREAK_ON_SECURITY_CHANGE|A key security item is changed for this key
0x08|BREAK_ON_CREATE_SUBKEY|A subkey of this key is created
0x10|BREAK_ON_DELETE_SUBKEY|A subkey of this key is deleted
0x20|BREAK_ON_SET_VALUE|A value is set to this key
0x40|BREAK_ON_DELETE_VALUE|A value is deleted from this key
0x80|BREAK_ON_KEY_VIRTUALIZE|This key is virtualized

##### Layered keys
Layered keys were introduced in Insider Preview builds of Windows 10 "Redstone 1". When a hive supports the layered keys feature, a kernel may treat some key nodes in a special way.

When a kernel is accessing a key node treated as a part of a layered key, it builds a key node stack, including the key node being accessed, its parent key node and no more than 2 (or 14 as of Windows 10 Insider Preview Build 14986, or 126 as of Windows 10 Insider Preview Build 15025) parent key nodes towards the registry root (crossing a mount point is possible). Then this stack is used to produce cumulative information about the layered key. For example, if you query the last written timestamp for a layered key, the most recent timestamp will be returned from the key node stack; if you enumerate key values for a layered key, key values from key nodes in the stack will be returned (except tombstone values; if there are two or more values with the same name in the key node stack, a value from a lower (child) key node is used and returned).

When the *Inherit class* field is set to 0, the layered key will have the same class name as the key node originally accessed by a kernel. Otherwise, the layered key will receive the same class name (possibly an empty class name) as an upper (parent) key node (from the stack) having the *Inherit class* field set to 0.

The *Layer semantics* field is set using the following values:

Value|Name|Description
---|---|---
0| |This key node and its parent key nodes can be included in the current layered key (based on the stack built)
1|IsTombstone|Is a tombstone key node: this key node and its parent key nodes can't be included in the current layered key (also, such a key node has no class name, no subkeys, and no values)
2|IsSupersedeLocal|This key node can be included in the current layered key, but its parent key nodes can't
3|IsSupersedeTree|This key node can be included in the current layered key, but its parent key nodes can't; child key nodes, except tombstone key nodes, are required to have the same value set in the field

#### Key values list
The *Key values list* has the following structure:

Offset|Length|Field
---|---|---
0|...|List elements

A list element has the following structure:

Offset|Length|Field|Description
---|---|---|---
0|4|Key value offset|In bytes, relative from the start of the hive bins data

List elements are not required to be sorted.

#### Key value
The *Key value* has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|2|Signature|vk|ASCII string
2|2|Name length| |In bytes, can be 0 (name isn't set)
4|4|Data size| |In bytes, can be 0 (value isn't set), the most significant bit has a special meaning (see below)
8|4|Data offset| |In bytes, relative from the start of the hive bins data (or data itself, see below)
12|4|Data type| |See below
16|2|Flags| |Bit mask, see below
18|2|Spare| |Not used
20|...|Value name string| |ASCII (extended) string or UTF-16LE string

##### Data size
When the most significant bit is 1, data (4 bytes or less) is stored in the *Data offset* field directly (when data contains less than 4 bytes, it is being stored as is in the beginning of the *Data offset* field). The most significant bit (when set to 1) should be ignored when calculating the data size.

When the most significant bit is 0, data is stored in the *Cell data* field of another cell (pointed by the *Data offset* field) or in the *Cell data* fields of multiple cells (referenced in the *Big data* structure stored in a cell pointed by the *Data offset* field).

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

Other values are allowed as well (but they are not predefined).

##### Flags

Mask|Name|Description
---|---|---
0x0001|VALUE_COMP_NAME|Value name is an ASCII string, possibly an extended ASCII string (otherwise it is a UTF-16LE string)
0x0002|IsTombstone|Is a tombstone value (the flag is used starting from Insider Preview builds of Windows 10 "Redstone 1"), a tombstone value also has the *Data type* field set to REG_NONE, the *Data size* field set to 0, and the *Data offset* field set to 0xFFFFFFFF

#### Key security
The *Key security* item has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|2|Signature|sk|ASCII string
2|2|Reserved| |
4|4|Flink| |See below
8|4|Blink| |See below
12|4|Reference count| |Number of key nodes pointing to this item
16|4|Security descriptor size| |In bytes
20|...|Security descriptor| |

##### Notes
1. Flink and blink are offsets in bytes, relative from the start of the hive bins data.
2. Key security items form a doubly linked list. A key security item may act as a list header or a list entry (the only difference here is the meaning of flink and blink fields).
3. When a key security item acts as a list header, flink and blink point to the first and the last entries of this list respectively. If a list is empty, flink and blink point to a list header (i.e. to a current cell).
4. When a key security item acts as a list entry, flink and blink point to the next and the previous entries of this list respectively. If there is no next entry in a list, flink points to a list header. If there is no previous entry in a list, blink points to a list header.

#### Big data
The *Big data* is used to reference data larger than 16344 bytes (when the *Minor version* field of the base block is greater than 3), it has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|2|Signature|db|ASCII string
2|2|Number of segments| |
4|4|Offset of a list of segments| |In bytes, relative from the start of the hive bins data

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

Data segments of a *Big data* record, except the last one, always have the maximum size.

### Unallocated cell
If a cell to be marked as unallocated has an adjacent unallocated cell, these cells are coalesced, this is why a single unallocated cell may contain multiple remnant records (entities).

In Windows 2000, the following record is written to the beginning of the *Cell data* field of an unallocated cell:

Offset|Length|Field|Description
---|---|---|---
0|4|Next|Offset of a next unallocated cell in a free list (in bytes, relative from the start of the hive bins data) or 0xFFFFFFFF, if there is no such a cell in that list (there are many free lists for a single hive in memory)

The record described above isn't used as of Windows XP.

### Summary
1. A *Base block* points to a root cell, which contains a *Key node*.
2. A *Key node* points to a parent *Key node*, to a *Subkeys list* (a subkey is a *Key node* too), to a *Key values list*, to a *Key security* item.
3. A *Subkeys list* can be subdivided with the help of the *Index root* structure.
4. A *Key value* points to data. Data may be stored in the *Data offset* field of a *Key value* structure, in a separate cell, or in a bunch of cells. In the last case, a *Key value* points to the *Big data* structure in a cell.

## Format of transaction log files

### Old format
A transaction log file (old format) consists of a base block, a dirty vector, and dirty pages.

![Transaction log file layout (old format)](https://raw.githubusercontent.com/msuhanov/regf/master/images/old-log.png "Transaction log file layout (old format)")

#### Base block
A partial backup copy of a base block is stored in the first sector of a transaction log file, only the first *Clustering factor * 512* bytes of a base block are written there.

A backup copy of a base block isn't an exact copy anyway, the following modifications are performed on it by a hive writer:

1. As of Windows XP, the *File type* field is set to 1 (1 means *transaction log*). In Windows 2000, the *File type* field is set to 2 (while 1 is reserved for in-memory hive recovery management using an alternate file for the System hive – *\WINNT\system32\config\SYSTEM.ALT*, an alternate file is a mirror of a primary file used as a backup copy).
2. In Windows 2000, Windows XP, and Windows Server 2003, a *primary sequence number* is incremented by 1 before writing a log of dirty data; a *secondary sequence number* is incremented by 1 after a log of dirty data was written. As of Windows Vista, a *primary sequence number* and the same *secondary sequence number* (incremented by 1) are written at once before writing a log of dirty data.
3. The *Checksum* field is recalculated for the modified data.

##### Notes
1. A partial backup copy of a base block is made using data from memory, not from a primary file.
2. A transaction log file is considered to be valid when it has an expected base block (including the modifications mentioned above), and its *primary sequence number* is equal to its *secondary sequence number*. An invalid transaction log file can be repaired by the self-healing process (this will reset the fields of a base block in memory to reasonable values, i.e. the *Signature* field will be reset to the proper value, the *Hive bins data size* field will be reset to a value calculated from the file size of a primary file, the *Clustering factor* field will be reset to a value for an underlying disk, the sequence numbers will be reset to 1, the *Checksum* field will be recalculated).
3. A valid transaction log file can be applied to a dirty hive when a *Last written timestamp* in its base block is equal to a *Last written timestamp* in a base block of a primary file (when a base block of a primary file is invalid, i.e. it has a wrong *Checksum*, a *Timestamp* from the first hive bin is used instead). Also, a transaction log file can be applied to a dirty hive after the self-healing process.
4. If a base block of a primary file has a wrong *Checksum*, it is being recovered using a base block from a transaction log file (and the *File type* field is set back to 0).
5. In Windows 2000, Windows XP, and Windows Server 2003, the same memory region is used to write a base block both to a transaction log file and to a primary file, that is why, after a successful write operation, a base block of a primary file will contain two sequence numbers equal to *N*, and a base block of a transaction log file will contain two sequence numbers equal to *N - 1* (sequence numbers are incremented by 1 for a transaction log file first, and then they are incremented by 1 again for a primary file). As of Windows Vista, a copy of a base block is used when writing to a transaction log file, that is why a transaction log file and a primary file will contain the same sequence numbers in their base blocks after a successful write operation.

#### Dirty vector
The *Dirty vector* is stored starting from the beginning of the second sector of a transaction log file, it has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|4|Signature|DIRT|ASCII string
4|...|Bitmap| |Bitmap of dirty pages

Each bit of a bitmap corresponds to the state of a specific *512-byte* page within hive bins data to be written to a primary file from memory, regardless of a logical sector size of an underlying disk (these pages don't overlap, there are no gaps between them):
* the first bit, *bit #1*, corresponds to the state of the first *512-byte* page within hive bins data;
* *bit #2* corresponds to the state of the second *512-byte* page within hive bins data, etc.

Bits of a bitmap are checked using the *bt* instruction or its equivalent based on bit shifting. This means that bits are packed into bytes, the first byte of a bitmap contains bits #1-#8, the second byte contains bits #9-#16, and so on. Within a byte, bit numbering starts at the least significant bit.

The meaning of each bit in a bitmap is the following:

Bit|Description
---|---
0|A corresponding page is clean
1|A corresponding page is dirty

Bitmap length (*in bits*) is calculated using the following formula: *Bitmap length = Hive bins data size / 512*.

##### Notes
1. The state of a base block isn't recorded in a dirty vector.
2. A dirty vector is stored in memory as a *RTL_BITMAP* structure. However, only the *Buffer* field of this structure is written to a transaction log file.
3. Windows tracks changes to hive bins data in 4096-byte blocks (as of Windows XP). This means that a dirty vector's bitmap is *expected* to contain only the following *bytes*: 0x00 and 0xFF.
4. A padding is likely to be present after the end of a bitmap (up to a sector boundary).

#### Dirty pages
*Dirty pages* are stored starting from the beginning of the sector following the last sector of a dirty vector. Each dirty page is stored at an offset divisible by 512 bytes and has a length of 512 bytes, there are no gaps between dirty pages.

The first dirty page corresponds to the first bit set to 1 in the bitmap of a dirty vector, the second dirty page corresponds to the second bit set to 1 in the bitmap of a dirty vector, etc. During recovery, contiguous dirty pages belonging to the same hive bin in a primary file are processed together, and a dirty hive bin is verified for correctness (its *Signature* must be correct, its *Size* must not be less than 4096 bytes, its *Offset* must match the *Offset* of a corresponding hive bin in a primary file); recovery stops if a dirty hive bin is invalid, an invalid dirty hive bin is ignored.

##### Notes
1. The number of dirty pages is equal to the number of bits set to 1 in the bitmap of a dirty vector. Remnants of previous dirty pages may be present after the end of the last dirty page.
2. A dirty page will be written to a primary file at the following offset: *File offset = 4096 + 512 * Bit position*, where *Bit position* is the index (zero-based) of a corresponding bit in the bitmap of a dirty vector.

### New format
A transaction log file (new format) consists of a base block and log entries. This format was introduced in Windows 8.1 and Windows Server 2012 R2.

![Transaction log file layout (new format)](https://raw.githubusercontent.com/msuhanov/regf/master/images/new-log.png "Transaction log file layout (new format)")

#### Base block
A modified partial backup copy of a base block is stored in the first sector of a transaction log file in the same way as in the old format and for the same purpose. However, the *File type* field is set to 6.

#### Log entries
*Log entries* are stored starting from the beginning of the second sector. Each log entry is stored at an offset divisible by 512 bytes and has a variable size (multiple of 512 bytes), there are no gaps between log entries.

A log entry has the following structure:

Offset|Length|Field|Value|Description
---|---|---|---|---
0|4|Signature|HvLE|ASCII string
4|4|Size| |Size of a current log entry in bytes
8|4|Flags| |Partial copy of the *Flags* field of the base block at the time of creation of a current log entry (see below)
12|4|Sequence number| |This number constitutes a possible value of the *Primary sequence number* and *Secondary sequence number* fields of the base block in memory after a current log entry is applied (these fields are not modified before the write operation on the recovered hive)
16|4|Hive bins data size| |Copy of the *Hive bins data size* field of the base block at the time of creation of a current log entry
20|4|Dirty pages count| |Number of dirty pages attached to a current log entry
24|8|Hash-1| |See below
32|8|Hash-2| |See below
40|...|Dirty pages references| |
...|...|Dirty pages| |

A dirty page reference describes a single page to be written to a primary file, and it has the following structure:

Offset|Length|Field|Description
---|---|---|---
0|4|Offset|Offset of a page in a primary file (in bytes), relative from the start of the hive bins data
4|4|Size|Size of a page in bytes

*Dirty pages* are attached to a log entry in the same order as in the *Dirty pages references* without an alignment or gaps.

##### Notes
1. *Hash-1* is the Marvin32 hash of the data starting from the beginning of the first page reference of a current log entry with the length of *Size - 40* bytes.
2. *Hash-2* is the Marvin32 hash of the first 32 bytes of a current log entry (including the *Hash-1* calculated before).
3. The following seed is used for calculating the *Hash-1* and *Hash-2* (hexadecimal bytes): 82 EF 4D 88 7A 4E 55 C5.
4. A transaction log file may contain multiple log entries written together (the base block in a transaction log file is updated only when writing the first log entry), as well as old (already applied) log entries.
5. If a primary file is dirty and has a valid *Checksum* (in the base block), only subsequent log entries are applied. A subsequent log entry is a log entry with a sequence number equal to or greater than a *primary sequence number* of the base block in a transaction log file.
6. If a primary file is dirty and has a wrong *Checksum* (in the base block), its base block is recovered from a transaction log file. Then subsequent log entries are applied.
7. If a log entry with a sequence number *N* is *not* followed by a log entry with a sequence number *N + 1*, recovery stops after applying a log entry with a sequence number *N*. If the first log entry doesn't contain an expected sequence number (equal to a *primary sequence number* of the base block in a transaction log file, not less than a *secondary sequence number* of the valid base block in a primary file), recovery stops.
8. If a log entry has a wrong value in the field *Hash-1*, *Hash-2*, or *Hive bins data size* (i.e. it isn't multiple of 4096 bytes), recovery stops, only previous log entries (preceding a bogus one) are applied.
9. A primary file is grown according to the *Hive bins data size* field of a log entry being applied.
10. Dirty hive bins are verified for correctness during recovery (but recovery doesn't stop on an invalid hive bin, an invalid hive bin is replaced with a dummy hive bin instead).
11. The *Flags* field of a log entry is set to 0x00000001 when a value of the *Flags* field of the base block has the bit mask 0x00000001 set, otherwise the *Flags* field of a log entry is set to 0x00000000. During recovery, the bit mask 0x00000001 is set or unset in the *Flags* field of the base block according to a value taken from a log entry being applied. This means that only the bit mask 0x00000001 is saved to or restored from a log entry.

## Dirty state of a hive
A hive is considered to be dirty (i.e. requiring recovery) when a base block in a primary file contains a wrong checksum, or its *primary sequence number* doesn't match its *secondary sequence number*. If a hive isn't dirty, but a transaction log file (new format) contains subsequent log entries, they are ignored.

## Multiple transaction log files
A hive writer may use no transaction log files, a single transaction log file (\*.LOG), or two transaction log files (\*.LOG1 and \*.LOG2). In the last case, also known as a dual-logging scheme (introduced in Windows Vista), a dummy third transaction log file (\*.LOG) may be present as an artifact from an installation image. When a hive writer is using a single transaction log file, two empty transaction log files from the dual-logging scheme (\*.LOG1 and \*.LOG2) may be present as well.

### Old format
Under normal circumstances, only the first transaction log (\*.LOG1) file is used. If an error occurs when writing to a primary file, a switch to the second transaction log file (\*.LOG2) is performed (this file will contain a cumulative log of dirty data, i.e. dirty pages that weren't written in full to a primary file due to an error and pages dirtied since the failed write, on the next write attempt). If write errors persist, a hive writer will swap the transaction log file being used on every write attempt (\*.LOG2 to \*.LOG1 and vice versa), keeping a cumulative log of dirty data. This allows a hive writer to keep a consistent copy of previous dirty data in another transaction log file on each write attempt (if a system crash occurs when writing to a current transaction log file, thus leaving it in the inconsistent state, a primary file may be recovered later using a previous transaction log file). After a successful write operation on a primary file, the first transaction log file will be used again. If an error occurs when writing a base block to a primary file in the beginning of a write operation (in order to update the *Primary sequence number* and *Last written timestamp* fields), the whole operation fails without changing the log file being used.

In the general case, the first transaction log file is used by a kernel to recover a dirty hive. Before Windows 8, if a primary file contains an invalid base block (i.e. it has a wrong *Checksum*), and the first transaction log file doesn't contain a valid backup copy of a base block (or, if this copy of a base block is valid, it doesn't contain a matching *Last written timestamp*, i.e. this transaction log file can't be applied to a dirty hive, see above), and the second transaction log file contains a valid backup copy of a base block with a mismatching *Last written timestamp* and a more recent log of dirty data than the first transaction log file (according to the *Last written timestamp* fields of the backup copies of a base block in these files, this check is ignored if the first transaction log file contains an invalid backup copy of a base block), then the second transaction log file is used by a kernel to recover a dirty hive.

Such a recovery algorithm is extremely ineffective, because it doesn't use the second transaction log file unless a base block of a primary file is invalid, and this base block is likely to be valid, because an error when writing a base block to a primary file in the beginning of a write operation will not trigger the switch to the second transaction log file, so the most probable event triggering this switch is a write error when storing dirty data in a primary file, that is likely to leave a valid base block in a primary file (the mid-update state).

In Windows 8, the second transaction log file is used by a kernel to recover a dirty hive when the conditions mentioned above are met, with the following exceptions: the second transaction log file is used even if a base block of a primary file is valid, the second transaction log file is used even if its backup copy of a base block has a matching *Last written timestamp*. The new algorithm is much more sound. In March 2016, Microsoft released an updated kernel for Windows 7 (6.1.7601.19160, KB3140410), this kernel implements a recovery algorithm similar to existing in Windows 8.

When a dirty hive is loaded by a boot loader (not by a kernel), the applicable transaction log file (i.e. having a matching *Last written timestamp* in a valid backup copy of a base block) is used in the recovery. The first transaction log file is tried first.

Another shortcoming in the implementation of the dual-logging scheme is that sequence numbers in a backup copy of a base block in a transaction log file are not used to record its mid-update state (see above). If a system crash occurs when writing to a transaction log file, there will be no clear indicators of which transaction log file is inconsistent. It is possible for an operating system to pick an inconsistent transaction log file for the recovery.

### New format
A hive writer will regularly swap the transaction log file being used (\*.LOG1 to \*.LOG2 and vice versa). This may divide log entries between two transaction log files; the first transaction log file isn't guaranteed to contain earlier log entries.

If a primary file contains a valid base block, both transaction log files are used to recover the dirty hive, i.e. log entries from both transaction log files are applied; the transaction log file with earlier log entries is used first (if recovery stops when applying log entries from this transaction log file, then recovery is resumed with the next transaction log file; the first log entry of the next transaction log file is expected to have a sequence number equal to *N + 1*, where *N* is a sequence number of the last log entry applied). If a primary file contains an invalid base block, only the transaction log file with latest log entries is used in the recovery.

## Flush strategies
Flushing a hive ensures that its dirty data was written to a disk. When the old format of transaction log files is used, this means that dirty data was stored in a primary file. When the new format of transaction log files is used, a flush operation on a hive will succeed after dirty data was stored in a transaction log file (but not yet in a primary file); a hive writer may delay writing to a primary file (up to an hour), in this situation dirty data becomes unreconciled.

## Sector size and clustering factor
As of Windows 8, the *Clustering factor* field is always set to 1, the logical sector size is always assumed to be 512 bytes when working with related offsets and sizes. For example, a backup copy of a base block in a transaction log file is 512 bytes in length regardless of a logical sector size of an underlying disk.

According to Microsoft, there is no support for a logical sector size different from 512 bytes and 4096 bytes in Windows; a logical sector size equal to 4096 bytes is supported as of Windows 8 and Windows Server 2012 [[2](https://msdn.microsoft.com/en-us/library/windows/desktop/hh848035(v=vs.85).aspx)]. This is why the *Clustering factor* field is expected to be equal to 1.

## Very old versions of Windows
The format description above applies to registry hives with the following major and minor version numbers in the *Base block* structure: 1.3, 1.4, 1.5, and 1.6. However, the following major and minor version numbers can be found in registry hives of Windows NT 3.1, Windows NT 3.5, and Windows NT 3.51: 1.1 and 1.2.

When the *Minor version* field of the base block is equal to 2, the *Fast leaf* records are not supported.

When the *Minor version* field of the base block is equal to 1, the *Fast leaf* records are not supported; the *Key name string* of the *Key node* and the *Value name string* of the *Key value* are always UTF-16LE; the *Flags* and *Spare* fields of the *Key value* don't exist, there is a *Title index* field at the offset 16 bytes with the length of 4 bytes; the *Access bits* (*Spare*) field of the *Key node* doesn't exist, there is a *Title index* field at the offset 12 bytes with the length of 4 bytes; and every cell has the following structure:

Offset|Length|Field|Description
---|---|---|---
0|4|Size|Size of a current cell in bytes, including this field (aligned to 16 bytes): the size is positive if a cell is unallocated or negative if a cell is allocated (use absolute values for calculations)
4|4|Last|Offset of a previous cell in bytes, relative from the start of a current hive bin (or 0xFFFFFFFF for the first cell in a hive bin)
8|...|Cell data|

The *Title index* field is used to store an index of a localized alias for a name string (there is no alias for a name string if this field is equal to 0). Although the *Title index* field can be set and read in Windows NT 3.1, localized aliases were never supported (and the *Title index* field became deprecated in Windows NT 3.5).

In Windows NT 3.1, the following record is written to the beginning of the *Cell data* field of an unallocated cell:

Offset|Length|Field|Description
---|---|---|---
0|4|Next|Offset of a next unallocated cell in a free list (in bytes, relative from the start of the hive bins data) or 0xFFFFFFFF, if there is no such a cell in that list
4|4|Previous|Offset of a previous unallocated cell in a free list (in bytes, relative from the start of the hive bins data) or 0xFFFFFFFF, if there is no such a cell in that list

Registry hives with the *Minor version* field of the base block set to 0 can be found in pre-release versions of Windows NT 3.1. This hive version is similar to the version 1.1, but it is out of the scope of this document.

Also, the *File type* field of a base block in a transaction log file is set to 2 in all versions of Windows NT up to and including Windows 2000.

## Additional sources of information
1. http://www.sentinelchicken.com/data/TheWindowsNTRegistryFileFormat.pdf
2. https://github.com/libyal/libregf/blob/master/documentation/Windows%20NT%20Registry%20File%20(REGF)%20format.asciidoc
3. http://amnesia.gtisc.gatech.edu/~moyix/suzibandit.ltd.uk/MSc/

-------
© 2015-2018 Maxim Suhanov

This work is licensed under the [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/).
