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

  <img width="1536" height="812" alt="image" src="https://github.com/user-attachments/assets/c4fca00d-1432-4e6b-b51e-7db813941745" />

Ở mức độ beginner, chúng ta sẽ chưa học các kĩ thuật nâng cao, vì vậy có 4 dấu hiệu cần để ý nhằm xác định liệu một hàm `_start` là an toàn và có thể lướt qua:
* Có code gọi hàm `__libc_start_main()` không: Là các đoạn `lea rdi, [main]; call __libc_start_main`. Đây gần như chắc chắn là các đoạn do compiler sinh ra.
* Có lệnh `call` nào đến các hàm lạ không.
* Có các nhánh điều kiện kì lạ nào không: `cmp eax, 1; jne LAB_1234`. Nếu có dấu hiệu như so sánh, loop,... thì khả năng đó không phải là hàm `_start` đơn giản nữa.
* Có đoạn gọi `syscall` hay tồn tại các hàm nào như `ptrace()`, `mprotect()`, `mmap()` không.



---

## Phân tích động (Dynamic Analysis)
