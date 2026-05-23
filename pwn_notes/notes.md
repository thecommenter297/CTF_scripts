<h3 style="text-align: center;">Pwn notes</h3>

Bài viết của beginner, văn phong lủng củng, public chỉ vì muốn dễ xem lại, mong đừng ai thấy

---
## 1.Steps để giải pwn
**Kết nối đến pwn.college bằng `ssh`**
```shell
ssh -i key hacker@dojo.pwn.college
```

**Cách tải file từ pwn.college về máy**
> Dùng lệnh này khi đã ssh đến server
```shell
scp -i <đường dẫn đến file key> hacker@dojo.pwn.college:<link_file> .
```
#### Giải nén:
```shell
unzip file.zip
```

```shell
tar -xJvf ten_file.tar.xz
```

##### Patch file libcapstone.so.5 và libc trên pwn.college
```bash
patchelf --set-interpreter ./ld-linux-x86-64.so.2 ./ello-ackers

patchelf --set-rpath '$ORIGIN' ./ello-ackers
```

**Cài môi trường mà challenge yêu cầu (ví dụ docker)**
<details>
<summary>Setup và chạy docker</summary>
    
```shell
$ sudo apt install docker.io #nếu bạn chưa cài docker
$ sudo docker build -t <tên_challenge> .
$ sudo docker run -p 1337:1337 <tên_challenge> #port này các bạn xem trong file docker nhé
```
</details>

**Kiểm tra các cơ chế bảo vệ của chương trình**
<details>
<summary>Gõ lệnh checksec để xem cơ chế bảo vệ</summary>

```shell
$ checksec --file=<binary>
```

- Ý nghĩa các mục hiện ra:

1. RELRO (Relocation Read-Only)    

| Kiểu              | Ý nghĩa                                     | Ảnh hưởng                                              |
| ----------------- | ------------------------------------------- | ------------------------------------------------------ |
| **No RELRO**      | GOT writable                                | Dễ overwrite GOT → dễ ret2got                          |
| **Partial RELRO** | GOT writable nhưng `.got.plt` vẫn có thứ tự | Vẫn overwrite GOT được                                 |
| **Full RELRO**    | GOT read-only                               | **Không overwrite GOT được**, phải dùng technique khác |

2. STACK CANARY

| Trạng thái   | Nghĩa                            | Ảnh hưởng                                   |
| ------------ | -------------------------------- | ------------------------------------------- |
| **Enabled**  | Có giá trị ngẫu nhiên giữa stack | Chống stack overflow đơn giản               |
| **Disabled** | Không có                         | Overflow → overwrite return address dễ dàng |

3. NX (No-eXecute bit)

| Trạng thái   | Nghĩa                | Ảnh hưởng                                               |
| ------------ | -------------------- | ------------------------------------------------------- |
| **Enabled**  | Stack không thực thi | Không sử dụng shellcode trực tiếp trên stack → dùng ROP |
| **Disabled** | Stack executable     | Shellcode chạy trực tiếp (ret2shellcode)                |

4. PIE (Position Independent Executable)

| Trạng thái      | Nghĩa                             | Ảnh hưởng                                     |
| --------------- | --------------------------------- | --------------------------------------------- |
| **No PIE**      | Base address cố định              | `.text` không random → gadgets/function fixed |
| **PIE enabled** | Base address random (ASLR đầy đủ) | Address thay đổi mỗi run, cần leak hoặc brute |

5. RPATH / RUNPATH
    
| Trạng thái    | Nghĩa                                                  | Ảnh hưởng                                             |
| ------------- | ------------------------------------------------------ | ----------------------------------------------------- |
| **Present**   | Chương trình load shared library từ đường dẫn tùy biến | Có thể ***hijack library*** nếu bạn kiểm soát thư mục |
| **Not found** | Không có                                               | Bình thường                                           |

6. FORTIFY_SOURCE

| Trạng thái   | Nghĩa                                                      | Ảnh hưởng                                         |
| ------------ | ---------------------------------------------------------- | ------------------------------------------------- |
| **Enabled**  | Compiler dùng hàm safe như `__chk_fail`, kiểm tra overflow | Một số overflow không khai thác được theo cách cũ |
| **Disabled** | Không có                                                   | Bỏ qua                                            |

7.SECURITY HARDENING (CFI / Shadow Stack / IBT)
    
