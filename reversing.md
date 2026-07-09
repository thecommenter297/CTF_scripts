## Phân tích tĩnh (Static Analysis)

### I. ELF File Structure 

Việc học phân tích tĩnh trong reverse engineering trên Linux nên bắt đầu từ việc hiểu ELF file.

<img width="680" height="1135" alt="image" src="https://github.com/user-attachments/assets/e13ddedf-8b5d-48ed-a642-ec57fa2402cd" />

<!-- <img width="263" height="620" alt="image" src="https://github.com/user-attachments/assets/579135fc-4b3e-4a08-8d16-aca15de317f7" /> -->


---

#### `Elf64_Ehdr` (Executable Header)

* **Elf64_Ehdr** chính là một bản tóm tắt thông tin của cả file ELF. Bao gồm các thông tin như:
  * Chỉ định đây là file ELF, file ELF32 hay ELF64,... (file ELF bắt đầu với `\x7fELF`): `e_ident[16]`
  * Loại tệp ELF cụ thể: Là tệp thực thi, thư viện liên kết động, hay tệp định vị (relocatable file): `e_type`
  * Tệp đang chạy trên kiến trúc nào: `e_machine`
  * Địa chỉ ảo của entry point: `e_entry`
  * Vị trí của Program Header Table, Section Header Table: `e_phoff`, `e_shoff`
  * ...
    
  <img width="657" height="358" alt="image" src="https://github.com/user-attachments/assets/e6a7f869-dcf8-4de4-b063-600bcd17ce5c" />

> [!NOTE]
> **💡 Ghi nhớ:** Muốn biết **thông tin tổng quan của file ELF**, hãy xem **`Elf64_Ehdr`**.

> [!IMPORTANT]
> Không nhất thiết phải nhớ toàn bộ các trường trong **Elf64_Ehdr**. Điều quan trọng là biết **Elf64_Ehdr** là nơi chứa thông tin tổng quan của file ELF.
>  Khi cần xác định kiến trúc, Entry Point hay vị trí của các bảng header, đây là cấu trúc đầu tiên cần xem.
> Tương tự như vậy đối với các struct khác bắt đầu bằng **Elf64_**

#### `Elf64_Phdr` (Program Header)

* **Elf64_Phdr** mô tả thông tin của một **segment** trong file ELF. Mỗi `Elf64_Phdr` tương ứng với một entry trong **Program Header Table**, giúp hệ điều hành biết **segment nào cần được nạp vào bộ nhớ và nạp như thế nào**. Chứa các thông tin quan trọng như:

  * Loại segment (`PT_LOAD`, `PT_DYNAMIC`, `PT_INTERP`,...): `p_type`
  * Quyền truy cập của segment (Read, Write, Execute): `p_flags`
  * Vị trí (offset) của segment trong file ELF: `p_offset`
  * Địa chỉ ảo sẽ được nạp vào bộ nhớ: `p_vaddr`
  * Kích thước của segment trong file và trong bộ nhớ: `p_filesz`, `p_memsz`
  * ...

  <img width="497" height="191" alt="image" src="https://github.com/user-attachments/assets/0cad4a59-1d56-4f07-a541-4a6c66718faf" />


> [!NOTE]
> **💡 Ghi nhớ:** Muốn biết **kernel sẽ nạp chương trình vào bộ nhớ như thế nào**, hãy xem **`Elf64_Phdr`**.

---

#### `Sections`

* **Section** là nơi chứa **dữ liệu thực tế** của file ELF. Mỗi section đảm nhiệm một vai trò riêng, chẳng hạn chứa mã máy, dữ liệu, symbol hay thông tin phục vụ quá trình liên kết. Một số section thường gặp gồm:

* `.text`:  Chứa mã máy (machine code) của chương trình.

* `.rodata`: Chứa dữ liệu chỉ đọc (Read-Only Data), chẳng hạn chuỗi hằng (`"Hello World"`), hằng số,...

* `.data`: Chứa các biến toàn cục hoặc biến tĩnh đã được khởi tạo.

* `.bss`: Chứa các biến toàn cục hoặc biến tĩnh **chưa được khởi tạo** hoặc được khởi tạo bằng `0`. Section này **không chiếm dung lượng trong file ELF**, mà chỉ được cấp phát khi chương trình được nạp vào bộ nhớ.

* `.plt`: Chứa các stub (đoạn mã trung gian) dùng để gọi các hàm trong thư viện liên kết động.

* `.got`: Chứa **Global Offset Table (GOT)**, lưu địa chỉ thực của các hàm hoặc biến được liên kết động sau khi dynamic linker hoàn tất việc ánh xạ thư viện.

* `.interp`: Chứa đường dẫn tới **Dynamic Linker** (ví dụ: `/lib64/ld-linux-x86-64.so.2`).

