# The Structure of Glibc Heap

> [!TIP]
> Tham khảo mã nguồn glibc tại đây:
> https://sourceware.org/git/?p=glibc.git;a=summary

## Dynamic Memory Allocator 

Mỗi khi chương trình của bạn cần cấp phát/giải phóng bộ nhớ động, nó thường sẽ sử dụng vùng nhớ **heap**.

Có nhiều thành phần tham gia vào một quy trình như cấp phát bộ nhớ động. Chúng ta có các khái niệm gọi là **arena**, **heap** và **chunk**.

* `Arenas`: Vùng nhớ thực tế được cấp phát cho một luồng được gọi là `arena` và có thể chứa nhiều vùng nhớ heap.
    * `Main Arena`: (dành cho thread chính) quản lý vùng heap chính của chương trình (khởi tạo qua `brk`). Vùng nhớ này có thể co giãn linh hoạt nên thông thường `Main Arena` không cần quản lý một danh sách gồm nhiều phân đoạn heap rời rạc giống như các Thread Arena khác.
    * `Thread Arenas` (dành cho các thread phụ) được tạo ra bằng `mmap`. Bộ nhớ của chúng được chia thành các phân đoạn nhỏ (sub-heaps). Khi một thread xài hết bộ nhớ trong sub-heap hiện tại, Arena của nó sẽ xin hệ điều hành thêm một sub-heap mới. Vì vậy, một `Thread Arena` thường sẽ quản lý nhiều heap rời rạc.

* `Heaps`: Đây được gọi là các vùng bộ nhớ liền kề, có thể được chia nhỏ thành nhiều "khối" (`chunk`). Một heap thuộc về một arena duy nhất.

* `Chunks`: Đây là những phần của vùng nhớ heap có thể được cấp phát, giải phóng, cấp phát lại, v.v. Hơn nữa, một khối (`chunk`) tồn tại trong một 1 vùng nhớ heap và thuộc về một 1 vùng nhớ arena.

---

## I. Arena

Trong thư viện C tiêu chuẩn (Glibc), bộ phận quản lý toàn bộ việc cấp phát và giải phóng bộ nhớ được gọi là **Arena**.

* Arena thực chất là một cấu trúc dữ liệu kiểu `struct malloc_state`. Nó giống như một "cuốn sổ hộ khẩu" ghi chép lại mọi thông tin về `heap` và các `bins`.

    > [!NOTE]
    > Trong mã nguồn gốc của Glibc (nằm tại file `malloc/malloc.c`), Arena được định nghĩa như sau:
    > 
    > ```c
    > typedef struct malloc_chunk* mchunkptr;
    > typedef struct malloc_chunk* mfastbinptr;
    > 
    > struct malloc_state {
    >     int mutex;                    /* Khóa bảo vệ đồng bộ luồng (Thread) */
    >     int flags;                    /* Cờ trạng thái nội bộ */
    >     int have_fastchunks;          /* Đánh dấu xem fastbin có rác không */
    >     
    >     mfastbinptr fastbinsY[10];    /* Mảng 10 Fastbins (danh sách liên kết đơn) */
    >     mchunkptr top;                /* Con trỏ trỏ tới vùng đất trống Top Chunk */
    >     mchunkptr last_remainder;     /* Chunk dư sau khi chia nhỏ bộ nhớ */
    >     
    >     mchunkptr bins[254];          /* Mảng chứa Unsorted, Small, Large Bins */
    >     unsigned int binmap[4];       /* Bản đồ bit hỗ trợ tìm kiếm nhanh */
    >     
    >     struct malloc_state *next;    /* Con trỏ nối các Arena (Circular Linked List) */
    >     struct malloc_state *next_free;
    >     INTERNAL_SIZE_T attached_threads;
    >     INTERNAL_SIZE_T system_mem;   /* Tổng bộ nhớ đã xin từ Hệ điều hành (OS) */
    >     INTERNAL_SIZE_T max_system_mem;
    > };

* Luồng chính (Main Thread) của chương trình sẽ sử dụng một Arena mặc định gọi là **`main_arena`**.
    * `main_arena` là một biến toàn cục (global variable) nằm trong **phân vùng DATA/BSS của thư viện `libc.so`**, hoàn toàn không nằm trên Heap. 
    * Nó là Arena duy nhất có thể sử dụng cơ chế `brk()` / `sbrk()` để mở rộng bộ nhớ liên tục từ hệ điều hành
    > **Ý nghĩa bảo mật**: Nếu trong quá trình pwn ta rò rỉ (leak) được địa chỉ của `main_arena`, ta sẽ tính được địa chỉ Base của Libc.
 
---

#### Chương trình demo

Nếu chỉ nói về mặt khái niệm sẽ rất khó để hình dung. Để dễ minh họa, chúng ta sẽ sử dụng chương trình C đơn giản sau:

```c
#include <stdlib.h>