| Cơ chế                             | Ý nghĩa                              | Ảnh hưởng                              |
| ---------------------------------- | ------------------------------------ | -------------------------------------- |
| **Shadow Stack**                   | Lưu return address riêng biệt        | Không làm ret2ret hay ret2reg          |
| **IBT (Indirect Branch Tracking)** | Chống call/jump vào giữa instruction | Chống ROP dạng jump ret bất thường     |
| **CFI**                            | Control Flow Integrity               | ROP gần như vô dụng nếu bật hoàn chỉnh |

</details>
    
**Chạy thử chương trình**

<details>
<summary>Chạy thử để xem hành vi chương trình</summary>

```shell
$ ./<binary_file>
```

Có thể chạy rồi điền input, xem chương trình có crash không, hoặc đưa các input kì lạ như `%p`, `%llx`,... để thử xem có lỗi overflow/format string không.
</details>
    
**Phân tích logic chương trình**
<details>
    <summary>Sử dụng IDA để xem logic / mở source code nếu có sẵn</summary>
    <ul>
    <li> Nếu chương trình có sẵn file source code thì sẽ xác định điểm gây lỗi</li>
    <li>Nếu chỉ có file binary thì dùng IDA để xem pseudocode rồi tìm điểm gây lỗi</li>
    </ul>
    <hr>
    (Ví dụ 1 file binary được đưa vào IDA)
    <img src="https://hackmd.io/_uploads/SyZv1T6gbx.png">
    Một vài lưu ý để không bị loạn:
    <!-- <ul>
        <li>Bỏ qua các label như sub_1020, sub_1030,...</li>
        <li>Bỏ qua các label in đậm như <b>__cxa_finalize</b>, <b>_write</b>,...</li>
        <li>Tập trung vào các label như start_routine, main, còn label start thì hên xui (vẫn nên xem qua liệu start được compiler tự sinh ra hay được tùy chỉnh bởi người ra đề)</li>
    </ul> -->
    Tiếp theo bấm F5 hoặc Fn + F5 để xem pseudocode, xác định nơi có input, output, chú ý đến những chỗ có gets, read, printf,malloc, free,...
    <img src="https://hackmd.io/_uploads/rJ9ErTpebg.png">
    Sau khi xác định nơi có input/output, đánh giá xem nó thuộc loại lỗi gì, ví dụ như buffer overflow, hoặc cái loại lỗi khác... Nếu vẫn chưa biết hàm input/output đó có khả năng gây lỗi không thì có thể search google
</details>

**Google/tra cứu các hàm/lệnh chưa biết**
    
**Xác định hướng khai thác dựa trên hàm/lệnh có lỗi**
    
**Cài seccomp-tools để phân tích seccomp profile**
```shell
sudo apt install ruby-dev gcc make
sudo gem install seccomp-tools
```
Chạy seccomp-tools:
```shell
seccomp-tools dump ./<binary>
```
    
---
## 2. notes
**Syscalls**
Xem các syscalls tại đây:
https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md#x86-32_bit
**File permissions**
Ví dụ với lệnh `open()` trong C trên Linux
```c
open("/link/to/file", <FILE_FLAGS>, <MODE>);
```
    
<details>
    <summary>File flags - Cách mở file</summary>
    
| Hằng số    | Decimal | Hex      | Octal   | Ý nghĩa / Symbolic                     |
| ---------- | ------- | -------- | ------- | -------------------------------------- |
| O_RDONLY   | 0       | 0x0      | 000000  | Chỉ đọc file                           |
| O_WRONLY   | 1       | 0x1      | 000001  | Chỉ ghi file                           |
| O_RDWR     | 2       | 0x2      | 000002  | Đọc & ghi file                         |
| O_CREAT    | 64      | 0x40     | 0100    | Tạo file nếu chưa tồn tại              |
| O_EXCL     | 128     | 0x80     | 0200    | Kết hợp O_CREAT → lỗi nếu file tồn tại |
| O_NOCTTY   | 256     | 0x100    | 0400    | Không gán terminal làm controlling tty |
| O_TRUNC    | 512     | 0x200    | 01000   | Xóa nội dung file nếu tồn tại          |
| O_APPEND   | 1024    | 0x400    | 02000   | Ghi vào cuối file                      |
| O_NONBLOCK | 2048    | 0x800    | 04000   | Non-blocking I/O                       |
| O_DSYNC    | 4096    | 0x1000   | 010000  | Đồng bộ data write                     |
| O_SYNC     | 1052672 | 0x101000 | 0204000 | Đồng bộ data + metadata                |

</details>

<details>
    <summary>Mode – quyền file khi tạo (open(..., mode))</summary>
    
Một file trong Linux có 12 bit để định nghĩa quyền:
```shell
[special bits] [owner] [group] [others]
```

