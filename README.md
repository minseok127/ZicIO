ZicIO is the name of the asynchronous I/O system implemented for the paper ***Rapid Data Ingestion through DB-OS Co-design***, accepted at SIGMOD 2025. This repository archives the code I wrote for ZicIO and my thoughts about it. It includes only partial code, which cannot independently run the full system.

## zicio_notify.h, zicio_data_buffer_descriptor.c

Originally, ZicIO was designed targeting PostgreSQL's sequential scan. Then it was later extended to support various columnar analytical systems, leading to significant changes. PostgreSQL's sequential scan allowed reading the entire file in any order, and ZicIO was initially designed based on this assumption. In contrast, columnar analytical systems often read only specific parts of a file, where the order of reads is critical. So new call paths and APIs were introduced, which are prefixed with 
***zicio_notify***. This naming reflects the idea that the user notifies ZicIO of the regions and order to read from the file.

The most significant change in *zicio_notify* was the process of generating NVMe commands. Creating an NVMe command requires a physical block address, which means an *ext4_extent* is needed when using the ext4 file system. If the read order does not matter, the process is simple, as any part of the file's data can be read into any position in the buffer. There is no need to search for a specific *ext4_extent* to read into the buffer.

When the read order becomes important, it turns into a more complex problem. There may be no consistent pattern in the reading process. The reading pattern varies by file format and differs for each query. The size of the skipped regions is not fixed and some regions such as metadata may need to be read again.

ext4 tries to minimize I/O operations when it looks for an ext4_extent. Previously accessed extents are managed in memory using a red-black tree. Before performing an I/O operation, ext4 first searches for the required extent in this cache. In ZicIO, the task of finding ext4_extent is handled by the interrupt handler. So I felt it was necessary to keep the time complexity as low as possible (although it may have been a premature optimization). This led me to consider whether it would be possible to directly obtain the information corresponding to the buffer position without traversing the tree.

So I started creating a mapping table that uses buffer positions as keys to retrieve the information needed for I/O for the given positions. The original ZicIO used a 4MB buffer to cache ext4_extent, and my focus was on maintaining this amount of memory, avoiding extra resource overhead over the previous implementation.

The mapping table consists of two arrays. One is an array of data structure containing the minimal information needed to create a single NVMe command, each element corresponds to each read I/O. The other specifies which range of that array is required for each 2MB huge page that makes up the buffer. When creating I/O commands for a 2MB huge page, the range of offsets listed in this array is used to access the elements of the aforementioned list. Since ZicIO's buffer is a circular buffer composed of 2MB huge pages, knowing the buffer position and round allows access to the compressed I/O command information for that position through the second array.

To calculate the size of read I/O that the mapping table can cover, it is necessary to know the size of the compressed information. The minimum I/O unit of an SSD is called a *sector*, which is 512 bytes. ext4 manages an NVMe page in units of 4KB, composed of 8 sectors. The device we used had a maximum I/O size of 256KB. Thus, the size of an I/O command can range from a minimum of 4KB to a maximum of 256KB, with intervals of 4KB. This range can be represented by values from 0 to 63, requiring 6 bits. The ext4_extent stores the physical block address across @ee_start_hi and @ee_start_lo, which together total 48 bits. ZicIO can read up to 128 files on a single channel, requiring 7 bits to represent this. So a total of 61 bits are required for a single read I/O. This information is declared as 8 bytes to ensure alignment, with the remaining 3 bits reserved.

In summary, 8 bytes of compressed information can be used to create a single read I/O command ranging from 4KB to 256KB. Naively, in the worst case 4MB can cover 2GB of read I/O, and in the best case it can cover 128GB of read I/O. The worst case occurs when the data to be read is fully fragmented into 4KB segments. In the experiments conducted for the paper, datasets ranging from 80GB to 100GB were covered using 4MB to 8MB of memory. 

This design still has constraints. The process of populating this mapping table occurs during channel opening. So if the file system changes logical-to-physical mappings after channel opening, affected information will need to be updated. Memory usage also depends on how well the file system handles fragmentation, making the design's overheads and constraints closely tied to file system behavior.

## zicio_flow_ctrl.h, zicio_flow_ctrl.c

## zicio_nvme_cmd_timer_wheel.h, zicio_nvme_cmd_timer_wheel.c
