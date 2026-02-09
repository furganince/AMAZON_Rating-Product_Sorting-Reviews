# Amazon Ürün Puanlama ve Yorum Sıralama

E-ticaret platformlarında ürün puanlamalarını zaman bazlı ağırlıklandırarak hesaplayan ve yorumları faydalılık skorlarına göre sıralayan bir proje.

## Proje Tanımı

Amazon ve benzeri e-ticaret platformlarında, ürünlerin doğru şekilde puanlanması ve yorumların sıralanması işletme performansı açısından kritik önem taşımaktadır. Eski ve güncel yorumlar farklı ağırlıklarla dikkate alınmalı, yanıltıcı yorumlar filtrelenerek en faydalı 20 yorum ön plana çıkarılmalıdır.

Bu proje, elektronik kategorisindeki en popüler ürüne verilen puanları zaman bazlı olarak ağırlıklandırarak güncel puanı hesaplayan ve **Wilson Lower Bound (WLB)** istatistiksel yöntemini kullanarak en güvenilir 20 yorumu belirlemektedir.

## Veri Seti

**Kaynak:** Amazon ürün puanları ve yorumları verisi (elektronik kategorisi)

### Veri Seti Özellikleri

- **reviewerID:** Kullanıcı ID'si
- **asin:** Ürün ID'si
- **reviewerName:** Kullanıcı adı
- **helpful:** Faydalı bulunma derecesi
- **reviewText:** Yorum metni
- **overall:** Ürün puanı (1-5 arasında)
- **summary:** Yorum özeti
- **unixReviewTime:** Yorum tarihi (Unix zaman formatında)
- **reviewTime:** Yorum tarihi (okunabilir format)
- **day_diff:** Yorum yazıldıktan sonra geçen gün sayısı
- **helpful_yes:** Yorumun faydalı bulunma sayısı
- **total_vote:** Yoruma verilen toplam oy sayısı

## Metodoloji

### GÖREV 1: Zaman Bazlı Ağırlıklı Puan Ortalaması Hesaplama

#### Adım 1: Veri Setini Yükleme ve İlk Ortalama Puanı Hesaplama
- CSV dosyasından veriler yüklenir
- Tüm puanların basit ortalaması hesaplanır

#### Adım 2: Zaman Bazlı Ağırlıklı Puan Ortalaması Hesaplama

Yorumlar gün farkına (day_diff) göre 4 çeyreğe bölünür:

**Çeyrek 1 (En Güncel):** day_diff ≤ %25 → Ağırlık: **28%**
- En yakın tarihte yapılan yorumlar
- Şu anda ürün hakkındaki düşünceler

**Çeyrek 2:** %25 < day_diff ≤ %50 → Ağırlık: **26%**
- Kısa vadeli gözlemlerin sonu

**Çeyrek 3:** %50 < day_diff ≤ %75 → Ağırlık: **24%**
- Orta vadeli deneyimler

**Çeyrek 4 (En Eski):** day_diff > %75 → Ağırlık: **22%**
- En eski yorumlar (daha az etkili)

**Formül:**
```
Ağırlıklı Puan = (Q1_avg × 0.28) + (Q2_avg × 0.26) + (Q3_avg × 0.24) + (Q4_avg × 0.22)
```

**Sonuç:** Güncel puanlar, geçmiş ortalamanın daha adil temsil edilmesini sağlar.

### GÖREV 2: Ürün Detay Sayfasında Gösterilecek 20 Yorumu Belirleme

#### Adım 1: helpful_no Değişkenini Oluşturma
```python
helpful_no = total_vote - helpful_yes
```
Toplam oyun çıkartıldığında, olumsuz oy sayısı hesaplanır.

#### Adım 2: Üç Farklı Skor Yöntemi ile Yorumları Puanlama

**1. Pozitif-Negatif Fark Skoru (score_pos_neg_diff)**
```
Score = helpful_yes - helpful_no
```
- **Avantaj:** Basit ve anlaşılır
- **Dezavantaj:** Az oylu yorumları yapay olarak yukarı taşıyabilir
- **Örnek:** 1 oy (1 faydalı, 0 olumsuz) = 1 puan ❌ (Güvenilir değil)

**2. Ortalama Faydalılık Oranı (score_average_rating)**
```
Score = helpful_yes / (helpful_yes + helpful_no)
```
- **Avantaj:** Oran temeline dayalı
- **Dezavantaj:** Az oylu yorumları %100 faydalı gösterebilir
- **Örnek:** 1 faydalı/1 toplam = %100 ⚠️ (Yanıltıcı)

**3. Wilson Lower Bound (WLB) Skoru**  **EN BAŞARILI**
```
WLB = (p̂ + z²/2n - z√(p̂(1-p̂)/n + z²/4n²)) / (1 + z²/n)

Burada:
- p̂ = helpful_yes / total_vote (faydalı oy oranı)
- n = total_vote (toplam oy sayısı)
- z = 1.96 (95% güven düzeyi için)
```

- **Avantaj:** 
  - İstatistiksel güven aralığını dikkate alır
  - Az oylu yorumları doğal olarak aşağı indirir
  - Yüksek oylu güvenilir yorumları ön plana çıkarır
  - Deneysel veri için en optimal sonuç
  