* `.init`: Chứa đoạn mã được thực thi trước hàm `main()`.

* `.fini`: Chứa đoạn mã được thực thi trước khi chương trình kết thúc.

* `.shstrtab`: Chứa tên của tất cả các section trong file ELF.

 <img width="666" height="478" alt="image" src="https://github.com/user-attachments/assets/9623f370-61de-4356-ae2a-c48dfde52f68" />


> [!NOTE]
> **💡 Ghi nhớ:** Muốn biết **mã máy, dữ liệu hay các thông tin khác được lưu ở đâu trong file ELF**, hãy xem các **Sections**.


---

#### `Elf64_Shdr` (Section Header)

* **Elf64_Shdr** mô tả thông tin của một **section** trong file ELF. Mỗi `Elf64_Shdr` tương ứng với một entry trong **Section Header Table**, giúp các công cụ như `readelf`, `objdump`, linker,... biết **mỗi section được lưu ở đâu và có đặc điểm gì**. Chứa các thông tin quan trọng như:

  * Tên của section: `sh_name`
  * Loại section (`SHT_PROGBITS`, `SHT_SYMTAB`,...): `sh_type`
  * Thuộc tính của section (Read, Write, Execute,...): `sh_flags`
  * Địa chỉ ảo của section: `sh_addr`
  * Vị trí (offset) của section trong file ELF: `sh_offset`
  * Kích thước của section: `sh_size`
  * ...

  <img width="762" height="402" alt="image" src="https://github.com/user-attachments/assets/bc36ee2a-6e92-4298-9632-e25d9cb0377d" />


> [!NOTE]
> **💡 Ghi nhớ:** Muốn biết **một section nằm ở đâu hoặc có đặc điểm gì**, hãy xem **`Elf64_Shdr`**.

#### II. Bắt đầu reverse

Để dễ hình dung quy trình reverse cơ bản, ta sẽ đến với 1 file binary đơn giản.

https://github.com/thecommenter297/CTF_scripts/blob/main/demo

* Xem các thông tin của file thực thi:

```shell
kgf95983@ubuntu:~/train/test_binaries$ file demo

demo: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=c0e1a7b9fb34edaef92b2b97832c2c00f180fb21, for GNU/Linux 3.2.0, not stripped
```

* Xem các cơ chế bảo vệ của file thực thi:
```shell
kgf95983@ubuntu:~/train/test_binaries$ checksec demo
[*] '/home/kgf95983/train/test_binaries/demo'
    Arch:       amd64-64-little
    RELRO:      Full RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        PIE enabled
    SHSTK:      Enabled
    IBT:        Enabled
    Stripped:   No
```

**1. `DÙNG GHIDRA ĐỂ XEM PSEUDOCODE`**

<img width="1536" height="811" alt="image" src="https://github.com/user-attachments/assets/3ee173bd-3869-4ef4-9f2a-8da2ee2c4673" />
<br>

**2. `XEM XÉT CÁC HÀM TRONG FUNCTIONS`**:


* **Imports**: Gồm các hàm hoặc biến không được định nghĩa trong binary, mà sẽ được lấy từ thư viện ngoài khi chương trình chạy.
* **Functions**: Gồm tất cả các hàm mà Ghidra nhận diện được trong chương trình.
* **Exports**: Chủ yếu xuất hiện ở Shared Library (.so). Trong một executable bình thường thì gần như không cần quan tâm.
> [!NOTE]
> Trong Ghidra, mục Exports có thể chứa các symbol như _start, main hoặc các hàm do người dùng định nghĩa. Đây là cách Ghidra hỗ trợ điều hướng khi phân tích, không nên hiểu rằng tất cả các symbol này đều là "dynamic exports" theo chuẩn ELF.

* **Labels**: Tên mà Ghidra đặt cho một địa chỉ trong bộ nhớ.

=> Ở bước này chúng ta cần để ý đến các hàm trong **Functions**

<img width="285" height="465" alt="image" src="https://github.com/user-attachments/assets/94be48d6-1c3a-4025-8d71-5c715fba1971" />
<br>

**3. `XEM XÉT CÁC HÀM ĐƯỢC THỰC THI TRƯỚC HÀM main()`**

Ở đây cần phân biệt giữa 2 trường hợp:
* Trường hợp xét với file ELF bình thường: Luồng thực thi sẽ là từ entry point `_start` => `__libc_start_main` => `main()`
* Trường hợp khác:
  * Nếu là dynamic ELF: Kernel => `ld-linux.so` => Relocation => entry point `_start` => `main()`
  * Hoặc có thể các kĩ thuật anti-debug có thể được đặt ở `_start` => hoặc nếu đặt sau `_start` sẽ có thể nằm ở `.init` và các địa điểm khác trước hàm `main()`.
  * Thậm chí có thể có những packer có thể đổi luôn cả entry point. Lúc này entry point không còn là `_start` nữa, mà thứ tự thực hiện sẽ là entry point => `_start`

