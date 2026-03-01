---
name: Email Spam Protection
overview: Email adresini HTML source'tan tamamen kaldirip JS ile runtime'da decode/render etmek ve robots.txt eklemek. Desktop/mobile davranislarinda sifir regresyon.
todos:
  - id: html-obfuscate
    content: "index.html: 3 email referansini (identity block + contact TR + contact EN) placeholder elementlere donustur, data-e attribute ekle"
    status: completed
  - id: js-decode
    content: "js/main.js: initEmailProtection() fonksiyonu ekle - char-code array'den email decode et, DOM'a yaz, mailto: href olustur"
    status: completed
  - id: js-init
    content: "js/main.js: DOMContentLoaded ve fallback init bloklarinda initEmailProtection() cagrisini ekle"
    status: completed
  - id: robots-txt
    content: "robots.txt: Harvester botlari engelleyen kurallarla yeni dosya olustur"
    status: completed
  - id: test-verify
    content: "HTML source'ta email gorunmedigini, sayfa yuklenince dogru render oldugunu, mailto: calistigini, mobile/desktop regresyon olmadigini dogrula"
    status: completed
isProject: false
---

# Email Spam Protection

## Mevcut Durum

Email `emircansac@gmail.com` plain text olarak 3 yerde gorunuyor:

- [index.html](index.html) satir 15-17: Identity block header (`mailto:` + text)
- [index.html](index.html) satir 540: Contact section TR (`mailto:` + text)
- [index.html](index.html) satir 552: Contact section EN (`mailto:` + text)

Bunlarin hepsi bot tarayicilari tarafindan kolayca okunabilir.

---

## Strateji: JS-Only Rendering

Email HTML'de **hic** yer almayacak. Bunun yerine:

- HTML'de sadece bir placeholder element + obfuscated data attribute olacak
- JS sayfa yuklenince veriyi decode edip DOM'a email linkini yazacak
- Botlar HTML source'ta email goremeyecek

### Obfuscation Yontemi: Reversed + char-code array

Base64 artik botlar tarafindan cozulebildigi icin, daha az standart bir yontem kullanacagiz:

- Email string'i reversed halde (`moc.liamg@casnacrme`) + her karakter charCode'a cevrilip array olarak saklanacak
- JS runtime'da array'i decode edip ters cevirecek
- Bu yontem basit regex tarayicilari ve base64 decoder botlarini atlatir

---

## Degisiklikler

### 1. [index.html](index.html) - 3 email referansini kaldir

**Identity block (satir 15-17):**

- `<a href="mailto:...">` yerine `<a href="#" class="identity-email email-protected" ...>` placeholder
- `data-e` attribute'u obfuscated email icerecek
- Gorunen text: `<span class="email-text">...</span>` (bos, JS dolduracak)

**Contact section TR (satir 540):**

- `<a href="mailto:...">emircansac@gmail.com</a>` yerine `<a href="#" class="email-protected">...</a>`
- Ayni `data-e` attribute

**Contact section EN (satir 552):**

- Ayni degisiklik

### 2. [js/main.js](js/main.js) - Email decode/render fonksiyonu

`main.js` icine yeni bir bolum eklenecek (ayri dosya yerine, mevcut yapiyla tutarli olsun). Initialization blogunda cagirilacak:

```javascript
function initEmailProtection() {
    // Obfuscated: reversed char codes of "emircansac@gmail.com"
    const d = [109,111,99,46,108,105,97,109,103,64,99,97,115,110,97,99,114,105,109,101];
    const email = d.map(c => String.fromCharCode(c)).reverse().join('');
    
    document.querySelectorAll('.email-protected').forEach(el => {
        el.href = 'mai' + 'lto:' + email;
        const textEl = el.querySelector('.email-text');
        if (textEl) {
            textEl.textContent = email;
        } else {
            el.textContent = email;
        }
        el.setAttribute('aria-label', 'Email ' + email);
    });
}
```

- `mailto:` prefix de parcalanmis halde (`'mai' + 'lto:'`) bot regex'lerini atlatmak icin
- `aria-label` eklenerek screen reader erisimi korunuyor
- Tum `.email-protected` elementleri tek seferde doldurulacak

### 3. [robots.txt](robots.txt) - Yeni dosya

```
User-agent: *
Allow: /

# Block common email harvester bots
User-agent: EmailCollector
Disallow: /

User-agent: EmailSiphon
Disallow: /

User-agent: WebBandit
Disallow: /

Sitemap: https://emircansac.github.io/sitemap.xml
```

### 4. CSS - Degisiklik yok

Mevcut `.identity-email`, `.email-icon`, `.email-text` stilleri aynen korunacak. Yeni class (`email-protected`) sadece JS targeting icin; ek stil gerektirmiyor.

---

## Etki Analizi

- **Desktop**: Sifir regresyon. Email ayni yerde, ayni gorunumde, `mailto:` calisir
- **Mobile**: Sifir regresyon. Sticky header'daki email ayni sekilde gorunur
- **JS devre disi**: Email gorunmez (bilinçli trade-off: koruma > JS-disabled kullanici)
- **Screen readers**: `aria-label` ile email okunabilir
- **Mevcut CSS**: Hicbir stil degismiyor; class'lar korunuyor

---

## Dosya Ozeti


| Dosya           | Degisiklik                                     |
| --------------- | ---------------------------------------------- |
| `index.html`    | 3 email referansi placeholder'a donusur        |
| `js/main.js`    | `initEmailProtection()` fonksiyonu + init call |
| `robots.txt`    | Yeni dosya                                     |
| `css/style.css` | Degisiklik yok                                 |


