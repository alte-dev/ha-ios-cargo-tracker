# 📦 Home Assistant & iOS Kestirmeler (Shortcuts) Kargo Takip Paneli

iPhone cihazlara gelen kargo teslimat/doğrulama SMS'lerini **iOS Kestirmeler (Shortcuts)** otomasyonu ile otomatik olarak yakalayan; gönderici firma, teslimat kodu ve tam SMS metnini ayıklayarak **Home Assistant** dashboard'unda son 10 kargoyu tarih/saat bilgisiyle listeleyen tam otomatik takip çözümü.

## ✨ Özellikler
- **Tam Otomatik SMS Yakalama:** İçinde belirlenen anahtar kelimeler (örn. `teslim`) geçen mesajlar geldiğinde arka planda otomatik tetiklenir.
- **Akıllı Kod Ayıklama (Regex):** Mesaj metni içerisindeki 4-8 haneli sayısal veya alfanümerik teslimat kodlarını otomatik ayıklar.
- **Gizlenebilir Tam SMS Metni:** Dashboard tablosunu kalabalıklaştırmadan, ilgili satırdaki **Oku** butonuna tıklandığında açılan kutu içinde orijinal SMS metninin tamamını gösterir.
- **Çoklu Kullanıcı Desteği:** Evdeki birden fazla kişinin (örn. Eş/Aile bireyleri) telefonundan gelen kargoları tek panelde alıcı adıyla birlikte listeler.
- **Mükerrer Kayıt Koruması:** Aynı kargo kodu art arda tetiklense dahi listeye ikinci kez eklenmez.
- **Tek Tuşla Sıfırlama:** Dashboard üzerinden onay pencereli buton ile tüm kargo geçmişini temizleme imkânı sunar.
- **Kalıcı Hafıza:** Trigger-based Template Sensor yapısı sayesinde Home Assistant yeniden başlatılsa bile son 10 kargo kaydı silinmez.

---

## 🛠️ Kurulum Adımları

### 1. Home Assistant Yapılandırması (`configuration.yaml`)
Home Assistant'ın `configuration.yaml` dosyasına aşağıdaki `rest_command` ve `template` bloklarını ekleyin. `YOUR_HOME_ASSISTANT_URL` alanını kendi dış erişim adresinizle (Nabu Casa, Cloudflare Tunnel veya Reverse Proxy URL'niz) değiştirin.

```yaml
rest_command:
  kargo_listesi_sifirla:
    url: "https://YOUR_HOME_ASSISTANT_URL/api/webhook/kargo_sms_listesi"
    method: POST
    headers:
      Content-Type: "application/json"
    payload: '{"sifirla": true}'

template:
  - trigger:
      - platform: webhook
        webhook_id: kargo_sms_listesi
        allowed_methods:
          - POST
        local_only: false
    sensor:
      - name: "Kargo Kodlari Listesi"
        unique_id: kargo_kodlari_listesi
        state: "{{ 'Temizlendi' if trigger.json.sifirla | default(false) else (trigger.json.kod | default('Kod Yok')) }}"
        attributes:
          son_guncelleme: "{{ now().strftime('%d.%m.%Y %H:%M') }}"
          gecmis: >
            {% if trigger.json.sifirla | default(false) %}
              {{ [] }}
            {% else %}
              {% set mevcut = this.attributes.get('gecmis', []) %}
              {% set yeni_kod = trigger.json.kod | default('-') | string %}
              {% if mevcut | length > 0 and mevcut[0].kod | string == yeni_kod %}
                {{ mevcut }}
              {% else %}
                {% set yeni_kayit = [{
                  'kisi': trigger.json.kisi | default('Bilinmiyor'),
                  'kod': yeni_kod,
                  'firma': trigger.json.firma | default('-'),
                  'mesaj': trigger.json.mesaj | default('Mesaj içeriği yok.'),
                  'zaman': now().strftime('%d.%m.%Y %H:%M')
                }] %}
                {{ (yeni_kayit + mevcut)[:10] }}
              {% endif %}
            {% endif %}
```

> **Not:** Değişikliklerin geçerli olması için **Geliştirici Araçları > YAML** sekmesinden yapılandırmayı kontrol edip Home Assistant'ı **tamamen yeniden başlatın (Restart)**.

---

### 2. iOS Kestirmeler (Shortcuts) Kurulumu

