---
title: "UVM Testbench Mimarisi: Bileşenler ve Fazlar"
date: 2026-10-19T10:00:00+03:00
description: "Bir UVM testbench'i hangi parçalardan oluşur, bu parçalar hangi sırayla kurulur ve simülasyon ne zaman biter? Çalışan küçük bir iskeletle adım adım."
kategoriler: ["UVM"]
etiketler: ["mimari", "fazlar", "objection", "temel"]
draft: false
---

UVM'e ilk bakışta sınıf sayısı göz korkutur. Oysa her UVM testbench'i aynı birkaç yapı taşının tekrar tekrar birleştirilmesinden oluşur. Bu yazıda o yapı taşlarını, birbirleriyle nasıl konuştuklarını ve **fazların** neden önemli olduğunu göreceğiz.

## Bileşen hiyerarşisi

Tipik bir testbench'in hiyerarşisi şöyledir:

```text
uvm_test_top            (benim_test)
└── env                 (benim_env)
    ├── agent           (benim_agent)
    │   ├── sequencer   (uvm_sequencer)
    │   ├── driver      (benim_driver)
    │   └── monitor     (benim_monitor)
    └── scoreboard      (benim_scoreboard)
```

| Bileşen | Görevi |
|---|---|
| **test** | Ortamı yapılandırır, hangi sequence'ın çalışacağını seçer. |
| **env** | Agent'ları, scoreboard'u ve coverage toplayıcıları bir araya getirir. |
| **agent** | Bir arayüze (ör. APB, AXI) ait sequencer, driver ve monitor'ü paketler. |
| **sequencer** | Sequence'lardan gelen transaction'ları driver'a iletir. |
| **driver** | Transaction'ı pin seviyesinde sinyallere çevirip DUT'a uygular. |
| **monitor** | DUT arayüzünü pasif olarak izler, gördüğünü transaction'a çevirip yayınlar. |
| **scoreboard** | Beklenen ile gerçekleşen sonuçları karşılaştırır. |

Bu sınıfların hepsi `uvm_component`'ten türer ve simülasyon boyunca yaşar. **Transaction** (`uvm_sequence_item`) ve **sequence** (`uvm_sequence`) ise `uvm_object`'ten türer; gerektiğinde oluşturulup tüketilen geçici nesnelerdir.

## Transaction: her şeyin başladığı yer

```systemverilog {title="benim_islem.sv"}
class benim_islem extends uvm_sequence_item;
  rand bit [7:0]  adres;
  rand bit [31:0] veri;
  rand bit        yaz;

  constraint adres_araligi { adres < 8'h40; }

  `uvm_object_utils_begin(benim_islem)
    `uvm_field_int(adres, UVM_ALL_ON)
    `uvm_field_int(veri,  UVM_ALL_ON)
    `uvm_field_int(yaz,   UVM_ALL_ON)
  `uvm_object_utils_end

  function new(string name = "benim_islem");
    super.new(name);
  endfunction
endclass
```

Alanların `rand` olması, sequence'ların bu nesneyi kısıtlar içinde rastgele üretebilmesini sağlar.

## Driver: transaction'dan sinyale

Driver, sequencer'dan bir sonraki transaction'ı ister, onu arayüze uygular ve işin bittiğini bildirir.

```systemverilog {title="benim_driver.sv"}
class benim_driver extends uvm_driver #(benim_islem);
  `uvm_component_utils(benim_driver)

  virtual benim_if vif;

  function new(string name, uvm_component parent);
    super.new(name, parent);
  endfunction

  function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    if (!uvm_config_db#(virtual benim_if)::get(this, "", "vif", vif))
      `uvm_fatal("NOVIF", "Sanal arayüz bulunamadı")
  endfunction

  task run_phase(uvm_phase phase);
    forever begin
      seq_item_port.get_next_item(req);
      @(posedge vif.clk);
      vif.adres <= req.adres;
      vif.veri  <= req.veri;
      vif.yaz   <= req.yaz;
      vif.gecerli <= 1'b1;
      @(posedge vif.clk);
      vif.gecerli <= 1'b0;
      seq_item_port.item_done();
    end
  endtask
