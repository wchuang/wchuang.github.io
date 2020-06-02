---
layout: post
title: "What Is Mmap and How It Works?"
date: 2018-06-18T17:17:35+08:00
categories: [Mmap]
---

# 前言 #

MMAP(Memory-mapped file), 是一種源於 Unix/Linux 作業系統內存映射文件的方法，主要用於**增加 I/O 存取效能**，尤其大文件效果更顯著。它在進程 process 中開闢一段**[虛擬內存](https://zh.wikipedia.org/wiki/%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98)***逐一字節的將一份文件或檔案從磁碟位置對應到該虛擬位置*，這樣使得 process 可以直接操作該虛擬內存，如同操作該文件，而不需要調用 read/write 等系統方法。同時也減少了一次從文件拷貝的動作。

# 原理 #

mmap 實現原理主要分為三個階段：

1. Start MMAP in process, creating a virtual mapping zone on virtual address space:

    - Process request a `mmap` system fuction in user-space:

        ``void *mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset);``

    - Seek a continuous and idle virtual address space in vitual memory address of current process.

    - Allocate a `vm_area_struct` struct for this virtual address space, and initialize the struct.

    - Insert the struct to linked list or tree of virtual address space of process.

2. Request a `mmap` system fuction in kernal-space, not in user-space, and implement the mapping relationship between the disk address of file and virtual address space of current process:

    - After allocating a new virtual address space, it will find the file descriptor in file descriptor table by the pointer of un-mapped files. And link to the struct file of every un-mapped file through the file descriptor in kernal's opened file set. Every struct file will maintain a opened file in kernal one by one.

    - Link to module of `file_operations` through the struct file and request function of `mmap` in kernal-space, not in user-space:

      ``int mmap(struct file *filp, struct vm_area_struct *vma)``
      
    - `mmap` function in kernal will locate the file physical address in disk by `inode` module.

    - Build a mapping relationship table between the disk address of file and virtual address space of current process. However, there is no data mapping in main memory from virtual address space.

3. Process start to read data on this virtual address space, that will occur [`Page Fault`](https://zh.wikipedia.org/wiki/%E9%A1%B5%E7%BC%BA%E5%A4%B1) and start copy document data from disk to main memory: (**In step 1 and 2, there is only create virtual address space and compelte the mapping relationshop table, but the data do not copy into main memory yet. When starting read/write operations on process, just start to copy data.**)

    - When process start to read or write data on mapping address on virtual address space, it find that it is not found on memory page. Due to there is only build a mapping and the file data is not copy to memory, so it occurs `Page Fault`.

    - Kernal request a page fault handler after confirming there are no illegal operations.

    - Seek to the memory page in `swap cache`, if not find, it will request `nopage` function in order to create page from disk to main memory.

    - Then the process can do read/write operations on main memory, if the data be changed, system will update those changed content in mapped disk address after a while. (You can use `msync` function to update data right now.)

# 函數介紹 #

`void *mmap(void *start, size_t length, int prot, int flags, int fd, off_t offset);`

##### 參數 #####

- start: mapping virtual address space 的開始位置，如果傳入 NULL，kernal 會自動選擇 page-aligned address。

- length: mapping space 長度。

- prot: 內存保護機制。

		PROT_EXEC  Pages may be executed.
		PROT_READ  Pages may be read.
		PROT_WRITE Pages may be written.
		PROT_NONE  Pages may not be accessed.
    
- flags: 是否該映射區可以被其他 processes 看到。
		
		MAP_SHARED Share this mapping.
		MAP_SHARED_VALIDATE This flag provides the same behavior as MAP_SHARED except that MAP_SHARED mappings ignore unknown flags in flags.
		MAP_PRIVATE Create a private copy-on-write mapping.
		MAP_32BIT Put the mapping into the first 2 Gigabytes of the process address space. 
		MAP_ANON Synonym for MAP_ANONYMOUS. Deprecated.
		...    

- fd: 文件描述詞。

- offset: 文件位移量。

##### 返回值 #####

如果建立成功，會返回一個指標指向這個映射空間的位置。假設失敗，將會回傳 `MAP_FAILED (that is, (void *) -1)`，以及 `errno`。

# 與一般文件存取差異 #

一般文件調用過程：

1. Process 發起讀取文件請求。
2. Kernal 通過尋找 file descriptor in file descriptor table，並且利用 kernal 中已開啟的文件集中的文件資訊找到該文件的 `inode`。
3. 判斷該文件是否在[頁緩存 page cache](https://en.wikipedia.org/wiki/Page_cache) 中，如果存在則返回該文件內容。不存在則利用 `inode` 定位到 physical address in disk，並拷貝文件內容到頁緩存中。之後再發起讀頁面請求，將頁緩存的資訊給該 process。

總而言之，一般文件讀取為了提高效率及保護磁碟，會利用頁緩存技術。這樣造成讀文件的時候，需要先將文件從磁碟拷貝到頁緩存，但由於頁緩存在 kernal-space 中，不能被 user-space 訪問，所以必須將資料從頁緩存再拷貝到 user-space。**這樣進行了兩次的資料拷貝**。

而使用 mmap，利用建立新的 virtual address space 以及磁碟位置與 virtual address space 的映射，沒有進行文件拷貝。直到訪問資料時造成 `Page Fault` 利用映射關係，**只進行一次的資料拷貝**，就從磁碟拷貝至 user-space，讓 process 得以使用。

# 優點 #

1. 對文件存取繞過了 paging 技術操作，減少了資料的拷貝次數，並且利用 read/write in memory instead of I/O operations，大大提高效能。
2. 內存映射文件可以只載入一部分內容到用戶空間，對於大型檔案非常有用。
3. 實現了 user-space 及 kernal-space 的交互，兩空間的修改可以直接反映在映射區，並且被對方更新。
4. 可以達到跨 process 溝通通信。
5. 避免因為大量的 data I/O 造成的記憶體不足問題，

# 如何確保不會讀到其他位址 #

MMAP 是利用自己的虛擬位址空間來處理 (必須先映射，否則無法透過 user-space 的虛擬記憶體存取)。所以其他 user-space 中的行程是沒辦法去存取這塊記憶體。因為其這塊超出的記憶體位址並沒有映射到它自己的虛擬記憶體位址，也因此當 user-space 存取超出的話，就會被作業系統踢出。

# 參考資料 #
- [Wiki](https://zh.wikipedia.org/wiki/%E5%86%85%E5%AD%98%E6%98%A0%E5%B0%84%E6%96%87%E4%BB%B6)
- [mmap分析](http://www.cnblogs.com/huxiao-tee/p/4660352.html)
- [mmap函數說明](http://man7.org/linux/man-pages/man2/mmap.2.html)

