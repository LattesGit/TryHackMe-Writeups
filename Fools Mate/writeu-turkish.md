# Fools Mate THM | TryHackMe

> Şah mat yapmanı engelleyen bir satranç oyunu. Kulağa adil geliyor, değil mi? Tabii bu kısıtlamanın yalnızca istemci tarafında olduğunu fark edene kadar.

---

## İlk Bakış

İlk olarak Gobuster kullanarak dizin taraması gerçekleştirdim:

```bash
gobuster dir -u http://10.112.187.82 -w /usr/share/wordlists/dirb/common.txt -t 30 -x php,txt,html,js
```

Birkaç ilginç yol buldum:

- `/index.html`
- `/js/`
- `/css/`
- `/vendor/`

Ana sayfa bir satranç uygulaması olan Endgame Trainer'dı. Oyun normal görünüyordu ancak sıra şah mat yapmaya geldiğinde uygulama hamleyi engelliyordu.

---

## Kaynak Kod Analizi

Ana JavaScript dosyasını indirdim:

```bash
curl -s http://10.112.187.82/js/app.js
```

Kod içerisinde `preMoveCheck()` fonksiyonunu buldum.

Bu fonksiyon, oyuncunun hamlesini sunucuya göndermeden önce simüle ediyordu. Eğer hamle şah mat ile sonuçlanırsa işlemi engelliyor ve şu mesajı gösteriyordu:

```text
I'll shut down your PC if you play that.
```

Önemli nokta, bu kısıtlamanın tamamen tarayıcı içerisinde gerçekleştirilmesiydi.

---

## API Keşfi

JavaScript dosyasında iki API endpoint'i dikkatimi çekti:

- `/api/reset`
- `/api/move`

İlk olarak tarayıcı konsolundan oyunu sıfırladım:

```javascript
fetch('/api/reset', {method: 'POST'})
  .then(r => r.json())
  .then(console.log)
```

Sunucu şu bilgileri döndürdü:

```text
ok: true
status: ongoing
turn: w
```
## Kısıtlamayı Atlatma

Hamleyi oyun arayüzünden yapmak yerine doğrudan `/api/move` endpoint'ine
