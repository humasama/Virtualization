# VM Address

Guest OS code is run by a pthread created by QEMU process.
For the guest, there are two kinds of addresses: guest virtual address (GVA) and guest physical address (GPA).
But they are not real VA and PA.
1. What are they indeed?
2. How does a program in VM access memory?

## GVA and GPA
The guest’s physical memory space is the memory the guest perceives as physical but, obviously,
it is inside QEMU’s process (host) virtual address.
Therefore, **`GPA is part of QEMU process VA`**.
In addition, GVA is faked virtual address space.

## Overview
QEMU’s memory management system is aware of where the guest physical memory space resides within its own memory space.
Therefore, it is able to translate guest physical addresses to host (QEMU’s) virtual addresses.

When a program insdie a VM wants to access a memory location,
it uses GVA, and the address translation procedure is below:
1. GVA -> GPA (guest page table)
2. GPA -> HPA (extended page table)

With [SLAT](https://en.wikipedia.org/wiki/Second_Level_Address_Translation) supportted,
**`both`** translations above are performed by **`MMU`** with using **Guest Page Table** and **EPT**.
The figure below shows the logic procedure of address translation.
- First, MMU traverses guest page tables to translate GVA to GPA;
- Then, MMUT traverses extended page tables to translate GPA to HPA (also known as Mathine Address, **MA**).

<img width="500" alt="MMU-VM" src="https://github.com/humasama/technologies/assets/6119088/9056bef7-7e99-486a-b557-f3400bf6bde5">

TLB stores **`GVA->MA (aka. HPA)`** translations to accelerate VM address translation.

<img width="500" alt="TLB-EPT" src="https://github.com/humasama/technologies/assets/6119088/46d47a52-7d5c-4e8f-bc5d-0465daa9a02d">

## Address Translation Procedure
As discussed above,
MMU needs to translate GVA to GPA, then GPA to HPA.
One VA to PA translation requires to lookup a 4 level page table,
so you might think GVA to HPA only needs to lookup 8 page tables.
But it is not true, as MMU doesn't know HPA of guest page tables.

### 1. GVA -> GPA
To translate GVA to GPA, MMU needs to look up 4 guest page tables.
Guest CR3 register keeps Page Map Level 4 (PML4) GPA.
The procedure is below (called **`Procedure_1`**, see figure below):
1. Offset(GVA) + **PML4 GPA** -> Page-Directory Pointer Table (PDPT) base GPA
2. Offset(GVA) + **PDPT GPA** -> Page Directory (PD) base GPA
3. Offset(GVA) + **PD GPA** -> Page Table (PT) base GPA
4. Offset(GVA) + **PT GPA** -> GPA

<img width="644" alt="Logic-GVA-GPA" src="https://github.com/humasama/technologies/assets/6119088/560ed52f-798e-469c-a0de-8a822c1ecdb6">

But the problem is that MMU cannot find guest page table in memory via GPA,
like PML4 cannot be found by GPA.
GPA to HPA map is stored in EPT.
To access a guest page table,
MMU must get HPA for the page table first.

### 2. Guest Page Table GPA -> HPA
GPA to HPA translation also requires to lookup a 4 level EPT.
VMCS keeps the EPT PML4 HPA.
The procedure is similar as GVA to GPA (in fact, it's the same as VA to PA in non-virtualization environment).
1. Offset(GPA) + **PML4 HPA** -> Page-Directory Pointer Table (PDPT) base HPA
2. Offset(GPA) + **PDPT HPA** -> Page Directory (PD) base HPA
3. Offset(GPA) + **PD HPA** -> Page Table (PT) base HPA
4. Offset(GPA) + **PT HPA** -> HPA

Therefore, every step in  **`Procedure_1`** require 4 EPT translations,
and total **`GVA to GPA`** generates **`16`** translations.

Here is a figure demonstrating the procedure above:

<img width="639" alt="EPT" src="https://github.com/humasama/technologies/assets/6119088/8376b68b-388f-4b58-b5e1-f6011981ca96">

### 3. GPA -> HPA
After getting GPA for GVA,
MMU searches 4-level EPT to get HPA like above.

## Cache
BTW, the figure below shows how CPU, TLB, page table and cache work together.
CPU operates VA, then TLB/page table translates it into PA, and cache stores data by PA.

<img width="490" alt="CPU-CACHE" src="https://github.com/humasama/technologies/assets/6119088/3eaf7d52-4fd7-4fb1-adfd-37c8caddf9d8">

## Reference
- Blog: https://revers.engineering/mmu-virtualization-via-intel-ept-index/
- Alibaba: https://www.alibabacloud.com/blog/599058
