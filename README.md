# QoS Routing Optimizer

**Karınca Kolonisi Optimizasyonu ve Genetik Algoritma ile çok kriterli ağ rotalama.**

250 düğümlü bir ağda kaynak ve hedef arasında rota arayan bu proje, yalnızca yol uzunluğunu değil; **gecikme**, **güvenilirlik** ve **kaynak kullanımını** birlikte değerlendirir. Amaç, trafik talebinin bant genişliği gereksinimini dikkate alarak ağırlıklı maliyeti düşük rotalar bulmak ve farklı QoS önceliklerinin rota seçimine etkisini incelemektir.

Bu depo, ekip çalışmasının Python algoritmalarını içeren sürümünün forkudur. Projenin final raporunda anlatılan **Electron, React ve Cytoscape.js masaüstü arayüzü**, [kaynak deponun `final` dalında](https://github.com/turkermenes/Bilgisayar-Aglari-Grup12/tree/final) bulunur. Bu forkun `main` dalındaki çalıştırma adımları aşağıdaki Python uygulamaları içindir.

## Temel özellikler

- CSV dosyalarından ağ topolojisi ve trafik taleplerini okuma.
- NetworkX ile düğüm ve bağlantı özelliklerini modelleme.
- **ACO:** Feromon, sezgisel geçiş ve buharlaşma ile rota arama.
- **GA:** Popülasyon, turnuva seçimi, çaprazlama, mutasyon ve elitizm ile rota arama.
- Gecikme, güvenilirlik ve kaynak maliyeti için ayarlanabilir ağırlıklar.
- Birden fazla trafik talebini sırayla değerlendirme.
- İyileşme durduğunda erken sonlandırma ve konsolda sonuç takibi.

## Problem ve maliyet modeli

Bir rotanın toplam maliyeti, üç bileşenin ağırlıklı toplamıdır:

```text
Maliyet(P) = w_delay × Gecikme(P)
           + w_reliability × GüvenilirlikMaliyeti(P)
           + w_resource × KaynakMaliyeti(P)
```

| Bileşen | Hesaplama yaklaşımı | Hedef |
| --- | --- | --- |
| Gecikme | Bağlantı gecikmeleri ve ara düğümlerin işlem sürelerinin toplamı | Azaltmak |
| Güvenilirlik maliyeti | Güvenilirlik değerlerinin negatif logaritmalarının toplamı | Azaltmak; güvenilirliği artırmak |
| Kaynak maliyeti | Her bağlantı için `1000 / bandwidth_mbps` değerlerinin toplamı | Daha yüksek kapasiteli hatları tercih etmek |

Kaynak ve hedef düğümler işlem gecikmesi toplamına dahil edilmez. Mevcut GA, güvenilirlik hesabına bütün rota düğümlerini katarken `aco_standalone.py` yalnızca ara düğümleri katar; karşılaştırmalarda bu uygulama farkı dikkate alınmalıdır.

Bu yöntemler sezgisel arama yapar; bulunan rotanın küresel optimum olduğu garanti edilmez.

## Veri seti

Depodaki mevcut dosyalar **250 düğüm**, **12.452 bağlantı** ve **30 trafik talebi** içerir. `main.py` ve `mainACO.py`, topolojiyi yönsüz graf olarak oluşturur.

| Dosya | Sütunlar | İçerik |
| --- | --- | --- |
| `data/node_data.csv` | `node_id`, `s_ms`, `r_node` | Düğüm kimliği, işlem gecikmesi ve güvenilirlik |
| `data/edge_data.csv` | `src`, `dst`, `capacity_mbps`, `delay_ms`, `r_link` | Bağlantı uçları, kapasite, gecikme ve güvenilirlik |
| `data/demand_data.csv` | `src`, `dst`, `demand_mbps` | Kaynak, hedef ve istenen bant genişliği |

CSV dosyaları noktalı virgülle (`;`) ayrılır; ondalıklı değerlerde virgül kullanılır. Ana çalıştırma dosyaları okuma sırasında ondalık virgülleri noktaya dönüştürür.

## Kurulum

Python 3 ve `pip` gereklidir. Bağımlılıklar: NetworkX, NumPy, Matplotlib, Pandas ve tqdm. Paket sürümleri özgün depoda sabitlenmediği için eklenen `requirements.txt` bir bağımlılık listesidir.

```bash
git clone https://github.com/muzafferturkoglu/qos-routing-optimizer.git
cd qos-routing-optimizer
python -m venv .venv
```

Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

macOS / Linux:

```bash
source .venv/bin/activate
```

Bağımlılıkları yükleyin:

```bash
python -m pip install -r requirements.txt
```

## Çalıştırma

Genetik Algoritma:

```bash
python -X utf8 src/main.py
```

Karınca Kolonisi Optimizasyonu:

```bash
python -X utf8 src/mainACO.py
```

Her iki komut da `data/demand_data.csv` içindeki talepleri sırayla işler. Veri dosyalarının yolları kaynak dosyalarının konumuna göre belirlenir. Konsolda rota arama ilerlemesi, bulunan yollar veya rota bulunamadığı bilgisi gösterilir. ACO ayrıca gecikme, güvenilirlik maliyeti ve kaynak maliyetini ayrı ayrı yazdırır.

### Mevcut başlangıç ayarları

| Parametre | GA (`main.py`) | ACO (`mainACO.py`) |
| --- | --- | --- |
| Popülasyon / karınca sayısı | 200 | 50 |
| Azami jenerasyon / iterasyon | 1.000 | 100 |
| Mutasyon oranı | 0,2 | - |
| Elitizm oranı | 0,05 | - |
| Turnuva büyüklüğü | 3 | - |
| Feromon etkisi (`alpha`) | - | 1,0 |
| Sezgisel bilgi etkisi (`beta`) | - | 2,0 |
| Buharlaşma | - | 0,5 |
| İyileşmesiz adım sınırı | 50 | 25 |
| Gecikme / güvenilirlik / kaynak ağırlıkları | 0,33 / 0,33 / 0,33 | 0,33 / 0,33 / 0,34 |

GA ayarları `main.py` içindeki `set_configurations(...)` ve `senaryolar` bölümlerinden değiştirilir. ACO ayarları `mainACO.py` içindeki `set_configurations(...)` çağrısından; maliyet ağırlıkları ise `optimize(...)` çağrısına verilen `weight_delay`, `weight_reliability` ve `weight_resource` parametrelerinden ayarlanır.

## Raporda incelenen senaryolar

Final raporu üç farklı QoS önceliğini ele alır:

- **Dengeli:** Gecikme, güvenilirlik ve kaynak kullanımına benzer ağırlık verilmesi.
- **Gecikme odaklı:** Gecikme ağırlığının 0,8'e çıkarılması.
- **Güvenilirlik odaklı:** Güvenilirlik ağırlığının 0,8'e çıkarılması.

Rapordaki masaüstü sürümü, Python hesaplama katmanını Electron üzerinden JSON tabanlı standart girdi/çıktı iletişimiyle çağırır; React kontrol panelleri ve Cytoscape.js ile topoloji ve rota görselleştirmesi sunar. Bu arayüzün dosyaları bu forkun `main` dalında yer almaz.

Rapordaki deney ayarları mevcut Python giriş dosyalarından farklıdır. Dolayısıyla rapordaki süreler bu sürüm için yeniden ölçülmüş sonuçlar olarak sunulmamıştır. Algoritmalar rastlantısal olduğundan farklı çalıştırmalarda farklı rotalar elde edilebilir.

## Dosya yapısı

```text
qos-routing-optimizer/
├── README.md
├── requirements.txt
├── .gitignore
├── data/
│   ├── node_data.csv
│   ├── edge_data.csv
│   └── demand_data.csv
└── src/
    ├── main.py                       # GA için ana çalıştırıcı
    ├── mainACO.py                    # ACO için ana çalıştırıcı
    ├── ga.py                         # Genetik Algoritma
    ├── aco_standalone.py              # Ana ACO uygulaması
    ├── aco_standalone_commented.py    # Açıklamalı ACO sürümü
    ├── aco_routing.py                # Alternatif ACO/CSV komut satırı sürümü
    └── aco_routing_commented.py      # Açıklamalı alternatif sürüm
```

Alternatif `aco_routing` dosyaları farklı veri okuma ve graf oluşturma davranışlarına sahiptir; yukarıdaki çalıştırma akışı `main.py` ve `mainACO.py` içindir. Bant genişliği kontrolü de iki ana algoritmada aynı uygulanmaz: ACO komşu seçiminde kapasiteyi denetler; GA başlangıç popülasyonunu süzer, ancak çaprazlama ve mutasyon sonrası aynı denetimi yeniden uygulamaz.

## Proje ekibi ve kaynak

Final raporunda yer alan ekip:

- Alaa Abdalla
- Berrinnur Sürücü
- Muzaffer Türkoğlu
- Mustafa Enes Türker
- Batuhan Tireki
- Tolga Karateke
- Hakan Kardeş
- Gülsüm Binaliyeva

Bu depo [ekibin kaynak projesinden](https://github.com/turkermenes/Bilgisayar-Aglari-Grup12) fork edilmiştir. Proje açıklaması ve ekip listesi final raporuna, çalıştırma adımları ve başlangıç ayarları bu depodaki kaynak koduna dayanır.

- [Masaüstü arayüzünü içeren final sürümü](https://github.com/turkermenes/Bilgisayar-Aglari-Grup12/tree/final)
