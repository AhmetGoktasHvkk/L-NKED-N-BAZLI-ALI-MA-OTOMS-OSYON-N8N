# 💼 LinkedIn Otomasyonu

> Haftanın yapay zeka gelişmelerinden LinkedIn postu ve görseli üretir, Telegram'dan onay alarak paylaşır (n8n)

Haftada iki kez güncel yapay zeka haberlerini araştırır, "Haftanın en önemli 5 AI gelişmesi" konulu Türkçe bir LinkedIn postu yazar ve posta uygun minimalist bir görsel üretir. Metin ve görsel Telegram'dan onaya gönderilir. Kullanıcı serbest metinle geri bildirim verir; asistan bu geri bildirimin metinle mi, görselle mi, yoksa ikisiyle mi ilgili olduğunu anlar, yalnızca gereken kısmı yeniden üretir ve onaydan sonra LinkedIn'de paylaşır.

> ℹ️ Bu dosyada kimlik bilgileri bulunmaz; LinkedIn kişi kimliği **yer tutucuyla değiştirilmiştir** (`LINKEDIN_PERSON_ID_BURAYA`).

---

## 📌 Program türü

| Özellik | Açıklama |
|---|---|
| Tür | [n8n](https://n8n.io) iş akışı (workflow), `.json` |
| Kod | Görsel düğüm akışı + yapay zeka ajanları (AI Agent) |
| Tetikleyici | Zamanlayıcı (her hafta pazar ve çarşamba 05:00) |
| Yapay zeka | Perplexity (haber araştırması), GPT-4.1 (post yazarı ve revize ajanları), GPT-4.1-mini (görsel promptu, geri bildirim sınıflandırma), gpt-image-1 (1024×1024 görsel) |
| Entegrasyonlar | Telegram (onay asistanı), LinkedIn (paylaşım) |

---

## 🔄 Akış

```
Zamanlayıcı (Paz, Çar 05:00)
  → Perplexity: bu haftanın AI haberlerini bul ve özetle
  → Post Yazarı (GPT-4.1): en fazla 700 karakter, Türkçe, maddeli post
  → Görsel Prompt Yazarı (GPT-4.1-mini) → gpt-image-1: minimalist görsel
  → Telegram: görseli gönder → postu gönder ve serbest metin yanıt bekle
  → Onay Verdi Mi? (Text Classifier, GPT-4.1-mini)
       ├─ onayladı          → LinkedIn'de paylaş
       ├─ metin_onay_yok    → Metin Revize Agent → Telegram onayı → paylaş
       ├─ gorsel_onay_yok   → Görsel Revize Agent → yeni görsel
       │                      → Telegram onayı → paylaş
       └─ hem_metin_hem_gorsel_onay_yok
                            → Metin Revize Agent 2 → Görsel Revize Agent 2
                            → yeni görsel → Telegram onayı → paylaş
```

---

## ✨ Öne çıkanlar

- **Döngüde insan (human-in-the-loop):** Hiçbir post onay alınmadan paylaşılmaz. Telegram'ın "gönder ve bekle" (`sendAndWait`) özelliğiyle iş akışı kullanıcının cevabını bekler.
- **Serbest metinle geri bildirim:** Kullanıcı düğme yerine "metni kısalt, görseli daha sade yap" gibi doğal bir cümle yazar. **Text Classifier** bu cevabı 4 sınıftan birine ayırır; sınıf açıklamalarında çakışmaları önleyen kurallar vardır ("hem metin hem görsel" durumunda yalnızca o etiket seçilir).
- **Yalnızca gereken kısmın yeniden üretilmesi:** Sadece metin eleştirildiyse görsel korunur, sadece görsel eleştirildiyse metin korunur. Bu, gereksiz görsel üretim maliyetini önler.
- **Görsel ile metnin birleştirilmesi:** Merge düğümleri (`combineByPosition`), revize edilen metni ve ilgili görseli LinkedIn'e tek gönderi olarak iletir.
- **Kolay kişiselleştirme:** Konu (Perplexity promptu), yazım tarzı (Post Yazarı) ve görsel tarzı (Görsel Prompt Yazarı) birer prompt değiştirilerek uyarlanabilir.

---

## ⚠️ Bilinen sorunlar

| Sorun | Etki | Çözüm |
|---|---|---|
| Revizyondan sonraki onay mesajlarında (`Onayını Al3/4/5`) yalnızca "Onayla" düğmesi var ve cevap kontrol edilmiyor | Revize edilen post ikinci kez reddedilemez; ikinci geri bildirim verilemez | İkinci onayı da serbest metin yapıp sınıflandırıcıya geri bağlamak (döngü) |
| "Hem metin hem görsel" dalında `Create a post2`, revize edilen metin yerine **ilk** metni paylaşıyor | Kullanıcı revize metni onaylasa bile LinkedIn'e eski metin gider | Metni `$('Metin Revize Agent 2').item.json.output` yapmak |
| Görsel ve metin revize ajanları geri bildirimi her zaman ilk onay mesajından (`Onayını Al`) okuyor | Tek tur revizyon yapılabiliyor | Döngü kurulursa geri bildirimi son onay mesajından okumak |
| 700 karakter sınırı yalnızca promptta yazılı, akışta kontrol edilmiyor | Uzun postlar oluşabilir | Paylaşmadan önce uzunluk kontrolü eklemek |

---

## ⚙️ Kurulum

1. n8n'de **Workflows → Import from File** ile `LinkedIn_Otomasyonu.json` dosyasını içe aktarın.
2. Gerekli kimlik bilgilerini (Credentials) oluşturun:

    | Servis | Kimlik bilgisi türü |
    |---|---|
    | Perplexity | Perplexity API |
    | OpenAI | OpenAI API (sohbet modelleri ve görsel düğümlerinde seçin) |
    | Telegram | Telegram Bot (BotFather token) |
    | LinkedIn | LinkedIn OAuth2 (developer.linkedin.com'da "Share on LinkedIn" ve "Sign In with LinkedIn using OpenID Connect" ürünleri açık bir uygulama) |

3. Yer tutucuları kendi değerlerinizle değiştirin:

    - `CHAT_ID_NİZİ_EKLEYİN`: bütün Telegram düğümleri (chat ID'yi öğrenmek için kapalı duran `Telegram Trigger` düğümünü geçici olarak açıp botunuza mesaj atabilirsiniz)
    - `LINKEDIN_PERSON_ID_BURAYA`: dört `Create a post` düğümü (LinkedIn hesabını bağladıktan sonra listeden seçin)

4. Gerekirse zamanlayıcıyı ve promptları düzenleyip iş akışını **Active** yapın.

> 💡 LinkedIn, yapay zeka ile üretilen içeriğin yanıltıcı olmamasını ister. Paylaşmadan önce haberlerin doğruluğunu kontrol etmek bu yüzden onay adımının asıl amacıdır.

---

## 📁 Dosya yapısı

```
linkedin-otomasyonu/
├── LinkedIn_Otomasyonu.json   ← n8n iş akışı
├── Dokumantasyon.pdf          ← açıklama + promptlar
└── README.md
```

---

## 🛠️ Yapımcı

**Ahmet Mücahid Göktaş**: Bütçe ve Raporlama Uzmanı
Yapay zeka destekli raporlama ve otomasyon çalışmaları kapsamında geliştirilmiştir.