Chương trình trên rơi vào trường hợp file ELF bình thường. Luồng thực thi sẽ là

```rust
Kernel
    │
    ▼
Dynamic Linker (nếu có)
    │
    ▼
Entry Point (_start)
    │
    ▼
__libc_start_main()
    │
    ├── Khởi tạo libc
    ├── Chạy .preinit_array (nếu có)
    ├── Chạy .init
    ├── Chạy .init_array (constructors)
    │
    ▼
main()
    │
    ▼
exit()
    │
    ├── .fini_array (destructors)
    ├── .fini
    ▼
Kết thúc
```

Vậy chúng ta sẽ xem xét theo quy trình đã nêu để đảm bảo `_start` không làm điều gì ngoài dự đoán.

> [!NOTE]
> Ở mức độ beginner, chúng ta chưa cần phân tích sâu các startup code. Mục tiêu ở bước này chỉ là **đánh giá nhanh** xem hàm `_start` (và các hàm khởi tạo khác) có giống startup code thông thường hay không. Có thể quan sát theo các tiêu chí sau:
> 
> * **Hàm có đang thực hiện đúng vai trò của nó không?**
> 
>   Ví dụ, `_start` thường chỉ chuẩn bị môi trường thực thi rồi gọi `__libc_start_main()` để chuyển quyền điều khiển cho runtime. Nếu nội dung của hàm chủ yếu xoay quanh việc khởi tạo và không có logic riêng, đó là dấu hiệu bình thường.
> 
> * **Có lời gọi hàm nào đáng ngờ không?**
> 
>   Nếu startup code gọi các hàm không liên quan đến quá trình khởi tạo, hoặc gọi tới các hàm tự định nghĩa (`FUN_123`, `decrypt()`, `check_debugger()`...), nên dành thời gian kiểm tra kỹ hơn.
> 
> * **Có xuất hiện logic xử lý bất thường không?**
> 
>   Một vài nhánh điều kiện hoặc lệnh nhảy đơn giản chưa nói lên điều gì, vì compiler cũng sinh ra chúng. Tuy nhiên, nếu startup code chứa nhiều phép so sánh, vòng lặp hoặc logic xử lý phức tạp thì đó là dấu hiệu đáng để điều tra.
> 
> * **Có sử dụng các API hoặc syscall bất thường không?**
> 
>   Những lời gọi như `ptrace()`, `mprotect()`, `mmap()` hoặc syscall trực tiếp thường không xuất hiện trong startup code của một chương trình C đơn giản. Nếu gặp các lời gọi này, nên kiểm tra xem chúng phục vụ mục đích gì.
> 

Quan sát `_start` của chương trình hiện tại, ta thấy hàm chỉ chuẩn bị các tham số cần thiết rồi gọi `__libc_start_main()`. Không xuất hiện logic xử lý đáng chú ý hay các lời gọi hàm bất thường, vì vậy ở giai đoạn này ta có thể **tạm thời** coi startup code là bình thường và chuyển sang phân tích `main()`.

  <img width="1536" height="812" alt="image" src="https://github.com/user-attachments/assets/c4fca00d-1432-4e6b-b51e-7db813941745" />

Đối với `__libc_start_main()`, ở giai đoạn này chúng ta sẽ chưa đi sâu vào phân tích vì:

* Đây là hàm thuộc thư viện glibc, không phải mã nguồn do tác giả chương trình viết.
* Nhiệm vụ của hàm này là khởi tạo môi trường thực thi trước khi gọi `main()`. Cơ chế bên trong khá phức tạp và không cần thiết đối với các bài reverse cơ bản.

Tiếp theo, ta kiểm tra nhanh hàm `_init`.

<img width="1536" height="813" alt="image" src="https://github.com/user-attachments/assets/06db6bb1-e49e-40bd-8e18-4548edf36213" />
<br><br>

Quan sát cho thấy `_init` chỉ thực hiện công việc khởi tạo của runtime (`__gmon_start__`) và không chứa logic đáng chú ý của chương trình. Vì vậy, ở giai đoạn này ta có thể tạm thời bỏ qua và tiếp tục đến `main()`.

