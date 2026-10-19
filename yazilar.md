---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
description: "Yazının bir-iki cümlelik özeti (listelerde ve arama motorlarında görünür)."
kategoriler: ["SystemVerilog"]   # veya ["UVM"]
etiketler: []
draft: true                      # yayınlamaya hazır olunca false yap
---

Giriş paragrafı.

## İlk başlık

```systemverilog
module ornek;
  initial $display("Merhaba");
endmodule
```
