CSE 321: Operating Systems - Summer 2026
Lab Term Project: SimpleFS - Implementation of a Simple File System in C

Group Number: 07
Student IDs: 22101475, 22301484

===============================================================================
FILES SUBMITTED
===============================================================================
1. simplefs.h          - Fixed constants/structures (unmodified, as required).
2. simplefs_builder.c  - Creates and initializes an empty SimpleFS image.
3. simplefs_adder.c    - Adds a file from the current directory into the image.
4. README.txt          - This file.

===============================================================================
COMPILATION
===============================================================================
gcc -Wall -Wextra -std=c11 simplefs_builder.c -o simplefs_builder
gcc -Wall -Wextra -std=c11 simplefs_adder.c -o simplefs_adder

Both files compile cleanly with no warnings under -Wall -Wextra -std=c11.

===============================================================================
EXECUTION EXAMPLES
===============================================================================
Create an empty image:
    ./simplefs_builder --image disk.img

Add a file to the image:
    ./simplefs_adder --input disk.img --file test1.txt
    ./simplefs_adder --input disk.img --file test2.txt
    ./simplefs_adder --input disk.img --file test3.txt

Inspect the image:
    xxd disk.img
    xxd -s 4096 -l 16 disk.img      (inode bitmap)
    xxd -s 8192 -l 16 disk.img      (data bitmap)
    xxd -s 16384 -l 128 disk.img    (root directory: "." and "..")

===============================================================================
IMPLEMENTATION DESCRIPTION
===============================================================================
simplefs_builder.c
------------------
- Creates a 262144-byte (64 x 4096) image and zero-fills all blocks.
- Fills the superblock (magic 0x53465331, block size 4096, 64 total blocks,
  32 inodes, and the fixed block numbers for the inode bitmap, data bitmap,
  inode table, and data region) and writes it to Block 0.
- Sets bit 0 of the inode bitmap (Block 1) to mark inode 1 (root) allocated.
- Sets bit 0 of the data bitmap (Block 2) to mark Block 4 (root's data block)
  allocated.
- Initializes the root inode (type = directory, links = 2, size = 128 bytes
  for the two 64-byte entries, direct[0] = 4) and writes it to inode slot 1
  in the inode table (Block 3).
- Writes the "." and ".." directory entries (both pointing to inode 1) to
  the beginning of Block 4.

simplefs_adder.c
-----------------
- find_free_inode(): scans inode-bitmap bit indexes 1..31 (inode 1 is
  reserved for root) for the first unallocated bit and returns it converted
  to an inode number (index + 1).
- find_free_data_block(): scans data-bitmap bit indexes 0..59 for the first
  unallocated bit and returns it converted to an absolute block number
  (DATA_REGION_BLOCK + index).
- filename_exists(): reads all 64 directory entries in the root block and
  reports a match against any entry whose inode_no is non-zero.
- find_free_directory_entry(): scans entries 2..63 (0 and 1 are "." and
  "..") for the first entry with inode_no == 0.
- main(): validates arguments, opens and validates the image (magic number
  check), opens the source file, checks its size against the 12288-byte
  limit and the file name against the 58-character limit, computes the
  number of required blocks via ceiling division, rejects duplicate names,
  allocates a free inode and the required number of free data blocks with
  first-fit, copies the source data into the allocated blocks in
  zero-filled 4096-byte buffers (so any unused tail bytes of the last block
  stay zero), builds and writes the new inode (type, links = 1, actual
  size, and direct pointers), updates both bitmaps, writes the new
  directory entry (inode number, type, null-terminated name) into the free
  slot, and finally increases the root inode's size by 64 bytes for the new
  entry.

===============================================================================
TESTING PERFORMED
===============================================================================
All test cases described in Section 19 of the specification were run and
verified with hexdump/xxd, including:
- Empty image is exactly 262144 bytes with correctly initialized superblock,
  bitmaps, and root directory.
- Adding a small (< 1 block) file: inode bitmap byte becomes 0x03, data
  bitmap byte becomes 0x03, and file contents are verified in Block 5.
- Adding a 5000-byte file uses two blocks (direct[0]=5, direct[1]=6, data
  bitmap byte 0x07).
- A 12288-byte file is accepted; a 12289-byte file is rejected.
- Adding three files in sequence allocates inodes 2, 3, 4 in order
  (first-fit), inode bitmap byte becomes 0x0F.
- Duplicate file names are rejected with the required error message.
- Missing source files and missing images are rejected gracefully (no
  crash), with exit code 1.
- A zero-byte file is accepted, consumes an inode but zero data blocks, and
  leaves the data bitmap unchanged.
- A file name longer than 58 characters is rejected.

===============================================================================
CONTRIBUTION OF EACH GROUP MEMBER
===============================================================================

- Gourab Krishna Saha (22301484): Full implementation of simplefs_builder.c (superblock,
  bitmaps, root inode, root directory entries); compilation and testing of
  image creation.

- Farsin Rahman Nahin (22101475): Full implementation of simplefs_adder.c (inode/data-block
  allocation, duplicate/size/name checks, file copy, new inode and directory
  entry creation); README documentation; final verification with xxd.

===============================================================================
KNOWN LIMITATIONS / PROBLEMS
===============================================================================
- As specified, SimpleFS supports only a flat root directory (no
  subdirectories), no indirect pointers, no deletion/renaming, and no
  permissions/journaling/caching — these were intentionally out of scope
  per the project specification.
- Maximum of 31 user files (32 inodes minus the root inode).
- Maximum file size is 12288 bytes (3 direct blocks x 4096 bytes).
- No other known issues; all required test cases in the specification pass.