> [!WARNING]
> Trong quá trình phân tích, bạn sẽ thường gặp các hàm runtime do compiler hoặc linker sinh ra như `__gmon_start__`, `register_tm_clones`, `frame_dummy`, `__do_global_dtors_aux`...
>
> Nếu chưa biết chức năng của chúng, đừng cố ghi nhớ ngay. Hãy tra cứu vai trò của từng hàm khi cần. Điều quan trọng ở giai đoạn này là phân biệt được đâu là **runtime code** do compiler sinh ra và đâu là **logic của chương trình** do tác giả viết.
> 
> Khi tìm hiểu sâu hơn về C Runtime và glibc, chúng ta sẽ quay lại phân tích chi tiết các hàm này.

> [!IMPORTANT]
> Không tồn tại một quy tắc có thể xác định chính xác 100% startup code có an toàn hay không. Các tiêu chí trên chỉ giúp đánh giá nhanh đối với các chương trình C thông thường. Khi gặp malware, packer hoặc các challenge RE nâng cao, startup code hoàn toàn có thể bị chỉnh sửa và cần được phân tích kỹ như bất kỳ phần nào khác của chương trình.

**4. `QUAN SÁT HÀM main() VÀ ĐỌC HIỂU LUỒNG THỰC THI`**

  <img width="1536" height="815" alt="image" src="https://github.com/user-attachments/assets/1451ab18-0ebb-4e82-bc6a-103815097478" />

Trong đa số các chương trình C bình thường thì hàm `main()` là điểm khởi đầu phù hợp để tìm hiểu luồng xử lý của chương trình.

Khi tới bước này thì việc chọn các thanh công cụ nào để hiển thị trên Ghidra cũng là một điều cần lưu ý. Theo kinh nghiệm của mình, với ghidra thường nên có 4 cửa sổ cần thiết:
* **Decompile**: Hiển thị giả mã
* **Functions Call Trees**: Hiển thị 2 mục Incoming và Outgoing để biết hàm nào đã gọi hàm đang được xem, và hàm đang được xem sẽ gọi tới hàm nào. Điều quan trọng nhất là nhìn vào các tree này có thể **đoán được sơ bộ** luồng xử lý của từng đoạn chương trình.
* **Function Graph**: Dùng để hiển thị mã assembly trực quan, bổ trợ cho **Decompile**. Nếu mã giả bị đoán sai thì chỉ cần nhìn sang đây để dễ dàng Retype.
* **Listing**: Hiển thị lệnh assembly đầy đủ hơn **Function Graph**. Mặc dù thường thì **Function Graph** cũng chẳng thiếu bao nhiêu đâu, nhưng đôi lúc nó khuyết vài kí tự lệnh vì màn hình không đủ thì cũng khó chịu (-_-")

Về cơ bản thì sau khi bật các cửa sổ cần thiết lên nó sẽ kiểu như thế này:

<img width="1536" height="817" alt="image" src="https://github.com/user-attachments/assets/ec2df8bd-d4ff-4324-8dcf-2c58b035d9ae" />

**5. `ĐI VÀO PHÂN TÍCH CHI TIẾT LOGIC CỦA CHƯƠNG TRÌNH`**

Dựa vào **Function Call Trees** và Pseudocode ta có thể thấy quy trình của chương trình này là:

```bash
scanf() => check() xem điều kiện có khớp không => Nhảy đến win() nếu khớp
```

Hàm `check()` sẽ quyết định ta có thể đi đến `win()` hay không. Vậy ta sẽ đi kiểm tra `check()` có gì.

<img width="1536" height="808" alt="image" src="https://github.com/user-attachments/assets/ba2b7b0f-771c-4bda-958e-ad30b4e8faba" />

Có thể thấy, hàm `check()` thực hiện phép cộng `global_init` với 1237, để so sánh với số mà ta đã nhập vào. Nếu bằng nhau sẽ `win()`.

Biến `global_init` có giá trị là `64h` (Click chuột 2 lần để bảng **Listing** hiện giá trị của `global_init` vì nó là biến toàn cục được đặt ở `.data`)

Các bạn cũng có thể đổi dạng **Data** của `global_init` sang `unit` (unsigned int) và sau đó **Convert** dạng hiển thị sang số Decimal hệ 10 để dễ tính toán bằng cách nhấn chuột phải rồi tùy chọn.

  <img width="1535" height="808" alt="image" src="https://github.com/user-attachments/assets/da163c75-5417-4068-bf47-2d4bf1f400fd" />

**6. `THỰC HIỆN VIỆC LẤY FLAG`**

Đã biết được yêu cầu của chương trình, ta tiến hành điền số để lấy flag.

Ta có: 100 + 1237 = 1337

Vậy chỉ cần điền 1337 vào là ta sẽ thành công lấy Flag.

```bash
kgf95983@ubuntu:~/train/test_binaries$ ./demo < <(echo -ne '1337')
=== Simple Reverse Challenge ===
Input: FLAG{Congratulations!}
```

---

## Phân tích động (Dynamic Analysis)