int main(void){
    void *p = malloc(1);
    return 0;
}
```

Mặc dù việc không hề `free()` bộ nhớ sau khi sử dụng là việc sẽ dẫn đến lỗi bảo mật nhưng vì ở đây phục vụ bài test nên ta sẽ tạm bỏ qua vấn đề đó.

```ps
gcc chunk.c -o chunk
```

---
 
Quan sát thực tế `main arena` thông qua GDB:
 
 * `main_arena` trước khi gọi `malloc()`

   ![image](https://hackmd.io/_uploads/SJ1WCD-Wzg.png)


   * `main_arena.top = 0`:  Cho thấy `main_arena` chưa biết phân đoạn bộ nhớ của mình ở đâu.
   * `main_arena.system_mem = 0`: OS chưa hề cấp vùng nhớ heap cho process.
   * `main_arena.bins = {0x0,...}`: Bins chưa được khởi tạo.
   
   ---
   
   <br>
   
   
 * `main_arena` sau khi gọi `malloc()`
     
   ![image](https://hackmd.io/_uploads/By4H0wZbGe.png)



   * `main_arena.top = 0x612079c4a2b0`: Trỏ vào điểm bắt đầu của **Top Chunk**. (Top Chunk là gì thì sẽ giải thích ở phần Chunk)
    * `main_arena.system_mem = 135168`: OS đã cấp vùng nhớ heap cho process.
    * `main_arena.bins = {0x7dd205003b20 <main_arena+96>,...}`:
        * Glibc quản lý các `bins` dưới dạng danh sách liên kết vòng kép (**Doubly Linked List**).
        * Khi khởi tạo, do chưa có chunk bị giải phóng, glibc thiết lập cho các bin này tự trỏ ngược lại chính nó bên trong `main_arena` để tạo thành các vòng lặp trống khép kín, sẵn sàng hoạt động.

    
> [!NOTE]
> **Lưu ý**: Trong cấu trúc `main_arena` vẫn còn các trường khác như `fastbinsY`, `last_remainder` hay `mutex`... Tuy nhiên, do chương trình hiện tại hoạt động đơn luồng và chưa thực hiện giải phóng (free) bất kỳ bộ nhớ nào, các trường này đều đang ở trạng thái mặc định (bằng 0). Chúng ta sẽ tìm hiểu và quan sát sự biến đổi của chúng khi đi tìm hiểu cơ chế `free()`.
 

## II. Heap in Virtual Memory Layout

Khi nói đến heap trong Glibc ta có thể chia làm 2 loại:
    
**1. Main heap**
    
Dưới đây là vị trí của main Heap trong Virtual Memmory Layout của 1 tiến trình (process) trong Linux.

<img style="margin-left: 20%; width: 50%; height: 50%;" src="https://hackmd.io/_uploads/Sk8WwDAzMl.png">

<br>
<br>

**Đặc điểm**

* Heap nằm phía bên trên của phân vùng `.data`/`.bss` của chương trình.
* Khác với Stack được cấp phát tĩnh (khi chương trình chạy thì Stack đã được cấp phát xong), main Heap là vùng nhớ động, chỉ xuất hiện khi chúng ta gọi các hàm như `malloc()`.
* Địa chỉ của Heap tăng dần về phía địa chỉ cao (`0xfffffff...`).


    > [!NOTE]
    > **Minh họa**: Debug để thấy heap được cấp phát động
    >   * Trước khi gọi `malloc()`, chương trình chưa hề được cấp phát heap
    >
    >   ![image](https://hackmd.io/_uploads/HkbdwwAGzg.png)
    >
    >   * Khi gọi `malloc()`, `main_arena` thấy chưa có vùng heap nào để nó quản lý nên đã thông qua `sbrk()` để xin một vùng heap lớn. Lúc này heap đã được cấp phát 
    >
    >   ![image](https://hackmd.io/_uploads/ryTuvPAfzx.png)


**2. Sub-heaps**

**Đặc điểm**:

* Các block nhớ được tạo ra bằng `mmap()` (mỗi block mặc định là 64MB trên 64-bit)
* Mỗi block Sub-Heap này sẽ bắt đầu bằng cấu trúc `heap_info`
    
```c
typedef struct _heap_info
{
  mstate ar_ptr; /* Arena for this heap. */
  struct _heap_info *prev; /* Previous heap. */
  size_t size;   /* Current size in bytes. */
  size_t mprotect_size; /* Size in bytes that has been mprotected
                           PROT_READ|PROT_WRITE.  */
  size_t pagesize; /* Page size used when allocating the arena.  */
  /* Make sure the following data is properly aligned, particularly
     that sizeof (heap_info) + 2 * SIZE_SZ is a multiple of
     MALLOC_ALIGNMENT. */
  char pad[-3 * SIZE_SZ & MALLOC_ALIGN_MASK];
} heap_info;
```
    
> [!NOTE]
> Việc đi sâu vào sub-heaps lúc này là chưa cần thiết và dễ gây rối. Bạn hoàn toàn có thể tạm bỏ qua phần này để tập trung vào main Heap. Chúng ta sẽ quay lại tìm hiểu sâu về sub-heaps khi học về đa luồng.
    
---

## III. Chunks

---

Chunk là đơn vị quản lý bộ nhớ cơ bản của ptmalloc    

### Cấu trúc các loại chunk
    
#### 1. Allocated Chunk

Allocated chunk trong GLIBC là một khối bộ nhớ được cấp phát động khi chương trình sử dụng các hàm như `malloc()` hoặc `calloc()`
    
![image](https://hackmd.io/_uploads/HkvVm5RXGg.png)

    
Khi gọi `malloc()/calloc()` thì con trỏ `user_ptr` được trả về sẽ trỏ đến đầu của vùng **Userdata**.

Mỗi allocated chunk bao gồm hai phần chính trong bộ nhớ:
    
* **Metadata**: Nằm ngay ở đầu chunk. chứa các thông tin về:
    * `prev_size`: **Kích thước của previous chunk**. Chỉ có ý nghĩa khi `P = 0`. Nếu `P = 1`, trường này không được **glibc** sử dụng và phần bộ nhớ đó thuộc về vùng User Data của previous chunk.
    
    * `size`: **Kích thước của chunk hiện tại** (bao gồm metadata và User Data). Giá trị này luôn được căn chỉnh theo alignment của kiến trúc (8 bytes trên 32-bit, 16 bytes trên 64-bit)..
    * **Các cờ trạng thái** (`A | M | P`): 
        * `A` (`NON_MAIN_ARENA`): Xét chunk hiện tại. Bằng `0` nếu thuộc về **Main Arena**. Bằng `1` nếu thuộc về **Thread Arena** (các arena phụ).
        * `M` (`IS_MMAPPED`): Xét chunk hiện tại. Bằng `0` nếu chunk hiện tại nằm trong vùng Heap chính. Bằng `1` nếu chunk hiện tại được cấp phát trực tiếp bằng lệnh `mmap()` của OS thay vì lấy từ phân vùng Heap chính (`brk()`).
        * `P` (`PREV_INUSE`): Bằng `0` nếu chunk liền kề ngay phía trước đang trống (**free chunk**). Bằng `1` nếu previous chunk allocated.
> [!IMPORTANT]
> Điều đặc biệt là các cờ `A`, `M`, `P` được lưu chung trong trường `size` nhưng không làm thay đổi giá trị kích thước thực tế của chunk.
>
> Nguyên nhân là các chunk trong glibc đều phải căn chỉnh theo bội số của alignment tương ứng, dẫn tới việc 3 bit cuối cùng luôn luôn bằng `0`.
>
> Khi **glibc** cần biết chunk size, nó thực hiện một phép toán logic `AND` với một bitmask (có 3 bit cuối là `0` và tất cả các bit trên là `1`) để loại 3 bit cờ ra khỏi kết quả.
>
> * **Ví dụ**:
>
> ```bash
>  size (0x21):          0010 0001
>  bitmask    :      AND 1111 1000
>  -----------------------------------------
>  Kết quả:              0010 0000  => 0x20 (32 byte)

