> 详见 [SHA1与SM3算法文档](../文档/SHA-1算法与SM3算法.docx)

SHA (Security Hash Algorithm) 由美国标准与技术研究所 (NIST) 设计, 并于1993年作为 FIPS 180 发布, 该版本称为 SHA-0. 修订版 FIPS 180 于1995年发布, 称之为 SHA-1.

### Python实现

define utils: 
```python
from typing import List
import sys

# define some type
Int32 = int # 32bit int
Word = Int32
Int512 = int
BitLength = int
ByteLength = int

# 定义一些小原语
def _i2os(num, target_len: BitLength):
    return num.to_bytes(target_len//8, 'big')

def _os2i(bstr):
    return int.from_bytes(bstr, 'big')

def _rotate(x: Int32, base: int) -> Int32:
    base = base % 32
    return ((x << base) & 0xffffffff) | (x >> (32-base))

def _cat_words(words: List[Word]):
    tmp = 0
    for word in words:
        tmp <<= 32
        tmp += word
    return tmp
```

单组单轮 fk 函数:
![|300](../../attach/Pasted%20image%2020230915151333.png)
```python
# SHA-1 主体
IV = (0x67452301, 0xefcdab89, 0x98badcfe, 0x10325476, 0xc3d2e1f0)

def fk(b: Word, c: Word, d: Word, t: int):
    if t >= 0 and t <= 19:
        f = b & c | (b ^ 0xffffffff) & d  # 与1异或实现取反
        k = 0x5a827999
    elif t >= 20 and t <= 39:
        f = b ^ c ^ d
        k = 0x6ed9eba1
    elif t >= 40 and t <= 59:
        f = b & c | b & d | c & d
        k = 0x8f1bbcdc
    elif t >= 60 and t <= 79:
        f = b ^ c ^ d
        k = 0xca62c1d6
    else:
        raise ValueError('round t is out of range')

    return (f+k) & 0xffffffff
```

主体:
![|450](../../attach/Pasted%20image%2020230915151101.png)
单组处理, F 函数:
![|250](../../attach/Pasted%20image%2020230915151358.png)
填充方式:

假设消息 m 长度为 l bit. 填充时, 先在末尾添比特 1, 然后添 k 个比特 0. k 是满足: $l+1+k\equiv 448\pmod{512}$ 的最小非负整数. 之后, 将 l 表示为 64bits 长二进制添至末尾.

```python
def padding(msg: bytes):
    '''padding msg end to 512*k length'''

    m_len: ByteLength = len(msg)
    origin_len = m_len

    msg, m_len = msg+b'\x80', m_len+1
    padding_len = (56-m_len%64)%64
    msg += b'\x00' * padding_len
    msg += _i2os(origin_len*8, 64)

    return msg


def extend(block: Int512):
    '''split 16 words, and transform into 80 words'''

    w: List[Word] = [0] * 80
    # divide vector into words
    for j in range(16):
        w[15-j] = block & 0xffffffff
        block >>= 32

    for j in range(16, 80):
        w[j] = _rotate(w[j-3] ^ w[j-8] ^ w[j-14] ^ w[j-16], 1)

    return w


def round(reg: List[int], t: Word, w: Word):
    a, b, c, d, e = reg
    # 注意将f(b, c, d)+k 合并了
    reg[0] = (_rotate(a, 5) + fk(b, c, d, t) + e + w) & 0xffffffff
    reg[1] = a
    reg[2] = _rotate(b, 30)
    reg[3] = c
    reg[4] = d


def sha1(msg: bytes) -> bytes:
    # msg to 256 bit digest, 输入上限为..

    # 消息填充, 类型转化
    msg = padding(msg)
    blocks = [_os2i(msg[i: i+64]) for i in range(0, len(msg), 64)]

    reg = list(IV)
    for block in blocks:
        w = extend(block)
        last_reg = reg.copy() #!
        for i in range(80): # 每组 80 轮 fk
            round(reg, i, w[i])
        reg = [(x+y) & 0xffffffff for x, y in zip(reg, last_reg)]

    digest = _cat_words(reg)
    return _i2os(digest, 160)
```