> 📲 **Tek Tıkla İndir:** [iOS Kestirmeyi İndir (iCloud)](https://www.icloud.com/shortcuts/f2294ffc246b4f38b773656274f522d7)  
> *İndirdikten sonra kestirme içerisindeki `YOUR_HOME_ASSISTANT_URL` alanını kendi adresinizle, `kisi` alanını ise kendi adınızla güncellemeyi unutmayın.*

#### A. Manuel Kurulum (İsteğe Bağlı)
1. **Kestirmeler (Shortcuts)** uygulamasını açın ve **+** simgesiyle yeni bir kestirme oluşturun (Örn: `HA Kargo Takip`).
2. **Metin Eşleştir (Match Text)** işlemini ekleyin:
   - **Desen (Pattern):** `\b[A-Z0-9]{4,8}\b`
   - **Metin (Text):** `Shortcut Input` (Kestirme Girişi) değişkenini doğrudan seçin.
   - Seçeneklerden **Case Sensitive** (Büyük/Küçük Harf Duyarlı) ayarını kapatın.
3. **Listeden Öğe Al (Get Item from List)** işlemini ekleyin:
   - **Liste:** `Matches` (Eşleşmeler)
   - **Seçim:** `Last Item` (Son Öğe - mesaj başındaki çağrı merkezi/takip numaraları yerine sondaki teslimat kodunu yakalamak için).
4. **URL İçeriğini Al (Get Contents of URL)** işlemini ekleyin:
   - **URL:** `https://YOUR_HOME_ASSISTANT_URL/api/webhook/kargo_sms_listesi`
   - **Yöntem (Method):** `POST`
   - **Üstbilgiler (Headers):** `Content-Type` : `application/json` *(Webhook kullanıldığı için Authorization Token gerekmez)*
   - **İstek Gövdesi (Request Body - JSON):**
     - `kisi` (Text): Telefon sahibinin adı (örn. `Umut`)
     - `kod` (Text): `Item from List` değişkeni
     - `firma` (Text): `Shortcut Input` değişkenine dokunup türü **Message** olarak seçin ve alt özelliğinden **Sender**'ı işaretleyin.
     - `mesaj` (Text): Doğrudan `Shortcut Input` değişkeni (SMS'in tam metnini gönderir).

#### B. Otomasyon Tetikleyicisini Bağlama
1. Kestirmeler uygulamasında alt menüden **Otomasyon (Automation)** sekmesine geçin ve **+** simgesine dokunun.
2. **Mesaj (Message)** tetikleyicisini seçin.
3. **Mesaj İçeriği (Message Contains):** `teslim` yazın *(hem "teslim" hem "teslimat" kelimelerini yakalar; çakışma olmaması için ikinci bir tetikleyici eklemeyin)*.
4. **Gönderen (Sender):** Boş bırakın.
5. **Hemen Çalıştır (Run Immediately)** seçeneğini işaretleyip **Sonraki**'ye basın ve oluşturduğunuz kestirmeyi seçin.

---

### 3. Dashboard Kartı Kurulumu (`dashboard-card.yaml`)
1. Home Assistant ana ekranınızda sağ üstteki **Düzenle (Kalem)** simgesine ve ardından **+ Kart Ekle** butonuna basın.
2. Herhangi bir kart seçmeden sol alt köşedeki **Kod Düzenleyiciyi Göster (Show Code Editor)** / **Manuel** seçeneğine tıklayın.
3. Aşağıdaki YAML kodunu yapıştırıp kaydedin:

```yaml
type: vertical-stack
cards:
  - type: markdown
    content: |
      {% set liste = state_attr('sensor.kargo_kodlari_listesi', 'gecmis') %}
      ### 📦 Kargo Takip Paneli
      {% if liste and liste | length > 0 %}
      **Son Gelen Kod:** `{{ liste[0].kod }}`{% if liste[0].firma | trim %} ({{ liste[0].firma | trim }}){% endif %}

      **Alıcı ve Zaman:** {{ liste[0].kisi }} • {{ liste[0].zaman }}
      <details><summary>📩 Son Mesajın Tamamını Gör</summary>

      > {{ liste[0].mesaj }}
      </details>

      ---

      #### Son 10 Kargo Geçmişi
      <table style="width: 100%; border-collapse: collapse; text-align: left;">
        <thead>
          <tr style="border-bottom: 1px solid rgba(255,255,255,0.2);">
            <th style="padding: 8px 4px;">Alıcı</th>
            <th style="padding: 8px 4px; text-align: center;">Kod</th>
            <th style="padding: 8px 4px;">Firma</th>
            <th style="padding: 8px 4px;">Tarih</th>
            <th style="padding: 8px 4px; text-align: right;">Mesaj</th>
          </tr>
        </thead>
        <tbody>
          {% for kargo in liste -%}
          <tr style="border-bottom: 1px solid rgba(255,255,255,0.1);">
            <td style="padding: 8px 4px; vertical-align: top;">{{ kargo.kisi }}</td>
            <td style="padding: 8px 4px; text-align: center; vertical-align: top;"><code>{{ kargo.kod }}</code></td>
            <td style="padding: 8px 4px; vertical-align: top;">{{ kargo.firma | trim }}</td>
            <td style="padding: 8px 4px; vertical-align: top; white-space: nowrap;">{{ kargo.zaman }}</td>
            <td style="padding: 8px 4px; text-align: right; vertical-align: top;">
              <details style="text-align: left;">
                <summary style="cursor: pointer; text-align: right;">Oku</summary>
                <div style="margin-top: 6px; padding: 6px; background: rgba(255,255,255,0.05); border-radius: 4px; font-size: 0.9em; white-space: normal; word-break: break-word;">
                  {{ kargo.mesaj }}
                </div>
              </details>
            </td>
          </tr>
          {% endfor %}
        </tbody>
      </table>
      {% else %}
      Henüz kayıtlı kargo kodu yok.
      {% endif %}
  - type: button
    name: Kargo Geçmişini Temizle
    icon: mdi:delete-sweep-outline
    icon_height: 24px
    tap_action:
      action: call-service
      service: rest_command.kargo_listesi_sifirla
      confirmation:
        text: Tüm kargo kayıtları silinecek, emin misiniz?
```

## 🔒 Güvenlik Notu
Webhook uç noktası (`webhook_id`) bir parola görevi görür. Kendi sisteminize kurarken `kargo_sms_listesi` değerini tahmin edilmesi zor rastgele bir metinle (örn. `kargo_sms_9f8a7b6c5d`) değiştirmeniz önerilir.