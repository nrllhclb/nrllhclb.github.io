---
title: "SystemVerilog Veri Tipleri: logic, bit ve Diziler"
date: 2026-10-12T10:00:00+03:00
description: "4-durumlu ve 2-durumlu tipler arasındaki fark, packed/unpacked diziler, dinamik diziler, kuyruklar ve ilişkisel diziler — testbench yazarken hangisini ne zaman kullanmalı?"
kategoriler: ["SystemVerilog"]
etiketler: ["veri-tipleri", "diziler", "temel"]
draft: false
---

Verilog'dan SystemVerilog'a geçen herkesin ilk karşılaştığı soru şudur: *`reg` mi, `wire` mı, yoksa `logic` mi?* Bu yazıda SystemVerilog'un veri tiplerini, doğrulama mühendisinin gözünden ele alıyoruz.

## 4-durumlu ve 2-durumlu tipler

SystemVerilog'da her bit ya **4 durum** (`0`, `1`, `X`, `Z`) ya da yalnızca **2 durum** (`0`, `1`) taşıyabilir.

| Tip        | Durum | Genişlik | İşaret   |
|------------|-------|----------|----------|
| `logic`    | 4     | 1 (vektör olabilir) | işaretsiz |
| `integer`  | 4     | 32       | işaretli |
| `bit`      | 2     | 1 (vektör olabilir) | işaretsiz |
| `byte`     | 2     | 8        | işaretli |
| `shortint` | 2     | 16       | işaretli |
| `int`      | 2     | 32       | işaretli |
| `longint`  | 2     | 64       | işaretli |

Kural basit: **DUT'a bağlanan her sinyal 4-durumlu olmalı**, çünkü başlatılmamış bir register'ın `X` değerini görmek istersin. Testbench içindeki sayaçlar, indisler ve transaction alanları ise çoğunlukla 2-durumlu olabilir; daha az bellek kullanır ve simülasyonu hızlandırır.

```systemverilog
module tipler;
  logic [7:0] veri_4d;
  bit   [7:0] veri_2d;

  initial begin
    $display("baslangic: logic=%b bit=%b", veri_4d, veri_2d);
    // baslangic: logic=xxxxxxxx bit=00000000

    veri_4d = 8'b1010_zzxx;
    veri_2d = veri_4d;          // X ve Z, 0'a dönüşür!
    $display("atama sonrasi: bit=%b", veri_2d);
    // atama sonrasi: bit=10100000
  end
endmodule
```

> **Dikkat:** 4-durumlu bir değeri 2-durumlu bir değişkene atadığında `X` ve `Z` sessizce `0` olur. DUT çıkışını bir `bit` değişkenine okuyup karşılaştırırsan hataları gözden kaçırabilirsin.

### `logic` neden `reg` ve `wire`'ın yerini aldı?

`logic`, hem `always`/`initial` bloklarında atanabilir hem de `assign` ile sürülebilir. Tek kısıtı şudur: **birden fazla sürücüsü olamaz.** Çift yönlü bir veri yolu (`inout`) gibi birden çok kaynağın sürdüğü bir net için yine `wire` kullanmalısın.

## Packed ve unpacked diziler

Boyutun değişken adından **önce** yazılması *packed*, **sonra** yazılması *unpacked* dizi anlamına gelir.

```systemverilog
logic [3:0][7:0] kelime;   // packed: 32 bitlik tek bir vektör, 4 bayt
logic [7:0]      bellek [0:255]; // unpacked: 256 elemanlı, her biri 8 bit

initial begin
  kelime = 32'hDEAD_BEEF;
  $display("%h", kelime[3]);   // de  (en anlamlı bayt)
  bellek[0] = kelime[0];       // ef
end
```

Packed diziler bellekte bitişik tutulur; bütün olarak aritmetik işlemlere girebilir ve dilimlenebilir. Unpacked diziler ise ayrı ayrı elemanlardan oluşan bir koleksiyondur.

## Testbench'in en iyi dostları: dinamik koleksiyonlar

Doğrulama tarafında boyutu derleme zamanında bilinmeyen veriyle sürekli çalışırız. SystemVerilog bunun için üç yapı sunar.

### Dinamik dizi

Boyutu çalışma zamanında `new[]` ile belirlenir.

```systemverilog
int paket[];

initial begin
  paket = new[4];               // 4 eleman
  foreach (paket[i]) paket[i] = i * 10;
  paket = new[8](paket);        // 8'e büyüt, eski değerleri koru
  $display("boyut=%0d", paket.size());  // boyut=8
end
```

### Kuyruk (queue)

`[$]` ile tanımlanır; iki uçtan ekleme/çıkarma yapılabilir. Scoreboard'larda beklenen transaction'ları tutmak için idealdir.

```systemverilog
int beklenen[$];

initial begin
  beklenen.push_back(5);
  beklenen.push_back(7);
  beklenen.push_front(1);       // {1, 5, 7}
  $display("ilk=%0d", beklenen.pop_front());  // ilk=1
  $display("kalan=%p", beklenen);             // kalan='{5, 7}
end
```

### İlişkisel dizi (associative array)

Seyrek veriler için birebirdir. Örneğin 4 GB'lık bir adres uzayını modellerken yalnızca yazılan adresler bellekte yer kaplar.

```systemverilog
bit [31:0] bellek_modeli [bit [31:0]];  // anahtar: adres
int        sayac         [string];      // anahtar: metin

initial begin
  bellek_modeli[32'h8000_0000] = 32'hCAFE_F00D;
  sayac["okuma"]++;
  sayac["yazma"] += 2;

  if (bellek_modeli.exists(32'h1234)) $display("var");
  else                                $display("adres yazılmamış");

  foreach (sayac[k]) $display("%s -> %0d", k, sayac[k]);
end
```

## Özet

- DUT sinyalleri için **`logic`**, testbench'teki sayılar için **`int`/`bit`** kullan.
- 4-durumludan 2-durumluya atamada `X`/`Z` kaybolur — karşılaştırmaları 4-durumlu yap.
- Sabit boyutlu veri için packed/unpacked dizi; değişken boyut için **dinamik dizi**, FIFO davranışı için **kuyruk**, seyrek veri için **ilişkisel dizi**.

Bir sonraki yazıda bu tipleri kullanarak UVM testbench'inin iskeletini kuracağız.
