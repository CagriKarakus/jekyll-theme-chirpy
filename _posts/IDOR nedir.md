#  IDOR (Insecure Direct Object Reference) Nedir?

**IDOR**, erişim kontrolü zafiyetlerinden biridir. Bir web uygulaması, kullanıcının gönderdiği girdileri (ör. `id`, `uuid`, dosya adı) yeterli yetkilendirme kontrolü yapmadan işlerse, saldırgan bu değerleri değiştirerek başkasına ait verilere erişebilir. Modern API dünyasında bu zafiyet, **BOLA (Broken Object Level Authorization)** olarak da bilinir.

---

##  Neden Oluşur?

* Nesne kimliğinin istemciden gelip güvenilir sanılması.
* Kayıt bazlı yetki kontrolünün eksikliği ("giriş yapmış mı?" kontrolü var, ama "bu kayıt bu kullanıcıya mı ait?" yok).
* İstemci tarafı kontrollerine güvenilmesi (UI'da gizlenmiş olsa da API tarafında erişim mümkün).
* Tutarsız erişim katmanları (bazı endpoint’lerde kontrol var, bazılarında yok).

---

##  Yetki İhlali Türleri

* **Yatay (Horizontal):** Aynı seviyedeki kullanıcı başka bir kullanıcının kaydını görür/değiştirir.
* **Dikey (Vertical):** Daha yüksek yetki gerektiren kaynaklara erişim sağlanır.

---

##  Tipik Örnekler

### 1. Sıralı ID ile erişim

```http
GET /api/invoices/1003 → Kullanıcı A’nın faturası
```

Saldırgan `1002` veya `1004` dener → Başka kullanıcının faturası görüntülenir.

### 2. Sahiplik kontrolü olmayan güncelleme

```http
PUT /api/profile/123
{
  "user_id": 456,
  "displayName": "Hacker"
}
```

Başkasının profili güncellenebilir.

### 3. Dosya indirme uçları

```http
GET /download?file=report_alice.pdf
```

Dosya adı parametreden alınıyor, erişim kontrolü yapılmıyor.

### 4. Mesaj okuma

```http
GET /messages/789
```

Başkasının mesajları görüntülenebilir.

---

##  Path Traversal ile Farkı

| Özellik  | IDOR                      | Path Traversal          |
| -------- | ------------------------- | ----------------------- |
| Amaç     | Yetkisiz veri erişimi     | Dosya sistemine erişim  |
| Zayıflık | Kimlik kontrolü eksikliği | Yol doğrulama eksikliği |

> Not: Bir uç nokta hem IDOR’a hem Path Traversal’a aynı anda açık olabilir.

---

##  Şifrelenmiş ID'ler

* Çoğu zaman raw data **encoding** (ör. Base64) ile gönderilir. Bu şifreleme değildir, sadece kodlamadır.
* CyberChef gibi araçlarla kolayca çözümlenebilir.
* Dolayısıyla sadece ID’yi encode etmek çözüm değildir; yetki kontrolü yapılmalıdır.

---

##  Geliştirici Perspektifi: Nasıl Önlenir?

1. **Sunucu tarafında yetkilendirme**: Her istekte kaynağın sahibinin kullanıcıyla eşleştiğini doğrula.
2. **İstemci verisine güvenme**: `user_id`, `role` gibi bilgileri istek gövdesinden değil oturum/token’dan al.
3. **Kayıt/alan bazlı ACL**: "Bu kullanıcı bu kaydı okuyabilir mi/güncelleyebilir mi?" kontrollerini merkezi yap.
4. **Varsayılan reddet (deny-by-default)**: Açıkça izin verilmedikçe erişim yok.
5. **Opak tanımlayıcılar**: UUID kullan, ama mutlaka sahiplik kontrolü ile birlikte.
6. **Tutarlı middleware**: Tüm GET/POST/PUT/DELETE uçlarında aynı yetki politikasını uygula.
7. **Test senaryoları**: Otomatik testlerde "Kullanıcı A, Kullanıcı B’nin kaynağına erişemez" senaryosu yer alsın.
8. **Hata yönetimi**: Bilgi sızdırmayan, doğru HTTP status kodları (403/404) döndür.

---

##  İyi ve Kötü Uygulama Örnekleri

###  Kötü Örnek (Node/Express – sahiplik kontrolü yok)

```javascript
app.get('/api/invoices/:id', requireAuth, async (req, res) => {
  const invoice = await Invoice.findById(req.params.id);
  if (!invoice) return res.status(404).end();
  res.json(invoice); // Kimin faturası olduğuna bakmıyor!
});
```

###  İyi Örnek (Sahiplik kontrolü var)

```javascript
app.get('/api/invoices/:id', requireAuth, async (req, res) => {
  const invoice = await Invoice.findById(req.params.id);
  if (!invoice) return res.status(404).end();
  if (invoice.userId !== req.user.id && !req.user.roles.includes('admin')) {
    return res.status(403).end();
  }
  res.json(pick(invoice, ['id','number','date','total','lines']));
});
```

###  Güncellemede istemci verisine güvenme

```javascript
app.put('/api/profile', requireAuth, async (req, res) => {
  // req.body.userId YOK SAYILIR
  const update = pick(req.body, ['displayName','bio','avatar']);
  const user = await User.updateById(req.user.id, update);
  res.json(user);
});
```

---

##  Özet

* **IDOR**, kullanıcıdan gelen nesne kimliğinin yetki doğrulaması yapılmadan işlenmesiyle ortaya çıkar.
* Modern API dünyasında aynı problem **BOLA** olarak adlandırılır.
* Çözüm: Sunucu tarafında merkezi ve tutarlı yetkilendirme kontrolleri uygulamaktır.

🔗 Daha fazla bilgi için: [OWASP Top 10 - A01:2021 Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/)
