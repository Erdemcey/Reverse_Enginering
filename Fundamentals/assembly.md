# Assembly'ye Giriş: CPU'nun Dilini Öğrenmek

> **Amaç:** Assembly'nin neden ortaya çıktığını, makine koduyla ilişkisini, bir assembly komutunun nasıl okunduğunu ve CPU'nun veriyi hangi biçimlerde kullandığını gündelik bir dille öğrenmek.

---

## İçindekiler

- [1. Bütün yollar yine CPU'ya çıkar](#1-bütün-yollar-yine-cpuya-çıkar)
- [2. Neden doğrudan makine kodu yazmıyoruz?](#2-neden-doğrudan-makine-kodu-yazmıyoruz)
- [3. Assembly tam olarak nedir?](#3-assembly-tam-olarak-nedir)
- [4. Assembly yüksek seviyeli bir dil değildir](#4-assembly-yüksek-seviyeli-bir-dil-değildir)
- [5. Bir assembly komutunun yapısı](#5-bir-assembly-komutunun-yapısı)
- [6. CPU veriyi nereden alır?](#6-cpu-veriyi-nereden-alır)
- [7. Immediate değerler](#7-immediate-değerler)
- [8. Register içindeki veriler](#8-register-içindeki-veriler)
- [9. Bellekteki veriler](#9-bellekteki-veriler)
- [10. Assembly'nin fiilleri](#10-assemblynin-fiilleri)
- [11. Komutlar kaç operand alabilir?](#11-komutlar-kaç-operand-alabilir)
- [12. İlk assembly örneğimiz](#12-ilk-assembly-örneğimiz)
- [13. Neden farklı assembly dilleri var?](#13-neden-farklı-assembly-dilleri-var)
- [14. x86 ve x86-64](#14-x86-ve-x86-64)
- [15. Intel ve AT&T sözdizimi](#15-intel-ve-att-sözdizimi)
- [16. Assembly öğrenirken nasıl düşünmeliyiz?](#16-assembly-öğrenirken-nasıl-düşünmeliyiz)
- [17. Bölüm özeti](#17-bölüm-özeti)
- [18. Mini alıştırmalar](#18-mini-alıştırmalar)
- [19. Cevap anahtarı](#19-cevap-anahtarı)
- [20. Kavram sözlüğü](#20-kavram-sözlüğü)

---

## 1. Bütün yollar yine CPU'ya çıkar

Python, C, Rust, JavaScript veya başka bir dil kullanıyor olabilirsin. Yazdığın kod ekranda ne kadar farklı görünürse görünsün, en sonunda bilgisayarın işlemcisi tarafından yürütülmesi gerekir.

Genel akış şöyledir:

```text
Yazdığımız program
        ↓
Derleme, JIT veya yorumlama
        ↓
Makine kodu
        ↓
CPU
```

Kodun CPU'ya ulaşması birkaç farklı yolla gerçekleşebilir.

### Önceden derleme — Ahead-of-Time

C gibi dillerde kaynak kod, program çalıştırılmadan önce makine koduna çevrilebilir.

```text
C kodu → Derleyici → Çalıştırılabilir dosya → CPU
```

### Çalışma sırasında derleme — Just-in-Time

Bazı sistemlerde kodun gerekli bölümleri program çalışırken makine koduna dönüştürülür.

```text
Kaynak kod → JIT derleyici → Makine kodu → CPU
```

### Yorumlama

Python gibi dillerde kaynak kodu işleyen bir yorumlayıcı bulunabilir. Ancak burada da CPU boşta değildir. CPU, yorumlayıcının makine kodunu çalıştırır.

```text
Python kodu
     ↓
Python yorumlayıcısı
     ↓
CPU'nun çalıştırdığı makine komutları
```

Yani hangi yolu seçersek seçelim, son durak çoğunlukla aynıdır:

> **İşlemcinin anlayabildiği ikili komutlar.**

---

## 2. Neden doğrudan makine kodu yazmıyoruz?

CPU, komutları bitlerden oluşan makine kodu biçiminde görür.

Örneğin karşında şöyle bir dizi olduğunu düşün:

```text
01010101 01001000 10001001
```

CPU açısından bu bitlerin belirli bir anlamı olabilir. İnsan açısından ise bunlar çoğunlukla rastgele rakamlar gibi görünür.

Bir insanın yüzlerce veya binlerce farklı bit dizisini ezberlemesi oldukça zordur. Üstelik tek bir bitin yanlış yazılması bile komutun tamamen değişmesine neden olabilir.

Bu nedenle insan ile makine kodu arasına daha okunabilir bir gösterim koyduk:

```text
Makine kodu  ↔  Assembly
```

Assembly sayesinde:

- Bit dizilerini ezberlemek zorunda kalmayız.
- Komutun ne yaptığını isminden anlayabiliriz.
- Kod daha kolay okunur.
- Hata bulmak daha kolay hale gelir.
- Makine koduna oldukça yakın kalmaya devam ederiz.

---

## 3. Assembly tam olarak nedir?

Assembly, CPU komutlarının insanlar tarafından okunabilen metinsel gösterimidir.

Örneğin işlemciye iki değeri toplamasını söyleyen makine komutunu, assembly tarafında şöyle görebiliriz:

```asm
add rax, rbx
```

Bu satırı kabaca şöyle okuyabiliriz:

```text
RAX ile RBX üzerinde toplama işlemi yap.
```

Assembly ile makine kodu arasında çok yakın bir ilişki vardır.

```text
Assembly komutu
       ↓ Assembler
Makine kodu
       ↓
CPU
```

Assembly metnini makine koduna çeviren programa **assembler** denir.

### Derleyici ile assembler aynı şey mi?

Tam olarak aynı değildir.

Bir C derleyicisi tek bir yüksek seviyeli ifadeyi birçok makine komutuna çevirebilir:

```c
total = price * quantity + tax;
```

Bu ifade assembly tarafında birden fazla adıma ayrılabilir:

```text
Değeri al
Diğer değeri al
Çarp
Vergiyi ekle
Sonucu sakla
```

Assembler ise assembly komutlarını işlemcinin anlayacağı ikili biçime dönüştürür.

---

## 4. Assembly yüksek seviyeli bir dil değildir

Python'da tek satırla çok büyük bir iş yaptırabiliriz:

```python
print(sorted(numbers))
```

Bu satırın arkasında:

- Listeyi dolaşma,
- Değerleri karşılaştırma,
- Sıralama,
- Metne dönüştürme,
- Ekrana yazdırma

gibi çok sayıda işlem bulunabilir.

Assembly'de ise CPU'ya küçük ve açık adımlar veririz:

```text
Bu değeri al.
Şuraya taşı.
Diğeriyle karşılaştır.
Şuraya atla.
İki değeri topla.
Sonucu kaydet.
```

Bu yüzden assembly için şu benzetme kullanılabilir:

> Yüksek seviyeli dil, restoranda “bir menü alabilir miyim?” demektir.  
> Assembly ise mutfağa girip yapılacak işlemleri tek tek söylemektir.

Assembly ayrıntılıdır; fakat bu ayrıntı sayesinde programın CPU seviyesinde gerçekte ne yaptığını görebiliriz.

---

## 5. Bir assembly komutunun yapısı

Bir assembly komutunu basit bir cümle gibi düşünebiliriz.

Normal bir cümlede:

```text
Erdem kapıyı açtı.
```

- **Açtı:** Fiil
- **Kapı:** İşlemin uygulandığı nesne

Assembly'de de benzer bir yapı vardır:

```asm
mov rax, 10
```

- `mov`: Yapılacak işlem
- `rax`: İşlemin hedefi
- `10`: Kullanılacak değer

Assembly terminolojisinde:

- Yapılacak işlemi belirten bölüme **mnemonic** veya işlem adı denir.
- İşlem üzerinde kullanılan değerlere **operand** denir.

Genel görünüm:

```asm
işlem operand1, operand2
```

Örnekler:

```asm
add rax, rbx
sub rcx, 4
mov rdx, rax
```

Bunları günlük dile çevirelim:

```text
RAX ile RBX'i topla.
RCX'ten 4 çıkar.
RAX'teki değeri RDX'e taşı.
```

---

## 6. CPU veriyi nereden alır?

Assembly öğrenirken en önemli konulardan biri verinin nerede bulunduğunu anlamaktır.

Bu bölümde üç temel veri bağlamı kullanacağız:

1. Komutun içinde doğrudan verilen veri
2. Register içinde bulunan veri
3. Bellekte bulunan veri

Bunu para örneğiyle düşünelim.

| Assembly dünyası | Para benzetmesi |
|---|---|
| Immediate değer | Elindeki para |
| Register | Kasadaki para |
| Bellek | Banka kasasındaki para |

Bu üçü de paradır; yalnızca bulundukları yer farklıdır.

Aynı şekilde CPU açısından da hepsi veridir. Fakat veriye ulaşma yöntemi değişir.

---

## 7. Immediate değerler

Komutun içinde doğrudan yazdığımız sabit değere **immediate value** denir.

Örnek:

```asm
mov rax, 10
```

Buradaki `10`, komutun içinde doğrudan verilmiştir.

Bunu şöyle okuyabiliriz:

```text
RAX register'ına doğrudan 10 değerini koy.
```

Başka örnekler:

```asm
add rax, 5
sub rbx, 2
```

Bu komutlarda:

- `5` doğrudan toplama işlemine verilir.
- `2` doğrudan çıkarma işlemine verilir.

Para benzetmesinde immediate değer, elindeki parayı doğrudan kasiyere vermeye benzer.

```text
“Al, şu 10 lirayı kullan.”
```

---

## 8. Register içindeki veriler

Register'lar CPU'nun içinde bulunan küçük ve çok hızlı depolama alanlarıdır.

x86-64 üzerinde sık göreceğimiz bazı register isimleri şunlardır:

```text
RAX
RBX
RCX
RDX
RSI
RDI
RSP
RBP
RIP
```

Şimdilik hepsinin özel görevini ezberlememiz gerekmiyor. Önce register'ı basitçe şöyle düşünelim:

> CPU'nun çalışırken elinin altında tuttuğu küçük veri kutusu.

Örnek:

```asm
mov rax, 10
mov rbx, 20
add rax, rbx
```

Adım adım inceleyelim:

```text
1. RAX içerisine 10 koy.
2. RBX içerisine 20 koy.
3. RBX'teki değeri RAX'e ekle.
```

Son durumda:

```text
RAX = 30
RBX = 20
```

`add rax, rbx` komutunda iki operand da register'dır.

Para benzetmesinde register, yazar kasanın para bölmesine benzer. Para doğrudan elinde değildir; fakat sana çok yakındır ve hızlıca erişebilirsin.

---

## 9. Bellekteki veriler

Register sayısı sınırlıdır. Bir programın bütün verilerini CPU içindeki birkaç küçük kutuda tutamayız.

Daha fazla veri gerektiğinde ana belleği, yani RAM'i kullanırız.

Bellekteki her konumun bir adresi vardır.

Bunu apartman adresine benzetebiliriz:

```text
Şehir → Mahalle → Sokak → Bina → Daire
```

Bilgisayar ise sayısal adresler kullanır:

```text
0x401000
0x7fffffffe120
0x555555559000
```

Assembly'de köşeli parantezler genellikle bir adresteki veriye erişildiğini gösterir.

Kavramsal örnek:

```asm
mov rax, [rbx]
```

Bu komut şöyle okunabilir:

```text
RBX'in içinde bulunan adresi kullan.
O adrese git.
Oradaki değeri RAX'e getir.
```

Burada önemli ayrım şudur:

```asm
mov rax, rbx
```

RBX'in **kendi değerini** RAX'e taşır.

```asm
mov rax, [rbx]
```

RBX'i bir **adres olarak kullanır** ve o adresteki veriyi RAX'e taşır.

Bu ayrım ileride tersine mühendislik ve exploit geliştirme sırasında çok önemlidir.

### Para benzetmesi

Belleği banka kasası gibi düşünebiliriz:

- Paraya hemen ulaşamayız.
- Önce hangi kasada olduğunu bilmeliyiz.
- Her kasanın bir numarası, yani adresi vardır.
- Kasadan alınan para işlem yapılmak üzere yakındaki alana getirilir.

---

## 10. Assembly'nin fiilleri

Assembly komutları CPU'ya veri üzerinde hangi işlemi yapacağını söyler.

En sık karşılaşacağımız temel işlem türleri şunlardır.

### Veri taşıma

```asm
mov rax, rbx
```

Bir değeri bir konumdan başka bir konuma aktarır.

### Toplama

```asm
add rax, rbx
```

İki değer üzerinde toplama yapar.

### Çıkarma

```asm
sub rax, 5
```

Bir değerden başka bir değer çıkarır.

### Çarpma

x86 üzerinde çarpma işlemleri için bağlama göre `mul` veya `imul` gibi komutlar görülebilir.

```asm
imul rax, rbx
```

### Bölme

Bölme işlemlerinde `div` veya `idiv` gibi komutlarla karşılaşabiliriz.

### Karşılaştırma

```asm
cmp rax, rbx
```

İki değeri karşılaştırır. Sonucu doğrudan bir register'a yazmak yerine CPU'nun durum bilgisini etkiler.

Daha sonra koşullu atlama komutları bu bilgiyi kullanabilir.

### Bit testi

```asm
test rax, 1
```

Değerin belirli bit özelliklerini kontrol etmek için kullanılabilir.

Örneğin bir sayının en düşük bitini kontrol etmek, sayının tek veya çift olması hakkında bilgi verebilir.

Şimdilik bu komutların bütün ayrıntılarını ezberlemek gerekmiyor. Asıl fikir şudur:

```text
Assembly komutu = Veri üzerinde yapılan küçük ve belirli bir işlem
```

---

## 11. Komutlar kaç operand alabilir?

Her assembly komutu aynı sayıda operand kullanmaz.

### Operand almayan komut

```asm
ret
```

Burada yalnızca işlem adı vardır.

### Tek operand alan komut

```asm
inc rax
```

Bu komut yalnızca `rax` üzerinde çalışır.

### İki operand alan komut

```asm
mov rax, rbx
```

Bu komut iki operand kullanır.

Genel görünüm:

```asm
işlem
işlem operand
işlem operand1, operand2
```

Bazı mimarilerde veya bazı özel komutlarda daha farklı yapılar bulunabilir. Ancak başlangıçta bu üç biçimi tanımak yeterlidir.

---

## 12. İlk assembly örneğimiz

Aşağıdaki kod iki sayıyı toplar:

```asm
mov rax, 7
mov rbx, 3
add rax, rbx
```

Adım adım çalıştıralım.

### Birinci komut

```asm
mov rax, 7
```

Durum:

```text
RAX = 7
```

### İkinci komut

```asm
mov rbx, 3
```

Durum:

```text
RAX = 7
RBX = 3
```

### Üçüncü komut

```asm
add rax, rbx
```

İşlem:

```text
RAX = RAX + RBX
RAX = 7 + 3
RAX = 10
```

Son durum:

```text
RAX = 10
RBX = 3
```

Önemli nokta:

> Intel sözdiziminde sonuç çoğunlukla soldaki operandın üzerine yazılır.

Yani:

```asm
add hedef, kaynak
```

şeklinde düşünebiliriz.

---

## 13. Neden farklı assembly dilleri var?

Assembly evrensel, tek bir dil değildir.

Çünkü farklı işlemci mimarileri farklı makine komutları kullanabilir.

Örnek mimariler:

- x86
- x86-64
- ARM
- AArch64
- MIPS
- RISC-V

Bir işlemcinin anladığı komutların tamamına **Instruction Set Architecture**, kısaca **ISA** denir.

Bunu farklı insan dillerine benzetebiliriz:

```text
Aynı niyet → Farklı dilde farklı cümle
```

Örneğin bir değeri taşıma fikri birçok mimaride vardır; ancak:

- Komutun adı,
- Kullanılan register'lar,
- Komutun ikili karşılığı,
- Operandların sırası,
- Belleğe erişim biçimi

değişebilir.

Bu yüzden “assembly biliyorum” demek genellikle hangi mimarinin assembly'sini bildiğimizi de belirtmeyi gerektirir.

---

## 14. x86 ve x86-64

Bu eğitim notlarında temel olarak **x86-64** mimarisini kullanacağız.

x86 ailesi uzun yıllar boyunca kişisel bilgisayarlarda yaygın biçimde kullanılmıştır. Mimari zaman içinde genişleyerek 16-bit, 32-bit ve 64-bit biçimlere ulaşmıştır.

Basitleştirilmiş gelişim görünümü:

```text
Eski x86 işlemciler
        ↓
16-bit x86
        ↓
32-bit x86
        ↓
64-bit x86 / x86-64
```

x86-64 için başka adlarla da karşılaşabilirsin:

```text
AMD64
Intel 64
x64
```

Telefonlarda, gömülü cihazlarda, bazı yönlendiricilerde ve modern bazı bilgisayarlarda ARM ailesiyle de sıkça karşılaşılır.

Bu durum bir mimarinin “her yerde tek başına kullanıldığı” anlamına gelmez. Donanımın kullanım alanına göre farklı mimariler tercih edilebilir.

---

## 15. Intel ve AT&T sözdizimi

Aynı x86 makine komutları farklı yazım biçimleriyle gösterilebilir.

En yaygın iki sözdizimi:

1. Intel syntax
2. AT&T syntax

Bu notlarda **Intel syntax** kullanacağız.

### Intel biçimi

```asm
mov rax, rbx
```

Genel sıra:

```text
işlem hedef, kaynak
```

### AT&T biçimi

Aynı fikir AT&T biçiminde farklı yazılabilir:

```asm
movq %rbx, %rax
```

Genel sıra:

```text
işlem kaynak, hedef
```

AT&T biçiminde ayrıca:

- Register isimlerinin önünde `%` görülebilir.
- Immediate değerlerin önünde `$` görülebilir.
- Komut adlarının sonunda veri boyutunu belirten ekler bulunabilir.

Örneğin:

```asm
# Intel
mov rax, 10
```

```asm
# AT&T
movq $10, %rax
```

İki yazım da aynı işlemci ailesinin makine komutlarını temsil edebilir. Fark, komutun metinsel olarak nasıl gösterildiğidir.

Bu eğitimde kafa karışıklığını azaltmak için Intel biçiminde ilerleyeceğiz.

---

## 16. Assembly öğrenirken nasıl düşünmeliyiz?

Assembly'yi korkutucu gösteren şey çoğunlukla komutların kendisi değil, aynı anda çok fazla yeni kavramla karşılaşmaktır.

Başlangıçta her satır için şu dört soruyu sor:

### 1. İşlem ne?

```asm
add rax, rbx
```

İşlem:

```text
add → toplama
```

### 2. Operandlar ne?

```text
rax ve rbx
```

### 3. Veriler nerede?

İki veri de register içindedir.

### 4. Sonuç nereye yazılıyor?

Intel sözdiziminde sonuç `rax` üzerine yazılır.

Bu yöntemi her komuta uygularsak assembly daha anlaşılır hale gelir.

Örnek:

```asm
sub rcx, 4
```

Sorular:

```text
İşlem ne?       → Çıkarma
Hedef ne?       → RCX
Kaynak ne?      → 4
Kaynak türü ne? → Immediate
Sonuç nerede?   → RCX
```

Matematiksel karşılığı:

```text
RCX = RCX - 4
```

---

## 17. Bölüm özeti

Bu bölümde öğrendiğimiz ana fikirler:

- Yazdığımız programlar en sonunda CPU'nun anlayabildiği makine komutlarına dönüşür.
- İkili makine kodunu insanlar için okumak ve yazmak zordur.
- Assembly, makine komutlarının okunabilir metinsel gösterimidir.
- Assembly kodu assembler tarafından makine koduna çevrilir.
- Bir komut genellikle işlem adı ve operandlardan oluşur.
- CPU veriyi doğrudan komuttan, register'dan veya bellekten alabilir.
- Immediate değer komutun içinde doğrudan bulunur.
- Register, CPU'nun içindeki küçük ve hızlı veri alanıdır.
- Bellekteki verilere adresleri üzerinden erişilir.
- `mov`, `add`, `sub`, `cmp` ve `test` gibi komutlar veri üzerinde küçük işlemler gerçekleştirir.
- Farklı CPU mimarileri farklı assembly dillerine sahiptir.
- x86 ve ARM aynı assembly dilini kullanmaz.
- Intel ve AT&T, x86 komutlarını yazmanın iki farklı sözdizimidir.
- Bu notlarda x86-64 ve Intel syntax kullanılacaktır.

Tek cümlelik özet:

> **Assembly, CPU'ya hangi küçük işlemi hangi veri üzerinde yapacağını açıkça söyleme yöntemidir.**

---

## 18. Mini alıştırmalar

### Soru 1

Assembly neden doğrudan `0` ve `1` dizileri yerine kullanılır?

---

### Soru 2

Aşağıdaki komutta işlem adı ve operandları ayır:

```asm
add rax, rbx
```

---

### Soru 3

Aşağıdaki değerlerden hangisi immediate değerdir?

```asm
mov rcx, 25
```

---

### Soru 4

Aşağıdaki iki komut arasındaki fark nedir?

```asm
mov rax, rbx
mov rax, [rbx]
```

---

### Soru 5

Aşağıdaki kod çalıştıktan sonra `RAX` kaç olur?

```asm
mov rax, 8
mov rbx, 5
add rax, rbx
```

---

### Soru 6

Aşağıdaki komutu matematiksel ifadeye çevir:

```asm
sub rdx, 3
```

---

### Soru 7

Neden x86 assembly kodu her zaman ARM işlemcide doğrudan çalışmaz?

---

### Soru 8

Intel sözdiziminde aşağıdaki komutta hedef hangisidir?

```asm
mov rdi, rax
```

---

### Soru 9

Aşağıdaki verileri uygun sınıfa yerleştir:

```text
10
RAX
[RBP]
```

Sınıflar:

```text
Immediate
Register
Memory
```

---

### Soru 10

Aşağıdaki işlemi assembly tarzında küçük adımlara ayır:

```text
7 ile 4'ü topla ve sonucu sakla.
```

---

## 19. Cevap anahtarı

<details>
<summary>Cevapları görmek için tıkla</summary>

### Cevap 1

Makine kodundaki uzun bit dizilerini okumak, yazmak ve ezberlemek insanlar için zordur. Assembly aynı komutları daha anlamlı isimlerle gösterir.

### Cevap 2

```text
İşlem adı: add
Birinci operand: rax
İkinci operand: rbx
```

### Cevap 3

```text
25
```

Çünkü değer komutun içinde doğrudan verilmiştir.

### Cevap 4

```asm
mov rax, rbx
```

RBX'in kendi değerini RAX'e taşır.

```asm
mov rax, [rbx]
```

RBX'i adres olarak kullanır ve o adresteki veriyi RAX'e taşır.

### Cevap 5

```text
RAX = 13
```

Çünkü:

```text
8 + 5 = 13
```

### Cevap 6

```text
RDX = RDX - 3
```

### Cevap 7

x86 ve ARM farklı komut setleri ve farklı makine kodları kullanır. Bir mimari için üretilen komut diğer mimari tarafından aynı şekilde anlaşılmayabilir.

### Cevap 8

```text
RDI
```

Intel sözdiziminde hedef genellikle soldaki operanddır.

### Cevap 9

```text
10    → Immediate
RAX   → Register
[RBP] → Memory
```

### Cevap 10

Olası bir çözüm:

```asm
mov rax, 7
mov rbx, 4
add rax, rbx
```

Sonuç `RAX` içinde `11` olur.

</details>

---

## 20. Kavram sözlüğü

| Kavram | Basit açıklama |
|---|---|
| Assembly | Makine komutlarının insanlar tarafından okunabilen gösterimi |
| Machine Code | CPU'nun doğrudan çalıştırdığı ikili komutlar |
| Assembler | Assembly kodunu makine koduna çeviren araç |
| Compiler | Yüksek seviyeli kodu daha düşük seviyeli bir biçime çeviren araç |
| Interpreter | Kaynak kodun davranışını çalışma sırasında gerçekleştiren program |
| JIT | Kodu çalışma sırasında makine koduna dönüştürme yöntemi |
| Instruction | CPU'ya verilen tek bir işlem komutu |
| Mnemonic | Assembly komutunun `mov`, `add` veya `sub` gibi okunabilir adı |
| Operand | Bir komutun üzerinde çalıştığı değer veya konum |
| Immediate | Komutun içinde doğrudan yazılan sabit değer |
| Register | CPU'nun içindeki küçük ve hızlı veri alanı |
| Memory | Verilerin adreslerle saklandığı ana bellek alanı |
| Address | Bellekteki bir konumu gösteren sayı |
| ISA | Bir işlemcinin desteklediği komutların ve davranışların tanımı |
| x86-64 | x86 ailesinin 64-bit mimarisi |
| Intel Syntax | x86 assembly için kullanılan yaygın yazım biçimi |
| AT&T Syntax | Aynı x86 komutlarını farklı kurallarla gösteren diğer sözdizimi |

---

## Sonraki Bölümde Ne Var?

Assembly'nin ne olduğunu ve komutların temel yapısını öğrendik. Bundan sonraki mantıklı adım x86-64 register'larını ayrıntılı biçimde incelemektir.

Önerilen sıra:

```text
1. Genel amaçlı register'lar
2. RAX, RBX, RCX ve RDX
3. RSI ve RDI
4. RSP ve RBP
5. RIP
6. 64-bit, 32-bit, 16-bit ve 8-bit register parçaları
7. Register değerlerini GDB ile izleme
```

> Komutları ezberlemekten önce verinin nerede olduğunu anlamaya çalış. Assembly'nin büyük bölümü, veriyi doğru yere taşımak ve doğru işlemden geçirmekten oluşur.