* **User Data**: Vùng nhớ thực tế mà chương trình ghi, đọc, và sử dụng. Đây chính là địa chỉ bộ nhớ được trả về cho lập trình viên.

> [!NOTE]
> Khi cấp phát vùng nhớ, **glibc** sẽ tự làm tròn để chunk size luôn chia hết cho alignment của từng kiến trúc (với x64 là 16 bytes).
> * **Ví dụ**:
>    Khi ta `malloc(1)` trên Linux x64, chunk size sẽ trông như thế này:
>    
>   ```bash
>   +-------------------+ <--- mchunkptr
>   |     prev_size     |  8
>   +-------------------+
>   |  size | A | M | P |  8
>   +-------------------+ <--- user_ptr
>   |                   |
>   |     User Data     | 
>   |                   |
>   +-------------------+
>   ```
>   Như vậy nếu User Data là `1`, tổng chunk size sẽ là: `8+8+1 = 17`. Do `17` không chia hết cho `16` nên **glibc** sẽ làm tròn kích thước chunk thành 32 byte (`0x20`). Sau khi trừ đi phần metadata 16 byte, vùng User Data khả dụng còn lại là 16 byte.
>   ```bash
>   +-------------------+ <--- mchunkptr
>   |     prev_size     |  8
>   +-------------------+
>   |  size | A | M | P |  8
>   +-------------------+ <--- user_ptr
>   |                   |
>   |     User Data     |  16
>   |                   |
>   +-------------------+
>   ```

