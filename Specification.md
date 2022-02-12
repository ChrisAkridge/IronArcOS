# IronArc OS Specifications

This file is mostly a place for draft specifications that will be split out into the Specifications folder when they get large enough.

## Layers of the Filesystem

### IronArc storage device

The hardware device mapped to a file on the host machine. Reads and writes into memory using streams that the VM can open or close.

### Sector Manager

Divides the storage device into evenly-sized sectors whose size is a power of 2. Provides methods to use _sector-aligned streams_. We may use the first however-many KB of the storage device as an un-sectored partition table and divide everything that follows into sectors, with address-of-first-sector being part of the partition table.

### IronArc OS Filesystem

Marks certain sectors as _table sectors_ and others as _file sectors_. Sectors are joined together in a doubly-linked list using the first and last 8 bytes as previous and next pointers. The table sectors all make up the _file table_, which stores all the file and folder information, including names, sizes, and addresses to the first and last sectors of each file. Each file is its own linked list of sectors.

The filesystem APIs abstract away the sector handling and allow you to read and write from files as if they were contiguous arrays of bytes that can be extended (although inserting bytes in the middle is still an O(n) operation). Fundamental operations include create, open, close, move (which also can be used for renaming), and delete.

## Booting into IronArc OS

I'm envisioning the boot process to look something like this:

1. User opens IronArc on the host machine and selects an IEXE file containing a bootloader. The hardware loadout should at least include a terminal and a storage device containing an install of IronArc OS.
2. Execution starts. The bootloader looks at the first few kilobytes of the storage device to read partition information, which includes what OS is installed on a partition.
3. The bootloader presents the OS options to the user, who uses the terminal to specify an option.
4. The parition data for an OS partition contains a pointer to a section of IronArc executable code that can be used to boot the OS.

## List of Things to Figure Out

Okay, we've defined a lot of terms here. Let's make a high-level List of Stuff this project will need.

- Kinds of storage device addresses (absolute, partition, sector-aware* etc.)
- Partition table format
- Binary executable format
- Folder and file table entry formats
- Kernel memory management and kernel/user mode separation
- Standard library (compare to the C standard library)
- User and group permissions, administration permissions
- Process handling, memory layout, and pre-emptive multitasking
- Process signals
- Shell and terminal display API

* Name subject to change.

## Kinds of Storage Device Addresses

Much like how Windows executables have absolute, relative, and virtual addresses, storage devices have multiple kinds of addresses.

An _absolute address_ on a storage device is an address inside the file on the host machine that is selected by the user. They can address the partition table and linked-list pointers in each sector.

A _partition address_, when paired with a partition ID, addresses bytes inside a partition. They can address the linked-list pointers in each sector.

A _file address_, implemented at the higher levels of the file system, addresses bytes inside a file in the file system. It must be translated by the file system into a partition address, which needs to use the linked-list pointers in each sector to follow the sectors that make up a file.

A _file table address_ addresses bytes inside the file table. They are translated in a very similar fashion to file addresses, except that there's only one "file" that makes up the file table.

## Partition Table Format

A file loaded into a storage device, formatted for running IronArc OS, has the following structure.

The file opens with a _partition table header_ that takes the following structure:

```c
struct PartitionTableHeader
{
    int MagicNumber;    // "ISPT" (0x49535054), short for IronArc Storage Partition Table
    int SectorSize;
    int PartitionCount;
    Partition[PartitionCount] Partitions;
}
```

The _sector size_ determines the size of the sectors that the file is split into. It must be a power of two and at least 32 bytes - 4,096 bytes is the preferred standard.

Immediately following the header is the list of partitions, each partition structured as follows:

```c
struct Partition
{
    long AbsoluteStartAddress;
    long Length;
    byte[64] Name;
    byte IsBootable;
    long OSBootProgramPartitionAddress;
    int OSBootProgramLength;
}
```

The _partition name_ is displayed to the user on the boot screen if `IsBootable` is true. If the user chooses to boot from this partition, the boot program copies `OSBootProgramLength` bytes from `OSBootProgramPartitionAddress` into memory, just after the boot program. It then jumps to the copied program and begins execution, handing over control permanently to the OS boot program and the operating system.

Each partition has a _partition ID_ that is implicitly equal to the index of the `Partition` struct in the `PartitionTableHeader.Partitions` array, starting from 0.

Debugging is important, so we'll extend the IronArc host program to read and write partition tables in specified files.

## Binary Executable Format