- **Dezavantaj:** Hesaplaması biraz daha karmaşık

**Karşılaştırma Örneği:**
| Yorum | helpful_yes | total_vote | pos_neg_diff | avg_rating | WLB (EN İYİ) |
|-------|-------------|------------|--------------|------------|--------------|
| A     | 1           | 1          | 1.0          | 1.00       | 0.207 ⬇️     |
| B     | 100         | 120        | -20          | 0.833      | 0.807 ⬆️     |
| C     | 1000        | 1100       | -100         | 0.909      | 0.900 ⬆️     |

**Sonuç:** WLB, güvenilir ve yüksek oylanmış yorumları doğru şekilde sıralar.

#### Adım 3: En Faydalı 20 Yorum

Wilson Lower Bound skoruna göre sıralanan ilk 20 yorum ürün detay sayfasında gösterilir.

**Bu 20 Yorumun Özellikleri:**
- Yüksek oylanma oranı (genellikle helpful_yes >> helpful_no)
- Yeterli sayıda oy almış (örn: minimum 10+ oy)
- İstatistiksel olarak güvenilir (WLB tarafından doğrulanmış)
- Kullanıcıların çoğunun ürüne olumlu baktığını gösterir

## Proje Çıktıları

### Ana Çıktılar

1. **Zaman Bazlı Ağırlıklı Ortalama Puan:** Güncel ve güvenilir ürün puanı
2. **En Faydalı 20 Yorum:** WLB skoruna göre sıralanmış yorumlar
3. **Skor Tabloları:** Üç farklı yöntemle hesaplanan skor karşılaştırmaları

### Dosyalar

```
├── Rating Product _ Sorting Reviews in Amazon.py  # Ana analiz kodu
├── amazon_review.csv                              # İnput veri seti
└── README.md                                      # Bu dosya
```

## Kullanılan Kütüphaneler

```python
- pandas: Veri manipülasyonu ve analizi
- numpy: Sayısal hesaplamalar
- scipy.stats: İstatistiksel testler (norm.ppf)
- matplotlib: Görselleştirme
```

## Kurulum

```bash
# Gerekli kütüphaneleri yükleyin
pip install pandas numpy scipy matplotlib

# Projeyi çalıştırın
python "Rating Product _ Sorting Reviews in Amazon.py"
```

## İş Değeri ve Faydaları

### E-Ticaret Platformu İçin
- **Müşteri Deneyimi Iyileştirmesi:** Yanıltıcı yorumlar filtrelenerek, gerçek ve faydalı feedbackler ön plana çıkartılır.
- **Satış Artışı:** Güvenilir yorumlar müşterilerin satın alma kararlarını olumlu etkiler.
- **Müşteri Memnuniyeti:** Objektif ve istatistiksel olarak desteklenmiş sıralama sistemi.

### Satıcılar İçin
- Ürünün objektif kalite göstergesi sağlanır.
- Yanıltıcı olumsuz yorumlardan korunma (az oylu yorumlar geri planda kalır).
- Gerçek müşteri geri bildirimleriyle ürün geliştirme fırsatı.

### Müşteriler İçin
- **Bilgilendirici Yorum Okuma:** En faydalı ve güvenilir yorumlar hemen görülebilir.
- **Karar Alma Kolaylığı:** Masif yorum içinde kaybolmadan doğru ürünü bulma.
- **Güvenilir Alışveriş Deneyimi:** Sahte ve reklam amaçlı yorumlardan korunma.

## Teknik Detaylar

### Wilson Lower Bound Tercih Sebepleri

1. **İstatistiksel Kesinlik:** Bernoulli parametresi p için güven aralığını kullanır.
2. **Oy Sayısına Duyarlılık:** Az oylu yorumlar daha aşağıda sıralanır.
3. **Dengeli Sıralama:** Hem oran hem de oy sayısını dikkate alır.
4. **Endüstri Standardı:** Reddit, Amazon ve Google gibi platformlar WLB kullanır.

### Örnek Hesaplama

```python
up = 100      # helpful_yes
down = 20     # helpful_no
n = up + down  # 120
p̂ = 100/120    # 0.8333
z = 1.96      # 95% confidence

WLB = (0.8333 + 3.8416/240 - 1.96*√(...)) / (1 + 3.8416/120)
    ≈ 0.807
```

Bu WLB skoru, "Bu yorum gerçekten %80.7 güvenlikle faydalı" demek anlamına gelir.

## Geliştirme Önerileri

- [ ] Yorum türü kategorileştirmesi (örn: "Tasarım", "Kalite", "Kargo", vb.)
- [ ] Mevsimsel ağırlıklandırma (bazı ürünler mevsime göre değişir)
- [ ] Satıcı yanıtları değişkeninin eklenmesi
- [ ] Görselleştirme paneli (matplotlib/seaborn)
- [ ] Tablo çıktısı (CSV/Excel format)

## Kaynaklar

- Wilson, B. (1927). "Probable Inference, the Law of Succession, and Statistical Inference"
- Amazon Reviews Dataset: Electronics Category
