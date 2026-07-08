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



---

## Phân tích động (Dynamic Analysis)