endclass
```

## Fazlar: kim, ne zaman çalışır?

UVM, simülasyonu sabit bir sırayla ilerleyen **fazlara** böler. Her bileşen yalnızca ilgilendiği fazları yeniden tanımlar (override eder).

| Faz | Tür | Sıra | Ne yapılır? |
|---|---|---|---|
| `build_phase` | function | yukarıdan aşağı | Alt bileşenler oluşturulur, config okunur. |
| `connect_phase` | function | aşağıdan yukarı | TLM portları birbirine bağlanır. |
| `end_of_elaboration_phase` | function | aşağıdan yukarı | Topoloji yazdırılır, son ayarlar yapılır. |
| `start_of_simulation_phase` | function | aşağıdan yukarı | Simülasyon öncesi hazırlık. |
| `run_phase` | **task** | paralel | Zaman ilerler; stimulus burada üretilir. |
| `extract_phase` | function | aşağıdan yukarı | Sonuç verisi toplanır. |
| `check_phase` | function | aşağıdan yukarı | Son kontroller yapılır. |
| `report_phase` | function | aşağıdan yukarı | Özet rapor basılır. |

`build_phase`'in **yukarıdan aşağı** çalışması kritiktir: bir ebeveyn, çocuklarını ancak kendisi kurulduktan sonra oluşturabilir. Simülasyon zamanı tüketebilen tek ortak faz `run_phase`'dir (bunun yanında `reset`, `configure`, `main`, `shutdown` gibi alt run fazları da vardır, ancak çoğu proje yalnızca `run_phase` ile yetinir).

## Simülasyon ne zaman biter? Objection'lar

`run_phase` bir task olduğu için UVM'in "işin bittiğini" bilmesi gerekir. Bunu **objection** mekanizmasıyla söyleriz: en az bir bileşen itiraz (objection) kaldırdığı sürece faz devam eder; tüm itirazlar düşürüldüğünde faz sona erer.

```systemverilog {title="benim_test.sv"}
class benim_test extends uvm_test;
  `uvm_component_utils(benim_test)

  benim_env env;

  function new(string name, uvm_component parent);
    super.new(name, parent);
  endfunction

  function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    env = benim_env::type_id::create("env", this);
  endfunction

  task run_phase(uvm_phase phase);
    benim_sequence seq = benim_sequence::type_id::create("seq");
    phase.raise_objection(this);
    seq.start(env.agent.sequencer);
    phase.drop_objection(this);
  endtask
endclass
```

> **Sık yapılan hata:** `raise_objection` çağrısını unutmak. Bu durumda `run_phase` sıfır anında biter ve hiçbir stimulus uygulanmaz — simülasyon hatasız ama boş tamamlanır.

`new()` yerine `type_id::create()` kullandığımıza dikkat et. Bu, UVM **factory**'sinin bir bileşeni çalışma zamanında türetilmiş bir sınıfla değiştirebilmesini (override) sağlar; testbench kodunu değiştirmeden farklı davranışlar denemenin anahtarıdır.

## Her şeyi başlatan top modülü

```systemverilog {title="tb_top.sv"}
module tb_top;
  import uvm_pkg::*;
  `include "uvm_macros.svh"

  bit clk;
  always #5 clk = ~clk;

  benim_if vif (clk);
  benim_dut dut (.clk(clk), .adres(vif.adres), .veri(vif.veri),
                 .yaz(vif.yaz), .gecerli(vif.gecerli));

  initial begin
    uvm_config_db#(virtual benim_if)::set(null, "uvm_test_top.*", "vif", vif);
    run_test("benim_test");
  end
endmodule
```

`run_test()` çağrısı UVM'i ayağa kaldırır: test sınıfını factory ile oluşturur, fazları sırayla çalıştırır ve sonunda raporu basar. Hangi testin çalışacağını komut satırından da seçebilirsin:

```bash
+UVM_TESTNAME=benim_test +UVM_VERBOSITY=UVM_MEDIUM
```

## Özet

- **Bileşenler** (`uvm_component`) kalıcıdır ve bir ağaç oluşturur; **transaction ve sequence'lar** (`uvm_object`) geçicidir.
- **Fazlar**, her bileşenin aynı sırayla kurulmasını, bağlanmasını ve çalışmasını garanti eder.
- **Objection** olmadan `run_phase` hemen biter.
- Nesneleri her zaman **`type_id::create()`** ile oluştur ki factory override'ları çalışsın.

Sonraki yazıda sequence'ları ve kısıtlı rastgele stimulus üretimini detaylı inceleyeceğiz.