| Phần    | Số bit | Ý nghĩa                |
| ------- | ------ | ---------------------- |
| special | 3      | setuid, setgid, sticky |
| owner   | 3      | rwx của chủ sở hữu     |
| group   | 3      | rwx của nhóm           |
| others  | 3      | rwx của người khác     |

Bảng tổng hợp MODE
    
| Mode (octal) | Symbolic    | Decimal | Ý nghĩa                                              |
| ------------ | ----------- | ------- | ---------------------------------------------------- |
| 0000         | --- --- --- | 0       | Không quyền gì                                       |
| 0400         | r-- --- --- | 256     | Chỉ owner đọc                                        |
| 0200         | -w- --- --- | 128     | Chỉ owner ghi                                        |
| 0100         | --x --- --- | 64      | Chỉ owner execute                                    |
| 0600         | rw- --- --- | 384     | Owner đọc & ghi                                      |
| 0644         | rw- r-- r-- | 420     | Owner đọc/ghi, group & others đọc                    |
| 0664         | rw- rw- r-- | 436     | Owner/group đọc/ghi, others đọc                      |
| 0755         | rwx r-x r-x | 493     | Owner read/write/execute, group/others đọc & execute |
| 0777         | rwx rwx rwx | 511     | Full quyền tất cả                                    |                                                           |
| 04000      | s--- --- --- |  | setuid: Khi chạy file, dùng quyền **owner**                                |
| 02000      | -s-- --- --- |  | setgid: Khi chạy file, dùng quyền **group**                                |
| 01000      | --t- --- --- |  | sticky: Thường dùng ở thư mục `/tmp` để người khác không xóa file của nhau |
| 04755        | rws r-x r-x | 2517    | setuid + 0755                                        |
| 02755        | r-x r-s r-x | 1469    | setgid + 0755                                        |
| 01777        | rwx rwx rwt | 1023    | sticky bit + 0777 (thường cho /tmp)                  |

Ví dụ: Octal: 0755
Tách ra là:
```c
    0    |   7   |   5   |   5

 special | owner | group | others
```
    
* special bit: Tắt hết
* 7 (owner) = 111 (bin) → rwx
* 5 (group) = 101 → r-x
* 5 (others) = 101 → r-x

Nếu còn thắc mắc là chỉ có nhiều nhất 9 chữ cái được in ra, ví dụ
```c 
-rwxrwxrwx
```
thì cái special kia đi đâu. Thì special bit được thể hiện luôn ở chữ cái thứ 3
Ví dụ:
| Octal | Symbolic    | Giải thích                          |
| ----- | ----------- | ----------------------------------- |
| 0777  | rwx rwx rwx | Special bits = 0, tất cả rwx đầy đủ |
| 4777  | rws rwx rwx | setuid bật → owner execute = s      |
| 2777  | rwx rws rwx | setgid bật → group execute = s      |
| 1777  | rwx rwx rwt | sticky bật → others execute = t     |
    
Nguyên tắc:

* Symbolic chỉ 9 ký tự rwx bình thường + special bits “đè lên execute”

* Nếu execute bit tắt (-) nhưng special bit bật, sẽ hiển thị S hoặc T (viết hoa):

| Symbolic | Ý nghĩa                                         |
| -------- | ----------------------------------------------- |
| S        | setuid/setgid bật nhưng execute owner/group = 0 |
| T        | sticky bật nhưng execute others = 0             |

Ví dụ:
```c
Octal: 4600 → setuid + owner rw- → Symbolic: rwS --- ---
```
</details>

**Patch libc**
<details>
    <summary>Steps</summary>
* Cài patchef:
    
```shell
cd /usr/local/bin
```

```shell
sudo apt install patchelf
```
```shell
cd <CTF_directory>
```

* Nhớ thêm quyền cho ./<chall_patched> và ld-linux-x86-64.so.2
    
```shell
sudo chmod +x ./<chall_patched>
sudo chmod +x ld-linux-x86-64.so.2 
```

* Chạy patchelf
```shell
patchelf --set-interpreter ./ld-linux-x86-64.so.2 --set-rpath . ./<chall_patched>
```

</details>
    
---
## 3. pwn tricks

* Nhận dữ liệu, chuyển từ byte sang int
```python
value = int.from_bytes(data, byteorder='little')
print(hex(value))
```

* Tra cứu các variable defined trong thư viện:
Tìm theo từ khóa "tên thư viện của hàm" + "github"
(Vì mã nguồn của C được public trên github)
