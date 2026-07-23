# x86-64 Memory, Stack ve Addressing Temelleri

> **Amaç:** `memory`, `virtual memory`, `stack`, `pointer`, `push`, `pop`, `memory addressing`, `little endian`, `LEA` ve `RIP-relative addressing` konularını günlük bir dille öğrenmek.

---

## İçindekiler

- [1. Neden Memory'ye ihtiyaç duyarız?](#1-neden-memoryye-ihtiyaç-duyarız)
- [2. Register ile Memory arasındaki fark](#2-register-ile-memory-arasındaki-fark)
- [3. Memory Address nedir?](#3-memory-address-nedir)
- [4. Virtual Memory ve Process görünümü](#4-virtual-memory-ve-process-görünümü)
- [5. Process Memory Layout](#5-process-memory-layout)
- [6. Stack nedir?](#6-stack-nedir)
- [7. PUSH ve POP](#7-push-ve-pop)
- [8. RSP ve Stack'in büyüme yönü](#8-rsp-ve-stackin-büyüme-yönü)
- [9. RAX ile [RAX] arasındaki fark](#9-rax-ile-rax-arasındaki-fark)
- [10. Pointer ve Dereference](#10-pointer-ve-dereference)
- [11. Memory'den Load ve Memory'ye Store](#11-memoryden-load-ve-memoryye-store)
- [12. Memory Access Size](#12-memory-access-size)
- [13. BYTE, WORD, DWORD ve QWORD](#13-byte-word-dword-ve-qword)
- [14. Little Endian](#14-little-endian)
- [15. Effective Address hesaplama](#15-effective-address-hesaplama)
- [16. LEA instruction](#16-lea-instruction)
- [17. RIP-relative addressing](#17-rip-relative-addressing)
- [18. Yaygın hatalar](#18-yaygın-hatalar)
- [19. GDB ile uygulama](#19-gdb-ile-uygulama)
- [20. Bölüm özeti](#20-bölüm-özeti)
- [21. Mini alıştırmalar](#21-mini-alıştırmalar)
- [22. Cevap anahtarı](#22-cevap-anahtarı)
- [23. English Term Glossary](#23-english-term-glossary)

---

## 1. Neden Memory'ye ihtiyaç duyarız?

CPU'nun üzerinde çalıştığı en hızlı alanlar `register`lardır. Fakat register sayısı ve kapasiteleri çok sınırlıdır.

Bir program yalnızca birkaç sayı tutmaz. Çalışırken şunlara ihtiyaç duyabilir:

- Program code
- Variables
- Strings
- Images
- Files
- Network packets
- Function data
- Dynamic allocations
- Shared libraries

Bütün bunları birkaç register içine sığdıramayız. Bu yüzden büyük miktardaki data `memory` içinde tutulur.

Günlük hayat benzetmesi:

```text
Register → Masanın üstündeki birkaç belge
Memory   → Büyük dosya dolabı
Disk     → Arşiv odası
```

Masanın üstündeki belgeye hemen ulaşırsın. Dosya dolabındaki belgeyi almak ise daha uzun sürer, fakat orada çok daha fazla şey saklayabilirsin.

### Genel data flow

```text
System Memory
      ↓
Cache
      ↓
Register File
      ↓
CPU Operation
```

CPU mümkün olduğunca register üzerinde çalışır. Gereken data memory'den register'a alınır, işlem yapılır ve sonuç gerekirse memory'ye geri yazılır.

---

## 2. Register ile Memory arasındaki fark

| Özellik | Register | Memory |
|---|---|---|
| Konum | CPU içinde | System memory içinde |
| Hız | Çok hızlı | Register'a göre daha yavaş |
| Kapasite | Çok küçük | Çok daha büyük |
| Erişim | Register adıyla | Memory address ile |
| Örnek | `RAX`, `RBX`, `RSP` | `0x401000`, `0x7fffffffe050` |

Register'a ismiyle erişiriz:

```asm
mov rax, 10
```

Memory'ye ise çoğunlukla bir address üzerinden erişiriz:

```asm
mov rax, [rbx]
```

Bu instruction içinde `RBX`, gidilecek memory location'ın address'ini taşır.

---

## 3. Memory Address nedir?

Memory'de çok fazla konum bulunduğu için her konuma ayrı isim vermek mümkün değildir. Bunun yerine memory location'lar sayılarla tanımlanır.

Bu sayılara `memory address` denir.

Örnek:

```text
0x1000
0x401000
0x7fffffffe050
```

Bunu öğrenci numarasına benzetebiliriz. Üniversitede aynı isimde birden fazla kişi olabilir; fakat her öğrencinin kendine ait bir ID'si vardır.

Memory'de de her konumun bir address'i bulunur.

### Her address neyi gösterir?

x86-64 sistemlerde memory `byte-addressable` olarak düşünülür.

Yani her address bir byte'lık konumu temsil eder:

```text
1 byte = 8 bit
```

Örnek:

```text
0x1000 → 1 byte
0x1001 → 1 byte
0x1002 → 1 byte
0x1003 → 1 byte
```

8-byte bir value yazarsak art arda gelen sekiz address kullanılır:

```text
0x1000 - 0x1007
```

---

## 4. Virtual Memory ve Process görünümü

Bir programın gördüğü bütün address'ler doğrudan fiziksel RAM üzerindeki aynı numaralı konumlar değildir.

Her `process`, kendine ait bir `virtual address space` görür.

```text
Process Virtual Address
          ↓
Operating System Mapping
          ↓
Physical Memory
```

Örneğin iki process aynı virtual address'i kullanabilir:

```text
Process A → 0x400000
Process B → 0x400000
```

Fakat operating system bu iki address'i farklı physical memory bölgelerine eşleyebilir.

Bu bölüm için bilmemiz gerekenler:

- Her process kendi memory görünümüne sahiptir.
- Her sayısal address geçerli değildir.
- Operating system hangi memory region'ların kullanılabileceğini belirler.
- Process gerektiğinde yeni memory region isteyebilir.
- Mapped olmayan address'e erişmek programın crash olmasına neden olabilir.

Kaynak derste x86-64 process'lerin çok büyük bir virtual address alanına sahip olduğundan söz edilmektedir. Buradaki önemli nokta, görülebilen address alanının makinedeki fiziksel RAM miktarıyla birebir aynı olmamasıdır.

---

## 5. Process Memory Layout

Bir process'in virtual memory alanında farklı görevleri olan bölgeler bulunur.

Basitleştirilmiş görünüm:

```text
Yüksek address
+----------------------+
| Stack                |
+----------------------+
| Shared Libraries     |
+----------------------+
| Memory Mappings      |
+----------------------+
| Heap                 |
+----------------------+
| Program Data         |
+----------------------+
| Program Code         |
+----------------------+
Düşük address
```

Bu bölgeler arasında şunlar yer alabilir:

- Program code
- Program data
- Heap
- Stack
- Shared libraries
- Dynamically mapped memory
- Operating system helper regions

Bu bölümde esas olarak `stack` ve register-memory data transfer konularına odaklanacağız.

---

## 6. Stack nedir?

`Stack`, process'in kullandığı özel memory region'lardan biridir.

Çoğunlukla geçici data için kullanılır:

- Saved register values
- Function arguments
- Local variables
- Return addresses
- Temporary values

Stack'i üst üste konmuş tabaklar gibi düşünebiliriz.

Yeni tabağı en üste koyarsın. Bir tabak alacağında yine en üsttekini alırsın.

Bu çalışma şekline `LIFO` denir:

```text
Last In, First Out
Son giren, ilk çıkar
```

Örnek:

```text
A push edildi.
B push edildi.
C push edildi.
```

Stack:

```text
Top → C
      B
      A
```

Pop sırası:

```text
C → B → A
```

---

## 7. PUSH ve POP

### PUSH

Bir value'yu stack'e koymak için `push` kullanılır:

```asm
push rax
```

Basitleştirilmiş olarak bu işlem şuna benzer:

```asm
sub rsp, 8
mov [rsp], rax
```

Yani:

1. `RSP` 8 azaltılır.
2. `RAX` içindeki 8-byte value yeni stack top'a yazılır.

Örnek:

```asm
mov rax, 1337
push rax
```

Son durumda:

```text
RAX = 1337
Stack Top = 1337
```

`push`, source register'ı temizlemez. Data hem register'da hem stack'te kalır.

### POP

Stack'in en üstündeki value'yu almak için `pop` kullanılır:

```asm
pop rbx
```

Basitleştirilmiş davranış:

```asm
mov rbx, [rsp]
add rsp, 8
```

Yani:

1. Stack top'taki value okunur.
2. Destination register'a yazılır.
3. `RSP` 8 artırılır.

Örnek:

```asm
mov rax, 1337
push rax
pop rbx
```

Sonuç:

```text
RAX = 1337
RBX = 1337
```

### POP edilen byte'lar gerçekten silinir mi?

Mantıksal olarak stack'ten çıkarılmış kabul edilir. Fakat memory içindeki eski byte'lar anında sıfırlanmayabilir.

`pop` çoğunlukla yalnızca:

- Data'yı okur,
- Destination'a yazar,
- `RSP` değerini değiştirir.

Eski byte'lar memory'de kalabilir; fakat artık aktif stack'in dışında kabul edilir.

---

## 8. RSP ve Stack'in büyüme yönü

`RSP`, `stack pointer` register'ıdır.

Aktif stack top'ın address'ini tutar:

```text
RSP → Stack Top Address
```

Örnek başlangıç:

```text
RSP = 0x7fffffffe050
```

Bir 8-byte value push edildiğinde:

```text
RSP = 0x7fffffffe048
```

Address küçülmüştür.

Şematik görünüm:

```text
Yüksek address
0x...e058
0x...e050  ← Eski RSP
0x...e048  ← Yeni RSP
0x...e040
Düşük address
```

x86-64 üzerinde stack genellikle düşük address yönüne büyür.

### Kısa kural

```text
PUSH → RSP 8 azalır
POP  → RSP 8 artar
```

### Tam örnek

```asm
mov rax, 0x1111
push rax

mov rax, 0x2222
push rax

mov rax, 0x3333
push rax

pop rbx
pop rcx
pop rdx
```

Sonuç:

```text
RBX = 0x3333
RCX = 0x2222
RDX = 0x1111
```

---

## 9. RAX ile [RAX] arasındaki fark

Bu, assembly öğrenirken en önemli ayrımlardan biridir.

### RAX

```asm
mov rbx, rax
```

Anlamı:

```text
RAX register'ının kendi value'sunu RBX'e kopyala.
```

### [RAX]

```asm
mov rbx, [rax]
```

Anlamı:

```text
RAX içindeki sayıyı memory address olarak kullan.
O address'teki data'yı RBX'e yükle.
```

Örnek durum:

```text
RAX = 0x1000
Memory[0x1000] = 0x1337
```

Instruction:

```asm
mov rbx, rax
```

Sonuç:

```text
RBX = 0x1000
```

Instruction:

```asm
mov rbx, [rax]
```

Sonuç:

```text
RBX = 0x1337
```

Köşeli parantezler `dereference` anlamına gelir.

---

## 10. Pointer ve Dereference

Bir register veya variable içinde memory address tutuluyorsa bu value `pointer` olarak kullanılabilir.

Örnek:

```text
RAX = 0x401000
```

Eğer bu value bir memory location'a referans veriyorsa:

```text
RAX points to 0x401000
```

diyebiliriz.

`Pointer`, başka bir memory location'ın address'ini tutan value'dur.

`Dereference` ise pointer'ın gösterdiği konumdaki data'ya erişmektir:

```asm
mov rbx, [rax]
```

CPU açısından address de normal bir bit dizisidir. Onu pointer yapan şey, programın bu value'yu memory access sırasında kullanmasıdır.

### Invalid Memory Access

Her sayı geçerli bir pointer değildir.

```asm
mov rax, 0x12345
mov rbx, [rax]
```

Eğer `0x12345` process için mapped değilse program hata verebilir:

```text
Segmentation fault
```

Bir address'i dereference etmeden önce bölgenin:

- Mapped,
- Readable veya writable,
- Gerekli size için geçerli

olması gerekir.

---

## 11. Memory'den Load ve Memory'ye Store

### Memory'den Register'a Load

```asm
mov rax, 0x1000
mov rbx, [rax]
```

Adım adım:

```text
1. RAX = 0x1000
2. CPU 0x1000 address'ine gider.
3. Oradaki data'yı okur.
4. Data RBX içine yazılır.
```

`RBX` 64-bit olduğu için 8 byte okunur.

### Register'dan Memory'ye Store

```asm
mov rax, 0x1000
mov rbx, 0x1337
mov [rax], rbx
```

Adım adım:

```text
1. RAX target address'i tutar.
2. RBX yazılacak value'yu tutar.
3. RBX içindeki 8-byte value RAX'in gösterdiği memory'ye yazılır.
```

Basit ifade:

```text
Memory[RAX] = RBX
```

---

## 12. Memory Access Size

Memory'den kaç byte okunacağı veya memory'ye kaç byte yazılacağı operand size'a bağlıdır.

| Instruction | Access Size |
|---|---:|
| `mov rbx, [rax]` | 8 byte |
| `mov ebx, [rax]` | 4 byte |
| `mov bx, [rax]` | 2 byte |
| `mov bl, [rax]` | 1 byte |

Aynı mantık store işlemi için de geçerlidir:

```asm
mov [rax], rbx   ; 8 byte
mov [rax], ebx   ; 4 byte
mov [rax], bx    ; 2 byte
mov [rax], bl    ; 1 byte
```

### EAX uyarısı

Memory'den `EAX`, `EBX`, `ECX` gibi 32-bit register'lara load yapıldığında, karşılık gelen 64-bit register'ın üst 32 biti sıfırlanır.

Örnek:

```asm
mov eax, [rbx]
```

Bu instruction:

- Memory'den 4 byte okur.
- EAX'e yazar.
- RAX'in üst 32 bitini sıfırlar.

---

## 13. BYTE, WORD, DWORD ve QWORD

x86 assembly'de sık kullanılan size isimleri:

| Terim | Boyut |
|---|---:|
| `BYTE` | 1 byte / 8 bit |
| `WORD` | 2 byte / 16 bit |
| `DWORD` | 4 byte / 32 bit |
| `QWORD` | 8 byte / 64 bit |

Örnek Intel syntax:

```asm
mov byte ptr [rax], 0x41
mov word ptr [rax], 0x1337
mov dword ptr [rax], 0x12345678
mov qword ptr [rax], rbx
```

### Immediate Value Memory'ye yazılırken neden size gerekir?

Şu instruction belirsiz olabilir:

```asm
mov [rax], 0x1337
```

Assembler şu sorunun cevabını bilmelidir:

```text
0x1337 kaç byte olarak yazılacak?
```

Bu nedenle size belirtilir:

```asm
mov word ptr [rax], 0x1337
```

Bu 2 byte yazar.

```asm
mov dword ptr [rax], 0x1337
```

Bu 4 byte yazar.

Ana fikir:

> Memory operand'ın size'ı başka bir operanddan anlaşılamıyorsa açıkça belirtilmelidir.

---

## 14. Little Endian

x86 architecture multi-byte value'ları memory'de `little endian` byte order ile saklar.

Little endian'da en düşük anlamlı byte, en düşük memory address'e yazılır.

Örnek 32-bit value:

```text
0x12345678
```

Byte'lara ayıralım:

```text
12 34 56 78
```

Memory'deki görünüm:

```text
0x1000 → 78
0x1001 → 56
0x1002 → 34
0x1003 → 12
```

Debugger'da byte byte baktığımızda value ters yazılmış gibi görünür.

### 64-bit örnek

Value:

```text
0x1122334455667788
```

Memory:

```text
0x1000 → 88
0x1001 → 77
0x1002 → 66
0x1003 → 55
0x1004 → 44
0x1005 → 33
0x1006 → 22
0x1007 → 11
```

### Bitler de ters çevrilir mi?

Hayır.

Little endian yalnızca byte sırasını etkiler. Byte içindeki bitler aynı kalır.

Örnek byte:

```text
0x75 = 01110101
```

Bu bit dizisi kendi içinde ters çevrilmez.

### Stack ile neden karıştırılır?

İki ayrı konu aynı anda vardır:

1. Stack düşük address yönüne büyür.
2. Multi-byte data little endian biçimde yazılır.

Stack growth direction ve byte order birbirinden farklı kavramlardır.

---

## 15. Effective Address hesaplama

x86 memory operand'ları yalnızca tek register içermek zorunda değildir.

Genel address formu:

```text
Base + Index × Scale + Displacement
```

Assembly görünümü:

```asm
[base + index*scale + displacement]
```

Örnek:

```asm
[rbx + rcx*8 + 16]
```

Hesaplanan address:

```text
RBX + RCX × 8 + 16
```

### Bileşenler

- `Base`: Ana address'i tutan register
- `Index`: Eleman numarası gibi kullanılan register
- `Scale`: Index için 1, 2, 4 veya 8 katsayısı
- `Displacement`: Sabit offset

### Array örneği

Her elemanı 8 byte olan bir array düşünelim.

```text
RBX = Array base address
RCX = Element index
```

Erişim:

```asm
mov rax, [rbx + rcx*8]
```

Örnek değerler:

```text
RBX = 0x1000
RCX = 3
```

Effective address:

```text
0x1000 + 3 × 8 = 0x1018
```

CPU `0x1018` address'inden 8 byte okur.

---

## 16. LEA instruction

`lea`, `Load Effective Address` anlamına gelir.

Örnek:

```asm
lea rax, [rbx + rcx*8 + 16]
```

Bu instruction memory'deki data'yı okumaz.

Yalnızca address hesabını yapar:

```text
RAX = RBX + RCX × 8 + 16
```

### MOV ile LEA farkı

```asm
mov rax, [rbx + rcx*8]
```

Anlamı:

```text
Address'i hesapla.
O address'teki memory data'yı oku.
RAX'e koy.
```

```asm
lea rax, [rbx + rcx*8]
```

Anlamı:

```text
Address'i hesapla.
Hesaplanan address'i RAX'e koy.
Memory'yi okuma.
```

Örnek:

```text
RBX = 0x1000
RCX = 2
```

`lea` sonucu:

```text
RAX = 0x1010
```

`mov` sonucu:

```text
RAX = Memory[0x1010]
```

---

## 17. RIP-relative addressing

`RIP`, sıradaki instruction'ın address'ini takip eden `instruction pointer` register'ıdır.

x86-64 architecture `RIP-relative addressing` destekler.

Örnek:

```asm
lea rdx, [rip + 0x200]
```

Bu instruction RIP'e göre bir address hesaplar:

```text
Address = RIP + 0x200
```

Memory data okumak için:

```asm
mov rax, [rip + 0x200]
```

Bu yöntem code ile data arasındaki göreli mesafeyi kullanır.

### Neden kullanışlıdır?

Program her çalıştırıldığında aynı absolute address'e yüklenmeyebilir.

Fakat code ve data birlikte taşınırsa aralarındaki offset aynı kalabilir.

```text
Code address değişti.
Data address değişti.
Aralarındaki relative offset değişmedi.
```

Bu sayede binary'nin farklı memory address'lerine yüklenmesine rağmen çalışması kolaylaşır.

Bu yaklaşım `position independent code` ve modern güvenlik mekanizmaları açısından önemlidir.

---

## 18. Yaygın hatalar

### Hata 1: RAX ile [RAX]'i aynı sanmak

```asm
mov rbx, rax
```

RAX'in value'sunu kopyalar.

```asm
mov rbx, [rax]
```

RAX'in gösterdiği memory'den data okur.

### Hata 2: PUSH source register'ı temizler sanmak

`push rax`, RAX'i silmez. Value stack'e kopyalanır.

### Hata 3: POP eski byte'ları sıfırlar sanmak

`pop`, çoğunlukla yalnızca data'yı okur ve RSP'yi değiştirir.

### Hata 4: Stack yüksek address yönüne büyür sanmak

```text
PUSH → RSP azalır
POP  → RSP artar
```

### Hata 5: Little endian bitleri ters çevirir sanmak

Yalnızca byte order değişir.

### Hata 6: Memory access size'ı unutmak

```asm
mov rbx, [rax]  ; 8 byte
mov ebx, [rax]  ; 4 byte
mov bx, [rax]   ; 2 byte
mov bl, [rax]   ; 1 byte
```

### Hata 7: LEA memory okur sanmak

`LEA`, effective address hesaplar; dereference yapmaz.

### Hata 8: Her sayıyı geçerli pointer sanmak

Address mapped değilse access programı crash ettirebilir.

---

## 19. GDB ile uygulama

### Bütün register'ları gösterme

```gdb
info registers
```

### RSP değerini gösterme

```gdb
p/x $rsp
```

### Stack top'tan 8 adet 64-bit value gösterme

```gdb
x/8gx $rsp
```

Buradaki parçalar:

```text
x → Examine memory
8 → Sekiz öğe
g → 8-byte giant word
x → Hexadecimal çıktı
```

### Stack'i byte byte gösterme

```gdb
x/32bx $rsp
```

### RAX'in gösterdiği memory'yi inceleme

```gdb
x/gx $rax
```

### Byte byte inceleme

```gdb
x/16bx $rax
```

### String inceleme

```gdb
x/s $rax
```

### Sıradaki instruction'ları görme

```gdb
x/10i $rip
```

### Tek instruction ilerleme

```gdb
si
```

### PUSH gözlem deneyi

```text
1. `p/x $rsp` ile başlangıç address'ini kaydet.
2. `push rax` instruction'ını çalıştır.
3. RSP'nin 8 azaldığını kontrol et.
4. `x/gx $rsp` ile stack'e yazılan value'yu incele.
5. `pop rbx` çalıştır.
6. RSP'nin 8 arttığını kontrol et.
```

---

## 20. Bölüm özeti

Bu bölümde öğrendiğimiz temel noktalar:

- Register sayısı sınırlı olduğu için büyük miktardaki data memory'de tutulur.
- Memory location'lar sayısal address'lerle tanımlanır.
- Her memory address bir byte'lık konumu temsil eder.
- Process, virtual address space üzerinden kendi memory görünümüne sahiptir.
- Stack, LIFO mantığıyla kullanılan özel bir memory region'dır.
- `push`, RSP'yi azaltıp data'yı stack'e yazar.
- `pop`, data'yı okuyup RSP'yi artırır.
- `RSP`, aktif stack top address'ini tutar.
- Stack x86-64 üzerinde genellikle düşük address yönüne büyür.
- `RAX`, register value'sunu; `[RAX]`, RAX'in gösterdiği memory data'yı ifade eder.
- Pointer, başka bir memory location'ın address'ini tutar.
- Invalid address dereference edilirse program crash olabilir.
- Memory access size operand size'a bağlıdır.
- `BYTE`, `WORD`, `DWORD` ve `QWORD` farklı data size'larını gösterir.
- x86 multi-byte value'ları little endian biçimde saklar.
- Little endian byte order'ı değiştirir, bitleri değil.
- Effective address formu `base + index × scale + displacement` biçimindedir.
- `LEA`, memory okumadan address hesabı yapar.
- `RIP-relative addressing`, code ve data'ya relative offset üzerinden erişmeyi sağlar.

Tek cümlelik özet:

> `Memory`, büyük miktardaki datayı address'lerle saklar; `stack` ise bu memory'nin LIFO mantığıyla kullanılan özel bir bölümüdür.

---

## 21. Mini alıştırmalar

### Soru 1

Register sayısı neden sınırlıdır ve büyük data neden memory'ye konur?

### Soru 2

Aşağıdaki instruction'lar arasındaki fark nedir?

```asm
mov rbx, rax
mov rbx, [rax]
```

### Soru 3

Başlangıçta:

```text
RSP = 0x1050
```

ise `push rax` sonrasında RSP kaç olur?

### Soru 4

Başlangıçta:

```text
RSP = 0x1048
```

ise `pop rbx` sonrasında RSP kaç olur?

### Soru 5

Aşağıdaki sıradan sonra ilk pop edilen value hangisidir?

```asm
push rax   ; RAX = 1
push rbx   ; RBX = 2
push rcx   ; RCX = 3
```

### Soru 6

Aşağıdaki instruction kaç byte okur?

```asm
mov ebx, [rax]
```

### Soru 7

Aşağıdaki instruction kaç byte yazar?

```asm
mov byte ptr [rax], 0x41
```

### Soru 8

`0x12345678` value'su little endian memory'de hangi byte sırasıyla görünür?

### Soru 9

Little endian byte içindeki bitleri de ters çevirir mi?

### Soru 10

Aşağıdaki effective address'i hesaplayın:

```text
RBX = 0x1000
RCX = 4

[RBX + RCX*8 + 16]
```

### Soru 11

Aşağıdaki instruction'lar arasındaki fark nedir?

```asm
mov rax, [rbx + rcx*8]
lea rax, [rbx + rcx*8]
```

### Soru 12

Pointer nedir?

### Soru 13

Mapped olmayan memory address dereference edilirse ne olabilir?

### Soru 14

`RIP-relative addressing` neden kullanışlıdır?

---

## 22. Cevap anahtarı

<details>
<summary>Cevapları görmek için tıkla</summary>

### Cevap 1

Register'lar CPU içinde bulunduğu için çok hızlı fakat donanımsal olarak pahalı ve küçüktür. Büyük miktardaki data bu nedenle memory'de tutulur.

### Cevap 2

```asm
mov rbx, rax
```

RAX'in kendi value'sunu RBX'e kopyalar.

```asm
mov rbx, [rax]
```

RAX'i memory address olarak kullanır ve o address'teki data'yı RBX'e yükler.

### Cevap 3

```text
RSP = 0x1048
```

### Cevap 4

```text
RSP = 0x1050
```

### Cevap 5

```text
3
```

Son push edilen value ilk pop edilir.

### Cevap 6

```text
4 byte
```

### Cevap 7

```text
1 byte
```

### Cevap 8

```text
78 56 34 12
```

### Cevap 9

Hayır. Yalnızca byte order değişir.

### Cevap 10

```text
0x1000 + 4 × 8 + 16
0x1000 + 32 + 16
0x1030
```

### Cevap 11

`mov`, hesaplanan address'teki memory data'yı okur.

`lea`, yalnızca address hesabını yapar ve sonucu RAX'e koyar.

### Cevap 12

Başka bir memory location'ın address'ini tutan value'dur.

### Cevap 13

Program segmentation fault gibi bir hata ile crash olabilir.

### Cevap 14

Programın belirli bir absolute load address'ine bağımlılığını azaltır. Code ve data arasındaki relative offset kullanılarak farklı memory konumlarında çalışmaya yardımcı olur.

</details>

---

## 23. English Term Glossary

| English Term | Türkçe açıklama |
|---|---|
| Memory | Program data'sının address'lerle saklandığı alan |
| System Memory | Aktif programların kullandığı RAM |
| Memory Address | Memory içindeki bir byte konumunu tanımlayan sayı |
| Address Space | Bir process'in görebildiği address aralığı |
| Virtual Memory | Process'e fiziksel RAM'den bağımsız address görünümü sağlayan sistem |
| Physical Memory | Gerçek RAM üzerindeki storage alanı |
| Memory Mapping | Virtual address ile memory resource arasındaki eşleme |
| Process | Çalışmakta olan program örneği |
| Process Memory Layout | Process address space içindeki memory region düzeni |
| Stack | LIFO mantığıyla kullanılan memory region |
| Heap | Dynamic allocation için kullanılan memory region |
| Shared Library | Programların kullanabildiği library code bölgesi |
| LIFO | Last In, First Out |
| PUSH | Bir value'yu stack üzerine koyan instruction |
| POP | Stack top'taki value'yu alan instruction |
| Stack Pointer | Aktif stack top address'ini tutan register |
| Pointer | Memory address tutan value |
| Dereference | Pointer'ın gösterdiği memory location'a erişme |
| Load | Memory'den register'a data alma |
| Store | Register'dan memory'ye data yazma |
| Invalid Memory Access | İzin verilmeyen veya mapped olmayan memory erişimi |
| Segmentation Fault | Geçersiz memory erişiminde oluşabilen hata |
| Byte-Addressable | Her address'in bir byte'ı tanımladığı memory yapısı |
| Access Size | Okunan veya yazılan byte miktarı |
| BYTE | 1-byte data size |
| WORD | 2-byte data size |
| DWORD | 4-byte data size |
| QWORD | 8-byte data size |
| Little Endian | En düşük anlamlı byte'ın düşük address'e yazıldığı byte order |
| Byte Order | Multi-byte data'nın memory'deki byte sırası |
| Effective Address | Memory operand için hesaplanan gerçek address |
| Base Register | Address hesabındaki ana register |
| Index Register | Eleman numarası gibi kullanılan register |
| Scale | Index'in çarpıldığı 1, 2, 4 veya 8 katsayısı |
| Displacement | Address hesabına eklenen sabit offset |
| LEA | Load Effective Address instruction'ı |
| RIP-relative Addressing | RIP'e göre offset kullanarak address hesaplama |
| Position Independent Code | Belirli bir load address'ine bağlı olmadan çalışabilen code |
| Mapped Memory | Process address space içinde erişime açılmış memory region |

---

## Sonraki Bölüm

Bu konudan sonra mantıklı çalışma sırası şöyledir:

```text
1. CMP ve TEST
2. Flags register
3. JMP
4. Conditional jumps
5. CALL
6. RET
7. Function arguments
8. Calling convention
9. Stack frame
10. Return address
11. Buffer overflow bağlantısı
```

> Memory incelerken her value için şu üç soruyu sor: Bu normal data mı, bir memory address mi, yoksa bir address'in gösterdiği data mı?
