# folder Soru - Video OCR Eşleştirme

Yayınevinden gelen soru kayıtlarını çözüm videoları ile eşleştirmek için hazırlanmış OCR ve yapısal eşleştirme pipeline'ı.

## Amaç

Bu proje şu işi yapar:

- soru veritabanı kayıtlarını video dosyaları ile eşleştirir
- önce yapısal eşleşme uygular
- sonra ihtiyaç varsa OCR tabanlı eşleşme uygular
- sonuçları CSV/XLSX olarak üretir
- gerekirse SQL update script üretimine temel hazırlar

## Çalışma Mantığı

Pipeline iki katmanlıdır:

### 1) Yapısal eşleştirme
Aşağıdaki alanlar kullanılır:

- ders
- test adı
- soru numarası
- kitap / kaynak adı
- video klasör yapısı
- video dosya adı

### 2) OCR eşleştirme
Yapısal eşleştirme yetersizse devreye girer:

- videoların ilk saniyelerinden frame alınır
- frame OCR yapılır
- soru görselleri URL üzerinden indirilir
- soru OCR yapılır
- metin benzerliği ile eşleşme skoru hesaplanır

## Sonuç Sınıfları

- `matched_high`
- `matched_medium`
- `unmapped_test`
- `test_mapped_but_question_file_missing`

## Beklenen Veri Girdileri

### `database.xlsx`
Beklenen alanlar:

- `OriginalFileName`
- `Name`
- `Name-2`
- `Id`
- `ImageId`
- `QuestionNumber`

> Not: `ImageId` alanı tam URL içermelidir.

Kullandığım sql sorgusu
```sql
select  
b."OriginalFileName",
b."Name",
bs."Name",
bsc."Id",
COALESCE(replace(m."Path", 'education.tosanalytics/public'::text, 'https://d2cqobm8wcb2vo.cloudfront.net'::text), ''::text) AS "ImageId",
bsc."QuestionNumber"
from "Books" b
left join "BookSections" bs on bs."BookId" = b."Id"
left join "BookSectionCrops" bsc on bsc."BookSectionId" = bs."Id"
left join "MediaFiles" m on m."Id" = bsc."ImageId"
where b."PublisherId" = <int>
order by b."Id",bs."StartPage", bsc."QuestionNumber"
```

Örnek:

```text
https://d2cqobm8wcb2vo.cloudfront.net/crops/8d/69c2cf422d39be9b0788a8ba_639099715225448.jpg
```

### `videos.csv`
En az şu alanlar olmalı:

- `Name`
- `FullName`

## Repo Yapısı

```text
.
├─ 01_extract_video_frames.py
├─ 02_ocr_video_frames.py
├─ 03_ocr_question_images.py
├─ 04_match_ocr.py
├─ run_all.ps1
├─ requirements.txt
├─ README.md
└─ examples/
```

## Gerekli Araçlar

- Python 3.11+
- Tesseract OCR
- FFmpeg

## Kurulum

### Python paketleri

```powershell
pip install -r requirements.txt
```

### Kontrol

```powershell
python --version
tesseract --version
ffmpeg -version
```

## Önerilen Klasör Yapısı

```text
..\folder\
  database.xlsx
  videos.csv
  ocr_work\
  pilot_ocr_work\
```

## Pilot Çalışma

Önce küçük örneklem ile test önerilir.

### Pilot output klasörü

```powershell
New-Item -ItemType Directory -Force -Path "..\folder\pilot_ocr_work\03_question_ocr"
```

### İlk 100 sorudan pilot Excel üret

```powershell
python -c "import pandas as pd; df=pd.read_excel(r'..\folder\database.xlsx', sheet_name='data-1774603564533'); df=df.head(100); df.to_excel(r'..\folder\pilot_ocr_work\pilot_database.xlsx', sheet_name='data-1774603564533', index=False)"
```

### Pilot soru OCR

```powershell
python 03_ocr_question_images.py `
  --database-xlsx "..\folder\pilot_ocr_work\pilot_database.xlsx" `
  --sheet-name "data-1774603564533" `
  --image-column "ImageId" `
  --id-column "Id" `
  --question-number-column "QuestionNumber" `
  --output-dir "..\folder\pilot_ocr_work\03_question_ocr" `
  --lang "tur+eng"
```

## Full Çalışma Akışı

### 1) Videolardan frame çıkar

```powershell
python 01_extract_video_frames.py `
  --videos-csv "..\folder\videos.csv" `
  --output-dir "..\folder\ocr_work\01_frames" `
  --times "0.5,1.0,1.5"
```

### 2) Video OCR

```powershell
python 02_ocr_video_frames.py `
  --frames-csv "..\folder\ocr_work\01_frames\video_frames.csv" `
  --output-dir "..\folder\ocr_work\02_video_ocr" `
  --lang "tur+eng"
```

### 3) Soru OCR

```powershell
python 03_ocr_question_images.py `
  --database-xlsx "..\folder\database.xlsx" `
  --sheet-name "data-1774603564533" `
  --image-column "ImageId" `
  --id-column "Id" `
  --question-number-column "QuestionNumber" `
  --output-dir "..\folder\ocr_work\03_question_ocr" `
  --lang "tur+eng"
```

### 4) OCR eşleştirme

```powershell
python 04_match_ocr.py `
  --question-ocr-csv "..\folder\ocr_work\03_question_ocr\question_ocr.csv" `
  --video-ocr-csv "..\folder\ocr_work\02_video_ocr\video_ocr_best.csv" `
  --output-dir "..\folder\ocr_work\04_match" `
  --top-n 3 `
  --min-question-ocr-len 15
```

## Üretilen Çıktılar

- `video_frames.csv`
- `video_ocr_best.csv`
- `question_ocr.csv`
- `ocr_best_matches.csv`
- `video_match_results.xlsx`
- `video_match_results.csv`

## Debug Kontrol Listesi

### `question_ocr.csv` oluşmuyorsa
Sırayla şunları kontrol et:

1. dosya yolu doğru mu
2. sheet adı doğru mu
3. `ImageId` alanı tam URL mi
4. output klasörü var mı
5. `tesseract --version` çalışıyor mu
6. `ffmpeg -version` çalışıyor mu
7. `pip install -r requirements.txt` tamam mı

### Sık hata nedenleri

- yanlış `database.xlsx` yolu
- yanlış sheet adı
- eski script argümanları ile yeni script çakışması
- Tesseract kurulu olmaması
- görsel URL erişim hatası
- eksik Python paketi

## Performans Notu

Süreyi en çok etkileyen adımlar:

- video frame OCR
- soru görsel OCR

Bu yüzden önce pilot çalışma önerilir.

## Yol Haritası

- [x] yapısal eşleştirme
- [x] video frame OCR
- [x] URL tabanlı soru OCR
- [x] OCR benzerlik eşleştirme
- [ ] SQL update script üretimi
- [ ] paralel OCR işleme
- [ ] manuel review sheet üretimi

## Lisans

İç kullanım / proje bazlı kullanım için hazırlanmıştır.
