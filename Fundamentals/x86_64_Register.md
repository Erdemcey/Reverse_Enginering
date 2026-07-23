# x86-64 Registers: CPU'nun Çalışma Masası

> **Amaç:** `register` yapısını, `partial register` kullanımını, `mov` instruction'ını, `zero-extension`, `sign extension` ve temel register işlemlerini günlük bir anlatımla öğrenmek.

---

## İçindekiler

- [1. Register nedir?](#1-register-nedir)
- [2. Register File ve data flow](#2-register-file-ve-data-flow)
- [3. Register boyutları](#3-register-boyutları)
- [4. RAX, EAX, AX, AH ve AL](#4-rax-eax-ax-ah-ve-al)
- [5. Partial Register kullanımı](#5-partial-register-kullanımı)
- [6. MOV instruction](#6-mov-instruction)
- [7. Immediate Value](#7-immediate-value)
- [8. MOV neden kopyalama gibi çalışır?](#8-mov-neden-kopyalama-gibi-çalışır)
- [9. Operand Size](#9-operand-size)
- [10. EAX'e yazınca RAX neden değişir?](#10-eaxe-yazınca-rax-neden-değişir)
- [11. Zero-Extension](#11-zero-extension)
- [12. Signed ve Unsigned yorum](#12-signed-ve-unsigned-yorum)
- [13. Two's Complement](#13-twos-complement)
- [14. Sign Extension](#14-sign-extension)
- [15. MOVSX ve MOVSXD](#15-movsx-ve-movsxd)
- [16. Arithmetic Operations](#16-arithmetic-operations)
- [17. Bitwise ve Shift Operations](#17-bitwise-ve-shift-operations)
- [18. XCHG](#18-xchg)
- [19. Special Registers](#19-special-registers)
- [20. RIP](#20-rip)
- [21. RSP](#21-rsp)
- [22. GDB ile register inceleme](#22-gdb-ile-register-inceleme)
- [23. Yaygın hatalar](#23-yaygın-hatalar)
- [24. Mini alıştırmalar](#24-mini-alıştırmalar)
- [25. Cevap anahtarı](#25-cevap-anahtarı)
- [26. English Term Glossary](#26-english-term-glossary)

---

## 1. Register nedir?

CPU sürekli `data` işler. Bu data bazen bir sayı, bazen bir `memory address`, bazen bir `function parameter`, bazen de önceki işlemin sonucu olabilir.

CPU'nun üzerinde çalışacağı datayı yakınında tutması gerekir. Bu küçük ve hızlı alanlara `register` denir.

> `Register`, CPU'nun içinde bulunan çok hızlı ve küçük bir data storage alanıdır.

Bunu çalışma masasına benzetebiliriz:

```text
Disk        → Kitaplık
Memory      → Odanın içindeki raf
Cache       → Masanın çekmecesi
Register    → Masanın üstü
```

O anda kullandığın şey masanın üstündedir. Diğer şeyleri gerektiğinde raftan veya çekmeceden alırsın. CPU da aktif datayı register içinde tutar.

### Register neden hızlıdır?

Çünkü doğrudan CPU'nun içindedir.

Genel hız sırası:

```text
Register
   ↓
Cache
   ↓
Memory
   ↓
Disk
   ↓
Network
```

Yukarı çıktıkça hız artar, kapasite küçülür ve maliyet yükselir. Bu yüzden register'lar çok hızlıdır fakat sayıları sınırlıdır.

---

## 2. Register File ve data flow

CPU içinde birden fazla register bulunur. Bu register grubunun tamamına `register file` denir.

```text
CPU
├── Control Unit
├── ALU
├── Cache
└── Register File
    ├── RAX
    ├── RBX
    ├── RCX
    ├── RDX
    ├── RSI
    ├── RDI
    ├── RSP
    ├── RBP
    └── ...
```

Data register'a üç temel yerden gelebilir:

### Instruction içinden

```asm
mov rax, 10
```

### Başka bir register'dan

```asm
mov rbx, rax
```

### Memory'den

```asm
mov rax, [rbx]
```

Basit data flow:

```text
Instruction ───────────────┐
                           │
Memory → Cache → Register File → CPU Operation
                           │
Another Register ──────────┘
```

---

## 3. Register boyutları

Modern x86-64 sistemlerde genel amaçlı register'ların büyük bölümü 64-bit genişliğindedir.

```text
64 bit = 8 byte
```

Bir register içinde yalnızca bitler bulunur. Bu bitlerin ne anlama geldiğini programın kullandığı instruction belirler.

Aynı bit dizisi:

- `signed integer`
- `unsigned integer`
- `memory address`
- `bit mask`
- başka bir binary data

olarak yorumlanabilir.

> Register datanın anlamını bilmez; yalnızca bitleri saklar.

---

## 4. RAX, EAX, AX, AH ve AL

x86 architecture yıllar içinde 8-bit'ten 16-bit'e, 32-bit'e ve ardından 64-bit'e genişlemiştir. Bu tarih nedeniyle aynı register'ın farklı boyutlarına farklı isimlerle erişilebilir.

```text
A → AX → EAX → RAX
```

`RAX` ailesi:

```text
RAX → 64-bit
EAX → RAX'in düşük 32-bit bölümü
AX  → RAX'in düşük 16-bit bölümü
AH  → AX'in yüksek 8-bit bölümü
AL  → AX'in düşük 8-bit bölümü
```

Şema:

```text
RAX = 64 bit
└── EAX = düşük 32 bit
    └── AX = düşük 16 bit
        ├── AH = bit 8-15
        └── AL = bit 0-7
```

Bunlar farklı register'lar değildir. Aynı register'ın farklı parçalarıdır.

---

## 5. Partial Register kullanımı

Bir register'ın tamamı yerine belirli bir bölümüne erişmeye `partial register access` denir.

Başlangıçta:

```text
RAX = 0
```

olsun.

Şu instruction'ları çalıştıralım:

```asm
mov ah, 0x05
mov al, 0x39
```

Sonuç:

```text
AX = 0x0539
```

Decimal karşılığı:

```text
1337
```

Çünkü `AH`, düşük 16-bit alanın yüksek byte'ını; `AL` ise düşük byte'ını değiştirir.

---

## 6. MOV instruction

Register'a data koymak için en sık kullanılan instruction'lardan biri `mov` instruction'ıdır.

Intel syntax:

```asm
mov destination, source
```

Örnek:

```asm
mov rax, 1337
```

Burada:

```text
Instruction = mov
Destination = rax
Source      = 1337
```

Günlük dilde:

```text
1337 değerini RAX içine kopyala.
```

Başka bir örnek:

```asm
mov rbx, rax
```

Anlamı:

```text
RAX içindeki datayı RBX içine kopyala.
```

Data flow sağdan sola gider:

```text
Source → Destination
RAX    → RBX
```

---

## 7. Immediate Value

Instruction içinde doğrudan yazılan sabit değere `immediate value` denir.

```asm
mov rax, 25
```

Buradaki `25`, immediate value'dur.

Başka örnekler:

```asm
add rax, 5
sub rbx, 2
xor rcx, 0xff
```

Buradaki `5`, `2` ve `0xff` değerleri immediate value olarak kullanılır.

Bunu şöyle düşünebilirsin:

```text
“Başka yerden data arama. Gereken değeri doğrudan veriyorum.”
```

---

## 8. MOV neden kopyalama gibi çalışır?

`mov` kelimesi “taşı” anlamına gelse de source data silinmez.

```asm
mov rax, 10
mov rbx, rax
```

Sonuç:

```text
RAX = 10
RBX = 10
```

`RAX` boşalmaz. Data yalnızca `RBX` içine kopyalanır.

Bu nedenle davranış olarak şuna daha yakındır:

```text
Copy source into destination.
```

---

## 9. Operand Size

Register'lar arasında data taşırken `operand size` uyumlu olmalıdır.

Geçerli örnekler:

```asm
mov rax, rbx   ; 64-bit → 64-bit
mov eax, ebx   ; 32-bit → 32-bit
mov ax, bx     ; 16-bit → 16-bit
mov al, bl     ; 8-bit  → 8-bit
```

Boyutları doğrudan karıştırmak çoğunlukla geçerli değildir:

```asm
mov rax, ebx
```

Çünkü:

```text
RAX = 64-bit
EBX = 32-bit
```

Boyut farkı olduğunda datanın nasıl genişletileceği belirtilmelidir. Bu noktada şu instruction'lar kullanılabilir:

- `movzx`
- `movsx`
- `movsxd`

---

## 10. EAX'e yazınca RAX neden değişir?

x86-64 üzerinde 32-bit bir general-purpose register'a yazıldığında, karşılık gelen 64-bit register'ın üst 32 biti sıfırlanır.

```asm
mov rax, 0xffffffffffffffff
mov eax, 1
```

İlk instruction sonrasında:

```text
RAX = 0xffffffffffffffff
```

İkinci instruction sonrasında:

```text
RAX = 0x0000000000000001
```

Bu yalnızca `EAX` için geçerli değildir:

```text
EBX'e yazmak → RBX'in üst 32 bitini sıfırlar.
ECX'e yazmak → RCX'in üst 32 bitini sıfırlar.
EDX'e yazmak → RDX'in üst 32 bitini sıfırlar.
ESI'ye yazmak → RSI'ın üst 32 bitini sıfırlar.
EDI'ye yazmak → RDI'ın üst 32 bitini sıfırlar.
```

Bu davranışı unutmak assembly analizinde çok sık hataya neden olur.

---

## 11. Zero-Extension

Küçük bir değeri daha büyük alana taşırken yeni açılan üst bitlerin sıfırlarla doldurulmasına `zero-extension` denir.

32-bit değer:

```text
0xffffffff
```

64-bit zero-extended hali:

```text
0x00000000ffffffff
```

Binary görünüm:

```text
00000000 00000000 00000000 00000000
11111111 11111111 11111111 11111111
```

Unsigned yorumda bu değer:

```text
4294967295
```

anlamına gelir.

---

## 12. Signed ve Unsigned yorum

Register içinde yalnızca bitler vardır. Aynı bit dizisi farklı biçimlerde yorumlanabilir.

```text
0xffffffff
```

32-bit olarak:

```text
Unsigned → 4294967295
Signed   → -1
```

CPU'nun tuttuğu bitler değişmez. Fark, programın bu datayı hangi anlamda kullandığıdır.

Assembly incelerken şu soruyu sormalısın:

> Bu değer signed mı, unsigned mı değerlendiriliyor?

---

## 13. Two's Complement

Negative integer değerler çoğunlukla `two's complement` biçiminde gösterilir.

32-bit `-1`:

```text
0xffffffff
```

64-bit `-1`:

```text
0xffffffffffffffff
```

32-bit `-1` değerinin başına sıfırlar eklersek:

```text
0x00000000ffffffff
```

elde edilir. Ancak bu 64-bit signed `-1` değildir.

Signed anlamı korumak için `sign extension` gerekir.

---

## 14. Sign Extension

Küçük signed bir değerin daha büyük bit genişliğine aktarılırken sign bilgisinin korunmasına `sign extension` denir.

Positive örnek:

```text
32-bit: 0x00000005
64-bit: 0x0000000000000005
```

Negative örnek:

```text
32-bit: 0xffffffff
64-bit: 0xffffffffffffffff
```

Negative value genişletilirken üst taraf `1` bitleriyle doldurulur. Böylece sayının signed anlamı korunur.

---

## 15. MOVSX ve MOVSXD

`movsx`, “move with sign extension” mantığıyla çalışır.

Genel yapı:

```asm
movsx destination, source
```

32-bit signed değeri 64-bit'e genişletmek için x86-64 üzerinde `movsxd` instruction'ı görülebilir:

```asm
mov eax, -1
movsxd rax, eax
```

İlk instruction sonrasında x86-64 davranışı nedeniyle:

```text
RAX = 0x00000000ffffffff
```

olur.

Ardından:

```asm
movsxd rax, eax
```

çalışınca:

```text
RAX = 0xffffffffffffffff
```

elde edilir.

Signed yorum:

```text
RAX = -1
```

---

## 16. Arithmetic Operations

Register içindeki data üzerinde arithmetic operation yapılabilir.

### ADD

```asm
add rax, rbx
```

```text
RAX = RAX + RBX
```

### SUB

```asm
sub rax, 5
```

```text
RAX = RAX - 5
```

### INC

```asm
inc rax
```

```text
RAX = RAX + 1
```

### DEC

```asm
dec rax
```

```text
RAX = RAX - 1
```

### IMUL

```asm
imul rax, rbx
```

Signed multiplication için kullanılabilir.

### DIV ve IDIV

```text
div  → unsigned division
idiv → signed division
```

Division instruction'larının özel register kuralları vardır; bu konu ayrı bir bölümde ele alınmalıdır.

---

## 17. Bitwise ve Shift Operations

### AND

```asm
and rax, rbx
```

Bit çiftleri üzerinde AND operation yapar.

### OR

```asm
or rax, rbx
```

Bit çiftleri üzerinde OR operation yapar.

### XOR

```asm
xor rax, rbx
```

Bit çiftleri üzerinde XOR operation yapar.

Bir register'ı sıfırlamak için sık görülen kullanım:

```asm
xor eax, eax
```

Sonuç:

```text
EAX = 0
RAX = 0
```

Bir değer kendisiyle XOR yapılınca sonuç sıfır olur. Ayrıca EAX'e yazmak RAX'in üst 32 bitini de sıfırlar.

### SHL

```asm
shl rax, 1
```

Bitleri sola kaydırır. Basit unsigned değerlerde bir bit sola kaydırmak çoğunlukla ikiyle çarpmaya benzer.

### SHR

```asm
shr rax, 1
```

Bitleri sağa kaydırır ve üst tarafı sıfırlarla doldurur.

### SAR

```asm
sar rax, 1
```

Signed arithmetic right shift yapar ve sign bilgisini korur.

---

## 18. XCHG

İki register'ın değerini değiştirmek için `xchg` kullanılabilir.

```asm
mov rax, 10
mov rbx, 20
xchg rax, rbx
```

Önce:

```text
RAX = 10
RBX = 20
```

Sonra:

```text
RAX = 20
RBX = 10
```

`xchg`, iki kutunun içindeki datayı birbiriyle değiştirmek gibidir.

---

## 19. Special Registers

Bütün register'lar yalnızca normal hesaplamalar için kullanılmaz.

Bazı register'ların özel görevleri vardır:

- `RIP`
- `RSP`
- `RBP`
- `Flags register`
- `Control registers`
- `SIMD registers`

Başlangıçta bütün special register'ları ezberlemene gerek yoktur. Fakat `RIP` ve `RSP`, reverse engineering ve binary exploitation açısından çok önemlidir.

---

## 20. RIP

`RIP`, x86-64 üzerindeki `instruction pointer` register'ıdır.

> CPU'nun sıradaki instruction'ı hangi code address'ten alacağını takip eder.

```text
RIP → Next instruction address
```

Debugger içinde:

```text
RIP = 0x401156
```

görülüyorsa execution flow ilgili code address çevresindedir.

`RIP` genellikle şu control-flow mekanizmalarıyla değişir:

- `call`
- `ret`
- `jmp`
- conditional jump
- exception
- interrupt

Binary exploitation sırasında `RIP control`, programın hangi instruction'a gideceğini etkileyebilmek anlamına gelir.

---

## 21. RSP

`RSP`, `stack pointer` register'ıdır.

```text
RSP → Current top of stack
```

Stack üzerinde şu tür datalar bulunabilir:

- local variables
- saved registers
- return addresses
- function arguments
- temporary data

Örnek instruction'lar:

```asm
push rax
pop rbx
```

`push`, datayı stack'e koyar. `pop`, stack'ten data alır. Bu işlemler sırasında `RSP` otomatik olarak değişir.

---

## 22. GDB ile register inceleme

Bütün register'ları görüntülemek için:

```gdb
info registers
```

Belirli bir register:

```gdb
info registers rax
```

Hexadecimal görünüm:

```gdb
p/x $rax
```

Signed decimal görünüm:

```gdb
p/d $rax
```

Unsigned decimal görünüm:

```gdb
p/u $rax
```

Binary görünüm:

```gdb
p/t $rax
```

Register'a manuel değer yazmak:

```gdb
set $rax = 1337
```

Tek instruction ilerlemek:

```gdb
si
```

Sıradaki instruction'ı görmek:

```gdb
x/i $rip
```

Yakındaki instruction'ları görmek:

```gdb
x/10i $rip
```

Önerilen çalışma yöntemi:

```text
1. Instruction'ı oku.
2. Register'ların nasıl değişeceğini tahmin et.
3. `si` ile bir instruction ilerle.
4. `info registers` ile sonucu kontrol et.
5. Tahmininle gerçek sonucu karşılaştır.
```

---

## 23. Yaygın hatalar

### MOV source değerini siler sanmak

```asm
mov rbx, rax
```

işleminden sonra RAX boşalmaz.

```text
RAX aynı kalır.
RBX içine bir kopya yazılır.
```

### Data flow yönünü ters okumak

Intel syntax:

```asm
mov destination, source
```

```asm
mov rbx, rax
```

Data flow:

```text
RAX → RBX
```

### EAX ve RAX'i bağımsız sanmak

`EAX`, `RAX` register'ının düşük 32-bit bölümüdür.

### EAX'e yazmanın yalnızca alt kısmı değiştirdiğini sanmak

x86-64 üzerinde EAX'e yazıldığında RAX'in üst 32 biti sıfırlanır.

### Signed ve Unsigned değerleri karıştırmak

```text
0xffffffff
```

hem `4294967295` hem de `-1` olarak yorumlanabilir.

### Operand Size kontrol etmemek

8-bit, 16-bit, 32-bit ve 64-bit operand'lar rastgele birbirine kopyalanamaz.

---

## 24. Mini alıştırmalar

### Soru 1

`Register` neden system memory'den daha hızlıdır?

### Soru 2

Aşağıdaki instruction'da `destination`, `source` ve `immediate value` hangisidir?

```asm
mov rax, 25
```

### Soru 3

Kod çalıştıktan sonra değerler nedir?

```asm
mov rax, 10
mov rbx, rax
```

### Soru 4

İsimleri boyutlarla eşleştir:

```text
RAX
EAX
AX
AH
AL
```

```text
64-bit
32-bit
16-bit
8-bit high
8-bit low
```

### Soru 5

Başlangıçta `RAX = 0` ise sonuç nedir?

```asm
mov ah, 0x05
mov al, 0x39
```

### Soru 6

Son `RAX` değeri nedir?

```asm
mov rax, 0xffffffffffffffff
mov eax, 1
```

### Soru 7

`0xffffffff` değeri signed ve unsigned olarak nasıl yorumlanır?

### Soru 8

`zero-extension` ve `sign extension` arasındaki fark nedir?

### Soru 9

Aşağıdaki instruction ne yapar?

```asm
xchg rax, rbx
```

### Soru 10

Sonuç nedir?

```asm
mov rax, 7
inc rax
inc rax
dec rax
```

### Soru 11

Aşağıdaki instruction neden RAX'i sıfırlar?

```asm
xor eax, eax
```

### Soru 12

`RIP` ve `RSP` görevlerini birer cümleyle açıklayın.

---

## 25. Cevap anahtarı

<details>
<summary>Cevapları görmek için tıkla</summary>

### Cevap 1

Register doğrudan CPU'nun içinde bulunur. CPU register içindeki dataya system memory'den çok daha hızlı ulaşır.

### Cevap 2

```text
Instruction     = mov
Destination     = rax
Source          = 25
Immediate Value = 25
```

### Cevap 3

```text
RAX = 10
RBX = 10
```

### Cevap 4

```text
RAX → 64-bit
EAX → 32-bit
AX  → 16-bit
AH  → 8-bit high
AL  → 8-bit low
```

### Cevap 5

```text
AX = 0x0539
```

Decimal:

```text
1337
```

### Cevap 6

```text
RAX = 0x0000000000000001
```

### Cevap 7

```text
Unsigned → 4294967295
Signed   → -1
```

### Cevap 8

`Zero-extension`, üst bitleri sıfırla doldurur. `Sign extension`, signed değerin işaretini korumak için üst bitleri sign bit ile doldurur.

### Cevap 9

RAX ve RBX içindeki değerleri birbiriyle değiştirir.

### Cevap 10

```text
RAX = 8
```

### Cevap 11

Bir değer kendisiyle XOR yapılınca sıfır olur. EAX'e yazıldığı için RAX'in üst 32 biti de sıfırlanır.

### Cevap 12

```text
RIP → Sıradaki instruction'ın code address bilgisini takip eder.
RSP → Stack'in aktif üst kısmını gösterir.
```

</details>

---

## 26. English Term Glossary

| English Term | Türkçe açıklama |
|---|---|
| Register | CPU içindeki küçük ve hızlı data storage alanı |
| Register File | CPU içindeki register grubunun tamamı |
| Data | CPU tarafından işlenen binary bilgi |
| Cache | Memory ile register arasındaki hızlı ara storage katmanı |
| System Memory | Programın aktif datalarının bulunduğu RAM alanı |
| Architecture | CPU'nun instruction ve çalışma yapısını tanımlayan sistem |
| Word Size | Architecture'ın doğal olarak işlediği bit genişliği |
| Partial Register | Büyük register'ın erişilebilen küçük bölümü |
| Immediate Value | Instruction içinde doğrudan verilen sabit değer |
| Instruction | CPU'ya belirli bir operation yaptıran komut |
| Operand | Instruction'ın üzerinde çalıştığı değer veya konum |
| Destination | Sonucun yazılacağı operand |
| Source | Datanın alındığı operand |
| Zero-Extension | Üst bitleri sıfırla doldurarak genişletme |
| Sign Extension | Sign bilgisini koruyarak genişletme |
| Signed Integer | Positive ve negative değerleri temsil edebilen integer |
| Unsigned Integer | Sıfır ve positive değerleri temsil eden integer |
| Two's Complement | Negative integer değerlerin binary gösterim yöntemi |
| Arithmetic Operation | Toplama, çıkarma, çarpma ve bölme işlemleri |
| Bitwise Operation | Bitler üzerinde ayrı ayrı yapılan işlem |
| Shift Operation | Bitleri sağa veya sola kaydırma işlemi |
| Special Register | Belirli architecture görevleri olan register |
| Instruction Pointer | Sıradaki instruction address'ini takip eden register |
| Stack Pointer | Stack'in aktif üst kısmını gösteren register |
| Execution Flow | Instruction'ların çalışma sırası |
| Memory Address | Memory içindeki konumu gösteren sayı |
| Debugger | Programı adım adım incelemeyi sağlayan araç |
| Hexadecimal | Base-16 sayı gösterimi |
| Decimal | Base-10 sayı gösterimi |
| Binary | Base-2 sayı gösterimi |

---

## Sonraki bölüm

Bu bölümden sonra `memory addressing` ve `stack` konularına geçmek mantıklıdır.

Önerilen sıra:

```text
1. Memory Address
2. Dereference
3. RAX ile [RAX] farkı
4. Stack
5. PUSH ve POP
6. RSP ve RBP
7. Function Call
8. CALL ve RET
9. Return Address
10. Stack Frame
```

> Register isimlerini yalnızca ezberlemeye çalışma. Her instruction için datanın nereden geldiğini, nereye gittiğini ve kaç bit olduğunu takip et.