---
    
#### 2. Free Chunks

**Free chunk** là trạng thái của một chunk sau khi chương trình trả lại cho hệ thống bằng cách gọi hàm `free(ptr)`. 
    
Thay vì lập tức trả vùng nhớ này cho OS, **glibc** sẽ biến nó thành một Free Chunk và đưa vào các danh sách liên kết (gọi là các **bins**) để tái cấp phát cho những lần `malloc` sau nhằm tối ưu hiệu năng.
    
![image](https://hackmd.io/_uploads/BJ9xQ5AXzx.png)

> [!NOTE]
> Sự thay đổi đáng chú ý nhất khi chuyển từ **allocated chunk** sang **free chunk** nằm ở nguyên lý **tái sử dụng không gian (Space Optimization)**:
>    
> Do chương trình không còn quyền sử dụng khối chunk này nữa, **glibc** lập tức "trưng dụng" chính vùng **User Data** cũ để lưu trữ các con trỏ quản lý danh sách liên kết(`fd`, `bk`).

Cấu trúc của một Free Chunk bao gồm 3 phần chính:

* **Metadata (Phần thông tin quản lý)**: Nằm ở 16 bytes đầu tiên của chunk, cấu trúc giống hệt Allocated Chunk:
    * **`prev_size`**: Khác với Allocated Chunk (nơi vùng này có thể bị lấn chiếm), đối với Free Chunk, trường này **luôn tồn tại**. Nếu chunk kề trước nó (previous chunk) cũng rảnh rỗi, trường này chứa kích thước của chunk trước đó để glibc có thể thực hiện **gộp lùi (coalesce backward)**.
    * **`size`**: Kích thước của toàn bộ chunk. Ba bit cuối cùng vẫn đóng vai trò là các cờ trạng thái `A | M | P` giống hệt Allocated Chunk. 
    
    $\rightarrow$ **Tuy nhiên**, cờ `P` của nextchunk (nếu có) sẽ phải tùy vào việc <u>**chunk hiện tại được xếp vào loại "bin" nào**</u> mà được set giá trị `0` hay `1` phù hợp.

* **Pointers (Các con trỏ liên kết)**: Nằm trên vùng `User Data` cũ. Tùy thuộc vào việc chunk nằm trong loại "**bin**" nào, glibc sẽ sử dụng 2 hoặc 4 con trỏ (thậm chí là chỉ 1 con trỏ đối với **Tcache** và **Fastbins**):
    * **`fd`** (Forward Pointer): Trỏ tới chunk rảnh rỗi **tiếp theo** trong cùng một Bin (hoặc trỏ về đầu danh sách ảo ở `main_arena` nếu nó là phần tử cuối cùng).
    * **`bk`** (Backward Pointer): Trỏ tới chunk rảnh rỗi **phía trước** trong cùng một Bin (hoặc trỏ về đầu danh sách ảo ở `main_arena` nếu nó là phần tử đầu tiên).
    * **`fd_nextsize`** & **`bk_nextsize`** (Chỉ dành cho Large Bins): Hỗ trợ nhảy nhanh giữa các dải kích thước khác nhau trong cùng một Large Bin.

* **Unused Space (Vùng trống còn lại)**: Nếu chunk có kích thước lớn, phần không gian còn lại sau khi ghi các con trỏ sẽ bị bỏ trống hoàn toàn cho đến khi được cấp phát lại.

> [!IMPORTANT]
> **Sự thật về vị trí của con trỏ `ptr` (Góc nhìn Developer vs Glibc)**
> 
> Có một sự hiểu lầm rất phổ biến trong các tài liệu trên mạng về vị trí của con trỏ khi giải phóng bộ nhớ. Thực tế, trong thế giới của Heap luôn tồn tại song song hai "lăng kính":
> 
> 1. **Góc nhìn của Glibc (`mchunkptr p`)**: Là con trỏ nội bộ của glibc, luôn trỏ vào `prev_size` để đọc các thông tin quản lý.
> 2. **Góc nhìn của lập trình viên (`void *ptr`)**: Là con trỏ trả về từ hàm `malloc` và truyền vào hàm `free`. Nó luôn trỏ tới đầu của vùng **Userdata**.
>
> ---
>
> * **Ví dụ trên kiến trúc 64-bit**:
>
>      Khi bạn gọi `free(ptr)`, Glibc lấy địa chỉ `ptr` ở Offset +16, tự động trừ đi 16 bytes (thông qua macro `mem2chunk()`) để tìm ngược lại vị trí của `mchunkptr p` và thực hiện giải phóng. Bản thân giá trị địa chỉ của biến `ptr` **không hề bị thay đổi hay dịch chuyển** sau lệnh `free`.

> [!NOTE]
> Hãy quan sát sơ đồ bộ nhớ 1 chương trình trên Linux x64 dưới đây để thấy rõ sự khác biệt giữa hai góc nhìn và cách vùng `User Data` bị trưng dụng thành con trỏ `fd` và `bk`:
>
> ```bash
>   Góc nhìn Glibc (Nội bộ)                      Góc nhìn Developer
> 
>  mchunkptr p ---> +-------------------------------+ ──► Offset +0
>                   |  prev_size                    | 
>                   +-------------------+---+---+---+ ──► Offset +8
>                   |  size             | A | M | P | 
>                   +-------------------+---+---+---+ <--- void *ptr = malloc(...) (Offset +16)
>                   |  fd pointer (Forward)         | 
>                   +-------------------------------+      (Giữ nguyên tại Offset +16)
>                   |  bk pointer (Backward)        | 
>                   +-------------------------------+
>                   |  ...                          |
>                   |  Unused Space                 |
>                   |  ...                          |
>                   +-------------------------------+
> ```
> 
> Chính vì `ptr` đứng im tại Offset +16, nên nếu sau khi `free()` mà developer vẫn cố tình đọc dữ liệu thông qua `ptr` (**lỗi Use-After-Free**), họ sẽ đọc trúng giá trị của con trỏ `fd`. 
>
> $\rightarrow$ Dữ liệu này thường chứa một địa chỉ Heap khác hoặc địa chỉ của `main_arena` (Libc), tạo tiền đề cho các kỹ thuật khai thác bộ nhớ cực kỳ nguy hiểm.
> 
> ---
>
> **Lưu ý**: Kể từ phiên bản **glibc 2.32** trở đi, cơ chế **safe linking** đã được áp dụng.
> 
> Địa chỉ của con trỏ `fd` trong `tcache` và `fastbins` sẽ được XOR với `address_of_fd_pointer >> 12`. 
> 
> Vậy nên giá trị đọc được từ `fd` sẽ không còn là một địa chỉ heap dạng raw pointer mà đã bị mã hóa bằng XOR.
    
---

#### 3. Top Chunk
    
**Top Chunk** là khối bộ nhớ nằm ở vị trí cuối cùng của phân vùng Heap, đại diện cho toàn bộ phần tài nguyên "chưa được khai phá"
    
Khi chương trình mới khởi chạy và gọi `malloc()` lần đầu tiên, tất cả "**bins**" đều trống rỗng, bộ nhớ của Heap lúc này thực chất chỉ là một khối Top Chunk khổng lồ. Ngay sau đó một phần bộ nhớ sẽ được chia sẻ lại để làm **Allocated Chunk** tùy vào số byte mà developer yêu cầu.
* **Trước khi bị cắt 1 phần cho Allocated Chunk**:
```
┌──────────────────┐
│                  │
│    TOP CHUNK     │
│                  │
└──────────────────┘ 
    
```

* **Sau khi bị cắt 1 phần cho Allocated Chunk**:
    
```
┌──────────────────┐
│  Allocated Chunk │
│       mới        │
│(Trả về cho user) │
├──────────────────┤
│                  │
│    TOP CHUNK     │
│   (Bị thu nhỏ)   │
│                  │
└──────────────────┘

```
    
> [!NOTE]
> **Top Chunk** không bao giờ bị xếp vào bất kì vùng "**bins**" nào.
    
<br>

**Cấu trúc của Top Chunk**:
    
Mặc dù là vùng nhớ "chưa được khai phá", **top chunk** vẫn được biểu diễn dưới dạng `struct malloc_chunk` trong source code của **glibc**.

    
```
(x64 Architecture):

mstate->top => ┌───────────────────────────────────────────────┐ ──► Offset +0
               |      prev_size (Không sử dụng)                │ 
     (top)     ├───────────────────────────────────┬───┬───┬───┤ ──► Offset +8 (M = 0, P = 1)
               │      size (Kích thước còn lại)    │ A │ 0 │ 1 │ 
               ├───────────────────────────────────┴───┴───┴───┤ ──► Offset +16
               │                                               │ 
               │                                               │
               │  VÙNG ĐẤT TRỐNG HOANG DÃ (Raw Free Space)     │
               │  (Bộ nhớ thô, không chứa fd và bk)            │
               │                                               │
               └───────────────────────────────────────────────┘ 
```
    
* **Chi tiết các cờ và trường dữ liệu**:
    * **Con trỏ quản lý nội bộ `mstate->top`**: là một con trỏ kiểu `mchunkptr` (`struct malloc_chunk*`) nằm trong cấu trúc của **arena** (cả 2 loại **main arena** và **thread arena** đều có loại con trỏ này)
    
    * **Trường `size` và các cờ quản lý trạng thái**:
        * `size`: Lưu phần memory còn lại của vùng heap ảo và glibc chưa phân chia.
        * `A`: `A = 0` nếu thuộc main arena, `A = 1` nếu thuộc thread arena.
        * `P = 1`: Luôn bật, nhằm đảm bảo **Top Chunk** tránh khỏi **coalesce backward**.
        * `M = 0`: Luôn tắt. Do **Top Chunk** thuộc phân vùng "chưa khai phá" (**Wilderness Chunk**) của main heap/sub-heap, nên không được cấp phát bằng `mmap()`, nên chắc chắn ở đây `M = 0`.
         
    * **Unallocated memory (còn gọi là Raw memory)**: 
        * Do **Top Chunk** không nằm trong bất kỳ một "**bins**" nào nên nó không cần các con trỏ liên kết `fd` và `bk`.
        * Không gian từ Offset +16 trở đi là vùng **raw memory** sẵn sàng để bị cắt ra khi cần.

> [!NOTE]
> Sau khi biết cấu trúc chi tiết của **Top Chunk**, quy trình một **Allocated Chunk** được cắt xén từ **Allocated Chunk** có thể được mô tả một cách rõ ràng hơn.
> ```
> Ví dụ với ptr = malloc(1) 
>
> Trạng thái 1: Trước khi Top Chunk được chia sẻ
> 
>                 top ──► ┌───────────────────────────────┐
>                         │  prev_size (0)                │ ──► Offset +0
>                         ├───────────────────────────────┤
>                         │  size: ___    | A | 0 | 1     │ ──► Offset +8 (Cờ P = 1)
>                         ├───────────────────────────────┤
>                         │                               │ ──► Offset +16
>                         │  RAW FREE SPACE               │
>                         │                               │
>                         └───────────────────────────────┘
> 
> 
> Trạng thái 2: Sau khi Top Chunk cắt xén 1 phần memory làm Allocated Chunk
> 
>                   p ───►┌───────────────────────────────┐ ──► Allocated Chunk mới (32 bytes)
>                         │  prev_size (0)                │      được cắt từ đầu Top Chunk
>                         ├───────────────────────────────┤
>                         │  size: 32      | A | 0 | 1    │
>                  ptr ──►├───────────────────────────────┤ ──► Trả về cho lập trình viên sử dụng
>                         │  User Data (Dữ liệu của bạn)  │
>                         |           ...                 |
>                  top ──►│───────────────────────────────┤ ──► Điểm bắt đầu của Top Chunk mới
>                         │  prev_size (0)                │      (Bị dịch xuống dưới 32 bytes)
>                         ├───────────────────────────────┤     
>                         │  size: ___     | A | 0 | 1    │
>                         ├───────────────────────────────┤
>                         │                               │
>                         │  RAW FREE SPACE còn lại       │
>                         └───────────────────────────────┘
> ```
    
    
 
    
    
    
    
## IV. Bins

Sau khi một chunk được giải phóng (`free()`), nếu nó không được gộp với chunk lân cận hoặc gộp vào Top Chunk, glibc sẽ đưa nó vào các danh sách liên kết gọi chung là "**bins**" để chờ tái cấp phát.    

### 1. Singly-Linked Lists (Nhóm danh sách liên kết đơn)

---

#### Tcache bins

* Được giới thiệu từ bản glibc 2.26, Tcache là một cơ chế bộ nhớ đệm (cache) độc lập dành riêng cho từng luồng (thread).

> [!NOTE]
> **Sơ lược về mục đích ra đời**: 
> * Trong chương trình đa luồng (multi-thread), mỗi khi nhiều luồng (threads) cùng gọi `malloc()` hoặc `free()` vào các **bins** thông thường, chúng phải tranh chấp khóa đồng bộ (`mutex`) của Arena để tránh xung đột dữ liệu
> * Việc các threads phải xếp hàng chờ nhả khóa `mutex` này làm giảm hiệu năng rất lớn.
> * **Tcache** ra đời để giải quyết triệt để vấn đề trên: Mỗi luồng được cấp một bộ đệm **Tcache** riêng biệt. Nhờ đó, threads có thể cấp phát và giải phóng các chunk nhỏ rất nhanh mà hoàn toàn không cần tranh chấp khóa arena.
>

* **Cấu trúc Tcache trong mã nguồn glibc**:
    
    ```c
    typedef struct tcache_entry
    {
        struct tcache_entry *next; /* Pointer trỏ trực tiếp tới User Data của chunk tiếp theo */
        
        uintptr_t key; /* This field exists to detect double frees.  */
    } tcache_entry;

    /* There is one of these for each thread, which contains the
    per-thread cache (hence "tcache_perthread_struct").  Keeping
    overall size low is mildly important.  Note that COUNTS and ENTRIES
    are redundant (we could have just counted the linked list each
    time), this is for performance reasons.  */

    typedef struct tcache_perthread_struct {
        uint16_t counts[TCACHE_MAX_BINS];
        tcache_entry *entries[TCACHE_MAX_BINS];
    } tcache_perthread_struct;
    ```
    
* **Đặc điểm của Tcache**:
    ```
    ========================================================================================================================
                                          BẢN ĐỒ BỘ NHỚ VÀ LIÊN KẾT TCACHE BINS
    ========================================================================================================================

        tcache_perthread_struct
             (Size: 0x290)
    ┌───────────────────────────────┐
    │ prev_size : 0x0000000000000000│  +0x00
    ├───────────────────────────────┤
    │ size      : 0x0000000000000291│  +0x08
    ├───────────────────────────────┤
    │ counts[64]                    │  +0x10
    │  [0] Bin 0x20 : 2             │
    │  [1] Bin 0x30 : 7 (MAX)       │
    │  ...                          │
    ├───────────────────────────────┤
    │ entries[64]                   │  +0x90
    │                               │              
    │                               │        ┌───────────────┐        ┌───────────────┐                                 
    │                               │        │   prev_size   │        │   prev_size   │                                      
    │                               │        ├───────────────┤        ├───────────────┤                              
    │                               │        │     size      │        │     size      │                             
    │                               │        ├───────────────┤        ├───────────────┤                                
    │  [0] Bin 0x20 ────────────────┼──────► │   next (fd)   ├───────►│   next (fd)   ├───────► NULL (0x00)         
    │                               │        ├───────────────┤        ├───────────────┤             
    │   (<= 24 bytes)               │        │ key = &tcache │        │ key = &tcache │                 
    │                               │        └───────────────┘        └───────────────┘                 
    │                               │        Chunk 1 (Vừa Free)       Chunk 2 (Free trước)                 
    │                               │        
    │                               │                                                                
    │                               │          
    │                               │       
    │                               │
    │                               │       ┌───────────────┐        ┌───────────────┐                 ┌───────────────┐   
    │                               │       │   prev_size   │        │   prev_size   │                 │   prev_size   │      
    │                               │       ├───────────────┤        ├───────────────┤                 ├───────────────┤  
    │                               │       │     size      │        │     size      │                 │     size      │ 
    │                               │       ├───────────────┤        ├───────────────┤                 ├───────────────┤ 
    │  [1] Bin 0x30 ────────────────┼──────►│   next (fd)   ├───────►│   next (fd)   ├───────► ... ───►│   next (fd)   ├───────► NULL (0x00) 
    │   (<= 40 bytes)               │       ├───────────────┤        ├───────────────┤                 ├───────────────┤ 
    │                               │       │ key = &tcache │        │ key = &tcache │                 │ key = &tcache │ 
    │                               │       └───────────────┘        └───────────────┘                 └───────────────┘ 
    │                               │       Chunk 1 (Vừa Free)       Chunk 2 (Free trước)              Chunk 7 (MAX)
    │                               │           
    │                               │       
    │                               │       
    │                               │       
    │                               │
    │  ...                          │
    │  [63] Bin 0x410 ──────────────┼──────► NULL (0x00)
    │                               │
    │   (<= 1032 bytes)             │
    └───────────────────────────────┘

    Chú thích:
    (*) (<= X bytes)  : Kích thước User Data tối đa mà Bin đó quản lý (Ví dụ: malloc(1..24) bytes sẽ chui vào Bin 0x20).
    (*) key = &tcache : Lưu địa chỉ của tcache_perthread_struct để phát hiện lỗi Double Free (Glibc >= 2.29).
    (*) next (fd)     : Từ Glibc >= 2.32, con trỏ next sẽ bị xáo trộn (mã hóa) bằng cơ chế Safe Linking thay vì lưu con trỏ thô.
    ```

    * **`tcache_perthread_struct`**:
        * Là cấu trúc dữ liệu quản lý trung tâm của hệ thống 64 **bins** trong  **Tcache** cho mỗi thread.
        * Cấu trúc `tcache_perthread_struct` về bản chất được glibc cấp phát động trên Heap dưới dạng 1 **Allocated Chunk** thay vì cấp phát tĩnh như **arena**.
        * Vị trí của Chunk chứa `tcache_perthread_struct` phụ thuộc vào vòng đời khởi tạo (Initialization Order) của thread:
            * `|>` Đối với Main Thread: Nó thường là Chunk đầu tiên nằm ngay tại Heap Base (được khởi tạo ngay trong lần gọi `malloc()` đầu tiên của luồng chính).
            * `|>` Đối với Sub-thread (Thread phụ): Nó nằm ngay phía sau cấu trúc metadata `heap_info` ở đầu vùng nhớ Sub-heap.
        * Số lượng và Kích thước: Tcache có tổng cộng 64 Bins. Trên hệ thống 64-bit, nó quản lý các chunk có kích thước từ 24 bytes đến 1032 bytes (bước nhảy 16 bytes).
        * Sức chứa tối đa: Mỗi một Tcache Bin chỉ chứa tối đa 7 chunks. Nếu một Bin đã đầy 7 chunks, các chunk bị giải phóng tiếp theo sẽ bị đẩy thẳng xuống các Bins bên dưới của Arena (như Fastbins hoặc Unsorted Bin).
        * Cơ chế hoạt động: Hoạt động theo cấu trúc Danh sách liên kết đơn (Singly-Linked List) và nguyên tắc LIFO (Last In, First Out - Vào sau ra trước). Chunk nào vừa free vào Tcache sẽ được malloc lấy ra đầu tiên.

    
    * **`tcache_entry`**:
        * Nói dễ hiểu có thể coi nó là struct đè lên vùng User Data của một Chunk khi nó bị `free()`.
        
        * **Cấu trúc**:
            * `next`: Trỏ sang User Data của Chunk tiếp theo trong **bin**. (Từ Glibc >= 2.32, con trỏ next sẽ bị xáo trộn (mã hóa) bằng cơ chế Safe Linking thay vì lưu con trỏ thô.)

            $\rightarrow$ Gọi là User Data, nhưng trong cái free chunk thì nó không được tính là User Data nữa, gọi vậy để dễ hiểu rằng nó trỏ đến cái vùng nằm sau Metadata thôi
            * `key`: Chứa địa chỉ `tcache_perthread_struct` để chống Double Free từ Glibc $\ge$ 2.29.
    
        * **Luồng vận hành**:

            Nhờ cơ chế **LIFO (Last-In, First-Out)**, luồng xử lý của Tcache Bins diễn ra rất trực quan dựa trên vị trí của các Chunk trong sơ đồ:

            1. **Khi giải phóng bộ nhớ (`free(ptr)`):**
               * Glibc tính toán size $\rightarrow$ xác định Bin (ví dụ Bin `0x20`).
               * Nếu `counts[bin] < 7`: Chunk mới bị free sẽ được đẩy lên làm **Chunk 1 (Vừa Free)** ở ngay đầu danh sách. 
               * Con trỏ `entries[0]` cập nhật trỏ vào **Chunk 1**, đồng thời `next` của Chunk 1 trỏ sang **Chunk 2 (Free trước)**.

            2. **Khi cấp phát bộ nhớ (`malloc(size)`):**
               * Glibc đổi request size sang Chunk size $\rightarrow$ xác định Bin tương ứng.
               * Nếu `counts[bin] > 0`: Glibc chỉ việc lấy ngay **Chunk 1 (Vừa Free)** đang nằm ở đầu danh sách ra để trả về cho người dùng (`return ptr`).
               * Bảng điều khiển `entries[bin]` lập tức cập nhật để trỏ thẳng sang **Chunk 2 (Free trước)**, biến Chunk 2 thành đại diện mới ở đầu danh sách.
    
> [!NOTE]
> Seriously, nhiều chữ thật, cũng chỉ để mô tả cho cái hình bên trên thôi, mà bỏ đi thì thiếu. Là mình thì mình chắc chắn lười đọc :))


---
    
### 2. Doubly-Linked Lists
    
    
    
    
References: 

* https://archive.crow.rip/nest/binexp/heap/the-heap
* https://azeria-labs.com/heap-exploitation-part-1-understanding-the-glibc-heap-implementation/
