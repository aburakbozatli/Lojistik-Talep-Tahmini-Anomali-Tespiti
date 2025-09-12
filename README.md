Lojistik Talep Tahmini & Anomali Tespiti
Kapsam: supply_chain_deliveries.csv ile ciro tahmini (günlük & haftalık) ve günlük sipariş anomalileri
Teknoloji: Python, pandas, Prophet, statsmodels (Holt-Winters), scikit-learn (RandomForest), matplotlib
Para birimi: USD
Amaç
Günlük (taktik) ve haftalık (operasyonel) ciroyu öngörerek kapasite/araç/vardiya ve bütçe planı yapmak
Günlük sipariş anomalilerini tespit edip Müşteri × Lokasyon bazında sürücülerini açıklamak
Tahmin aralığı (Prediction Interval, PI) ile KPI’ları tek nokta yerine band uyumu üzerinden değerlendirmek
Veri
Dosya: supply_chain_deliveries.csv
Sütunlar: WorkDate, Customer, Location, BusinessType, OrderCount, NumberOfPieces, TotalRevenue
Kapsam: 2020–2025 (~1.987 gün)
Hazırlık: Tarih/sayısal dönüşüm; günlük toplulaştırma (Orders, Pieces, Revenue); haftalık ciro için W-MON resample
EDA özet
Trend: Taban seviye yıllar içinde yükseliyor (kapasite/portföy veya fiyat etkisi)
Haftalık mevsimsellik: Hafta içi yüksek, hafta sonu düşük
Genlik artışı: Pik–dip farkı büyüyor → ölçekle birlikte oynaklık artıyor
Öneri: 28 günlük merkezli hareketli ortalama ile trendi pürüzsüz görün
Modeller & Sonuçlar
1) Günlük Ciro — Prophet
Neden: Çoklu mevsimsellik (haftalık+yıllık) ve olası trend kırılmalarını ayrıştırır; belirsizlik bandı üretir
Ayarlar: weekly_seasonality=True, yearly_seasonality=True, seasonality_mode="multiplicative", changepoint_prior_scale=0.10, interval_width=0.80
Backtest (son 90 gün)
MAE (Mean Absolute Error): ~$38.7k
MAPE (Mean Absolute Percentage Error): ~%37.2
wMAPE (Weighted MAPE): ~%26.7
SMAPE (Symmetric MAPE): ~%50.8
Belirsizlik
%80 PI medyan genişlik: ±%27.8 → günlük planlamada ŷ × (1 ± 0.278) bandı
Günlükte küçük/oynak günler yüzdesel hatayı şişirebilir; Prophet’i hafta içi şekli ve band izlemesi için kullan.
2) Haftalık Ciro — Holt-Winters (log + damped)
Neden: Haftalık agregasyonda tek sezon (52) baskın; hızlı ve stabil hedef. Log varyansı stabilize eder, damped trend aşırı eğimi törpüler
Model: Log ölçekte trend="add", damped_trend=True, seasonal="add", seasonal_periods=52 (tahminler üssel geri çevrilir)
Backtest (son 12 hafta)
MAE: ≈ $61k
MAPE: ≈ %6.0
Tahmin aralığı
%80 PI medyan belirsizlik: ±%7.1 → ör. $1.40M ⇒ $1.30–$1.50M
Kullanım: Kapasite/araç/vardiya planı üst banda, bütçe/KPI takibi merkeze göre; başarı band içinde kalma ile ölçülür
3) Günlük Sipariş — RandomForest ile Anomali
Yaklaşım: Beklenen siparişi (ŷ) öğren → artık (y−ŷ) → z-skoru ile standartlaştır → eşik üstü anomali
Özellikler: lag1…lag7, hafta günü, hafta no, ay
Uyum (in-sample): R² = 0.994
Eşik duyarlılığı (|z| > thr)
2.0 → 114 gün (hassas)
2.5 → 64 gün (varsayılan)
3.0 → 35 gün (seçici)
Sürücü analizi
Anomali gününde Customer × Location kırılımında Orders / Revenue / ASP / Share% ile “kim sürükledi?” açıklanır
(ör. 2023-01-02’de Home Depot–Chicago yüksek ASP (~$438/order) ile değer sürücüsü)
Metrikler
wMAPE: wMAPE = 100 * (sum(|y - y_hat|) / sum(|y|))
SMAPE (ortalama): SMAPE = (100/n) * Σ[ 2*|y_t - yhat_t| / (|y_t| + |yhat_t|) ]
PI (Prediction Interval): 80% PI ≈ y_hat × (1 ± u) — günlük u=0.278 (±%27.8), haftalık u=0.071 (±%7.1)
