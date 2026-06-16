# Veri Hazırlama Reçetesi (140 saatlik genel Türkçe eğitim seti)

Bu belge, M1 (genel Türkçe) modelinin LoRA ince ayarında kullanılan **~140 saatlik
dengeli Türkçe konuşma alt kümesinin** ham veriden nasıl üretildiğini adım adım,
sayılarıyla açıklar.

> **Neden ham ses değil de reçete?** Eğitim verisi, lisansları farklı üç **açık
> kaynak** veri kümesinden derlenmiştir. Bu kümeleri tek bir arşiv olarak yeniden
> dağıtmak kaynakların kullanım koşullarına aykırı olur. Bunun yerine, herkesin
> kendi yasal kopyasından **birebir aynı seti üretebilmesi** için temizleme ve
> seçim betiklerini, kurallarını ve elde edilen istatistikleri yayımlıyoruz. Tüm
> betikler `scripts/` altındadır ve hiçbiri ses dosyasının içeriğini paylaşmaz.

## 0. Kaynaklar

| Kaynak | İçerik | Edinme |
|---|---|---|
| **Common Voice Türkçe** (Mozilla) | Topluluk kaydı, çok konuşmacılı | commonvoice.mozilla.org — CC0 |
| **ISSAI Türkçe** | Okuma konuşması | Kaynağın kendi kullanım koşullarıyla |
| **OpenSLR 106 Türkçe** | Kısa Türkçe ifadeler | openslr.org — kaynağın lisansıyla |

Her kaynak kendi orijinal lisansı altında, kullanıcı tarafından ayrıca edinilmelidir.
Yolları ortam değişkenleriyle verilir (bkz. [REPRODUCIBILITY.md](REPRODUCIBILITY.md)).

## 1. Temizleme — `scripts/clean_datasets.py`

Her kaynak için uygulanan kurallar:

**Common Voice TR**
- Aynı konuşmacı (`client_id`) + aynı cümle → tekrar at (dedup).
- Metin filtresi: boş veya < 3 karakter, ya da > 300 karakter at; boşluk normalize.
- Süre filtresi: `clip_durations.tsv` ile < 1,0 sn at.
- Konuşmacı tavanı: konuşmacı başına en fazla 1.000 klip (baskınlık kırma).

**ISSAI**
- Metin filtresi (boş/çok kısa) + boşluk normalize.
- Ses kalite kontrolü (3.000 örneklik denetimle): kırpılma (clipping), > %70 sessizlik
  ve < 1,0 sn olanlar elenir; bozuk WAV at.

**OpenSLR 106 TR**
- Metin filtresi + boşluk normalize.
- FLAC → 16 kHz mono `s16` WAV dönüşümü (FFmpeg).
- Süre < 1,0 sn at.

Çıktı: kaynak başına `manifest_clean.csv` + birleşik `combined_manifest.csv`.

### Temizleme öncesi/sonrası

| Kaynak | Ham satır | Temiz satır | Temiz süre |
|---|---:|---:|---:|
| Common Voice TR | 119.325 | 61.371 | 65,2 s |
| ISSAI | 186.170 | 182.690 | 217,3 s |
| OpenSLR 106 TR | 2.513 | 2.513 | 10,0 s |
| **Toplam (havuz)** | **308.008** | **246.574** | **292,6 s** |

## 2. Dengeli 140 saatlik seçim — `scripts/build_finetune_subset.py`

Temiz havuz 292,6 saat ve ISSAI baskın. Eğitim için kaynak dağılımını
değerlendirme setine yaklaştıran **dengeli** bir alt küme seçilir:

| Kaynak | Politika | ~Süre |
|---|---|---:|
| ISSAI | 65 saate indir (rastgele, seed=42) | ~65 s |
| Common Voice TR | tamamı | ~65 s |
| OpenSLR 106 TR | tamamı | ~10 s |
| **Toplam** | dengeli | **~140 s** |

**Sızıntı kontrolü:** Değerlendirme (benchmark) cümleleri Türkçe-duyarlı
normalize edilip (İ/I katlama, küçük harf, noktalama temizliği) eğitimden
**çıkarılır**. ISSAI/OpenSLR'de konuşmacı etiketi olmadığından metin-dedup asıl
güvencedir.

Çıktı: `train_140h_manifest.csv` (tam ölçek) ve `train_30h_pilot_manifest.csv`
(140s içinden oran korunarak ~30s pilot).

## 3. Eğitim — `scripts/train_lora_whisper_v2.py`

140s manifestten eval sızıntı dedup'ı sonrası **118.257 eğitim satırı** kalır ve
`openai/whisper-large-v3` üzerine LoRA ile ince ayar yapılır:

| Ayar | Değer |
|---|---|
| Temel model | openai/whisper-large-v3 |
| LoRA rank / alpha | r=64 / alpha=128 |
| Hedef modüller | q_proj, v_proj |
| Epoch | 1 |
| Eğitim satırı | 118.257 |

Eğitim sonrası adapter temel modele birleştirilir (`scripts/merge_lora.py`) ve tek
parça model olarak yayımlanır. M2, M1 üzerine ikinci aşama (genel + `medv3` tıbbi)
LoRA ince ayarıdır.

## 4. Manifest şeması

`combined_manifest.csv` ve alt kümeler şu sütunları taşır:

```csv
audio_filepath,text,duration_sec,source,speaker_id,original_split
```

## 5. Yeniden üretim (özet)

```powershell
# 1) Temizle (kaynak yolları ortam değişkenleriyle)
python scripts/clean_datasets.py
# 2) 140s dengeli alt küme + pilot
python scripts/build_finetune_subset.py
# 3) LoRA eğitimi
python scripts/train_lora_whisper_v2.py --help
```

Sonuç sayıları ve değerlendirme sınırları için [RESULTS.md](RESULTS.md),
ortam ve komut ayrıntıları için [REPRODUCIBILITY.md](REPRODUCIBILITY.md).
