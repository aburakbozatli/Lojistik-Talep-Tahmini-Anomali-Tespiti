# 📦 Lojistik Talep Tahmini & Anomali Tespiti

Bu proje, **supply_chain_deliveries.csv** veri seti kullanılarak:  
- **Ciro tahmini** (günlük & haftalık)  
- **Günlük sipariş anomalilerinin tespiti**  
yapılmaktadır.  

---

## 🎯 Amaç

- Günlük (taktik) ve haftalık (operasyonel) **ciroyu öngörmek** → kapasite, araç, vardiya ve bütçe planı için  
- Günlük sipariş anomalilerini **müşteri × lokasyon** bazında açıklamak  
- Tahmin aralığı (Prediction Interval, PI) ile KPI’ları tek nokta yerine **band uyumu** üzerinden değerlendirmek  

---

## 📂 Veri Seti

- **Dosya:** `supply_chain_deliveries.csv`  
- **Kapsam:** 2020–2025 (~1.987 gün)  
- **Sütunlar:**  
  - `WorkDate`  
  - `Customer`  
  - `Location`  
  - `BusinessType`  
  - `OrderCount`  
  - `NumberOfPieces`  
  - `TotalRevenue`  

### 🔧 Hazırlık

- Tarih / sayısal dönüşümler  
- Günlük toplulaştırma → `Orders`, `Pieces`, `Revenue`  
- Haftalık ciro için → `W-MON` resample  

---

## 📊 EDA (Özet)

- **Trend:** Yıllar içinde taban seviye yükseliyor (kapasite/portföy veya fiyat etkisi)  
- **Haftalık mevsimsellik:** Hafta içi yüksek, hafta sonu düşük  
- **Genlik artışı:** Pik–dip farkı büyüyor → ölçekle birlikte oynaklık artıyor  
- **Öneri:** 28 günlük merkezli hareketli ortalama ile trendi pürüzsüzleştirmek  

---

## ⚙️ Modeller & Sonuçlar

### 1️⃣ Günlük Ciro — Prophet  
- **Neden:** Çoklu mevsimsellik (haftalık + yıllık) ve olası trend kırılmalarını ayrıştırır; belirsizlik bandı üretir  
- **Ayarlar:**  
  - `weekly_seasonality=True`  
  - `yearly_seasonality=True`  
  - `seasonality_mode="multiplicative"`  
  - `changepoint_prior_scale=0.10`  
  - `interval_width=0.80`  

📌 **Backtest (son 90 gün)**  
- MAE: ≈ **$38.7k**  
- MAPE: ≈ **%37.2**  
- wMAPE: ≈ **%26.7**  
- SMAPE: ≈ **%50.8**  
- Belirsizlik bandı: %80 PI, medyan genişlik = **±%27.8**  

👉 Günlük küçük/oynak günler yüzdesel hatayı şişirebilir. Prophet, hafta içi şekli ve band izlemesi için uygundur.  

---

### 2️⃣ Haftalık Ciro — Holt-Winters (log + damped)  
- **Neden:** Haftalık agregasyonda tek sezon (52) baskın; hızlı ve stabil sonuç. Log varyansı stabilize eder, damped trend aşırı eğimi törpüler.  
- **Model:**  
  - Log ölçekte  
  - `trend="add"`  
  - `damped_trend=True`  
  - `seasonal="add"`  
  - `seasonal_periods=52`  
  - (Tahminler üssel geri çevrilir)  

📌 **Backtest (son 12 hafta)**  
- MAE: ≈ **$61k**  
- MAPE: ≈ **%6.0**  
- Belirsizlik bandı: %80 PI, medyan genişlik = **±%7.1**  

👉 Kullanım: Kapasite/araç/vardiya planı için üst bant, bütçe/KPI takibi için merkez değer.  

---

### 3️⃣ Günlük Sipariş — RandomForest ile Anomali Tespiti  
- **Yaklaşım:** Beklenen siparişi (ŷ) öğren → artık (y−ŷ) → z-skoru ile standartlaştır → eşik üstü anomali  
- **Özellikler:** `lag1…lag7`, hafta günü, hafta no, ay  
- **Uyum (in-sample):** R² = **0.994**  

📌 **Eşik duyarlılığı**  
- |z| > 2.0 → 114 gün (hassas)  
- |z| > 2.5 → 64 gün (varsayılan)  
- |z| > 3.0 → 35 gün (seçici)  

📌 **Sürücü Analizi**  
- Anomali gününde **Customer × Location** kırılımı  
- Orders / Revenue / ASP / Share% ile “kim sürükledi?” açıklanır  
- Örn: *2023-01-02 → Home Depot–Chicago yüksek ASP (~$438/order) ile değer sürücüsü*  

---

## 📐 Metrikler

- **wMAPE:**  
  \[
  wMAPE = 100 \times \frac{\sum |y - \hat{y}|}{\sum |y|}
  \]  

- **SMAPE:**  
  \[
  SMAPE = \frac{100}{n} \sum \frac{2 \times |y_t - \hat{y}_t|}{|y_t| + |\hat{y}_t|}
  \]  

- **PI (Prediction Interval):**  
  - Günlük: 80% PI ≈ ŷ × (1 ± 0.278)  
  - Haftalık: 80% PI ≈ ŷ × (1 ± 0.071)  

---

## 🔑 Özet

- Prophet → Günlük planlama ve belirsizlik bandı için uygun  
- Holt-Winters → Haftalık stabil ve düşük hata oranı ile operasyonel planlama için güçlü  
- RandomForest → Sipariş anomalilerini başarılı biçimde yakaladı; sürücü analizi ile iş birimlerine aksiyon sağlıyor  
- KPI ölçümü, **band içinde kalma** ile değerlendirilmeli  

