# Dubblesort (pwnable.tw)



> Sort the memory!
>
> `nc chall.pwnable.tw 10101`
>
> [dubblesort](https://github.com/th3-5had0w/writeups/raw/main/dubblesort%5Bpwnable.tw%5D/dubblesort)
>
> [libc.so](https://github.com/th3-5had0w/writeups/raw/main/dubblesort%5Bpwnable.tw%5D/libc_32.so.6)

[Phiên bản libc local (Ubuntu 20.04)](https://github.com/th3-5had0w/writeups/raw/main/dubblesort%5Bpwnable.tw%5D/local_libc.so.6)

## Những đoạn code cần lưu ý
![pic1](https://github.com/th3-5had0w/writeups/blob/main/dubblesort%5Bpwnable.tw%5D/res/pic_1.png)

#### Ở đây chương trình đọc input vào nhưng không chặn chuỗi bằng nullbyte (`\x00`) => Ta có thể nhập các kí tự để hàm printf leak được địa chỉ của libc.

![pic3](https://github.com/th3-5had0w/writeups/blob/main/dubblesort%5Bpwnable.tw%5D/res/pic_3.png)

#### Mình đã nhập 8 chữ `a` kèm theo kí tự enter `\x0a`, và như ảnh trên thì có thể thấy địa chỉ chỉ libc rất gần với nơi ta nhập input. (1)
#### Ngoài ra kí tự enter của mình đã ghi đè lên byte nullbyte (`\x00`) của libc, mà printf sẽ in các bytes ra màn hình cho đến khi gặp kí tự nullbyte (`\x00`) (2)

#### Từ (1) và (2) => Ta có thể leak được libc.
