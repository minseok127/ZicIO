This repository archives my contributions to ZicIO, a project featured in a research paper accepted at sigmod 2025. It includes only partial code, which cannot independently run the full system.

## zicio_notify.h, zicio_data_buffer_descriptor.c

Originally, ZicIO was designed targeting PostgreSQL's sequential scan. However, it was later extended to support various columnar analytical systems, leading to significant changes. PostgreSQL's sequential scan allowed reading the entire file in any order, and ZicIO was initially designed based on this assumption. In contrast, columnar analytical systems often read only specific parts of a file, where the order of reads is critical. To meet these new requirements, new call paths and APIs were introduced, which are prefixed with 
***zicio_notify***. This naming reflects the idea that the user notifies ZicIO of the regions and order to read from the file.

The most significant change in *zicio_notify* was the process of generating NVMe commands. Creating an NVMe command requires a physical block address, which means an *ext4_extent* is needed when using the ext4 file system. If the read order does not matter, the process is simple, as any part of the file's data can be read into any position in the buffer. There is no need to search for a specific *ext4_extent* to read into the buffer.

However, the problem is different for columnar analytical systems.

## zicio_flow_ctrl.h, zicio_flow_ctrl.c

## zicio_nvme_cmd_timer_wheel.h, zicio_nvme_cmd_timer_wheel.c
