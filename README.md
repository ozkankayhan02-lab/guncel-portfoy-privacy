# Ortak hesap kurulumu

Uygulama Supabase Auth ve PostgreSQL kullanır. İki kişi iki ayrı cihazda **aynı e-posta ve şifreyle** giriş yapar. İki cihazın da internete bağlı olması gerekir.

## 1. Supabase projesini oluşturun

1. https://supabase.com/dashboard üzerinden giriş yapın.
2. Gerekirse bir organizasyon oluşturun, ardından **New project** seçin.
3. Proje adını girin, veritabanı şifresi belirleyin, bölge seçin ve projeyi oluşturun.
4. Projedeki **SQL Editor > New query** alanına bu klasördeki `schema.sql` dosyasının tamamını yapıştırıp **Run** seçin. `Success` sonucunu kontrol edin.
5. **Authentication** bölümünde e-posta/şifre sağlayıcısını etkin tutun. E-posta doğrulama açıksa ilk kayıt sırasında gelen bağlantıyla e-postayı doğrulayın, sonra uygulamadan giriş yapın. Doğrulama bağlantısının yönleneceği Site URL için sahip olduğunuz geçerli bir web adresi kullanın.
6. **Tek oturum / single session per user** seçeneğini açmayın; aynı hesapta iki cihazın oturumu açık kalmalıdır.
7. **Connect** bölümünden **Project URL** ve **Publishable Key** kopyalayın. Anahtarlar **Settings > API Keys** bölümünde de bulunur. `anon` public anahtarı da desteklenir.

Veritabanı şifresini, `service_role` veya `sb_secret_…` anahtarını uygulamaya eklemeyin. Uygulama yalnızca public/publishable anahtar ve giriş yapan kişinin JWT'sini kullanır. SQL şemasını kurmak için public anahtar yeterli değildir; SQL Editor adımı ayrıca gereklidir.

Resmî kaynaklar: [proje oluşturma](https://supabase.com/docs/guides/getting-started/quickstarts/reactjs), [API anahtarları](https://supabase.com/docs/guides/getting-started/api-keys), [oturumlar](https://supabase.com/docs/guides/auth/sessions).

## 2. iOS bağlantısını ayarlayın

### Tekrar kayıt kontrolü ve e-posta onay sayfası

Güncellenmiş uygulamayı dağıtmadan önce:

1. SQL Editor'da `registration.sql` dosyasını çalıştırın. Kontrol onaylanmış ve onay bekleyen hesapları kapsar. Mevcut adreste uygulama signup isteği göndermez; kontrol başarısızsa da e-posta göndermeden hata gösterir.
2. `release/privacy-site/account-confirmed.html` sayfasını mevcut gizlilik sitesiyle birlikte yayımlayın.
3. Authentication > URL Configuration > Site URL değerini `https://ozkankayhan02-lab.github.io/guncel-portfoy-privacy/account-confirmed.html` yapın; bu tam adresi Redirect URLs listesine de ekleyin.
4. Confirm signup e-posta şablonundaki bağlantı `{{ .ConfirmationURL }}` kullanmalıdır. Doğrulama kapatılmamalıdır.
5. Yeni bir adresle kayıt olun, bağlantıyı açın ve “Hesabınız onaylandı” mesajından sonra uygulamadan giriş yapın. Onaylanmış ve onay bekleyen adreslerle tekrar kayıt deneyin: hesap var uyarısı görünmeli ve yeni mail gelmemeli. Süresi dolmuş bağlantıda başarı gösterilmemelidir.

Bu RPC, istenen açık hesap var uyarısı için yalnızca bir boolean döndürür; anonim istemci bu yolla bir adresin kayıtlı olup olmadığını öğrenebilir. Hesap bilgileri veya onay durumu döndürülmez. İstemci ön kontrolü aynı anda gelen iki ilk kayıt isteğini atomik olarak kilitlemez; doğrudan Auth API çağrılarını da engellemez. Sunucu düzeyinde tüm istemciler için kesin tek gönderim gerekiyorsa ayrıca hız sınırlamalı, seri çalışan bir kayıt uç noktası gerekir.

Canlı veritabanına uygulama ve sayfa yayını ayrı adımlardır; yalnızca iOS derlemesi bu değişiklikleri etkinleştirmez.

Aşağıdaki iki satırı mevcut diğer değerleri koruyarak hem `ozkan/Config/Config.local.xcconfig` (Debug), hem `ozkan/Config/Config.release.local.xcconfig` (Release) dosyasına ekleyin:

```xcconfig
INFOPLIST_KEY_SUPABASE_URL = https:/$()/PROJE_KIMLIGI.supabase.co
INFOPLIST_KEY_SUPABASE_ANON_KEY = sb_publishable_ANAHTARINIZ
```

`https:/$()/` yazımı xcconfig dosyasının `//` kısmını yorum saymasını engeller; uygulamada normal `https://` adresine dönüşür. Bu iki yerel dosya `.gitignore` kapsamındadır. Release yapılandırması `Config.release.local.xcconfig` dosyasını zaten içerir.

Uygulamayı yeniden derleyin. URL ve anahtar eksikse yerel kayıtlara geçmek yerine bağlantı kurulumu bekleniyor ekranı gösterilir.

## 3. İki cihazı bağlayın

1. İlk cihazda **Yeni hesap oluştur** seçeneğini kullanın; gerekiyorsa e-postayı doğrulayıp giriş yapın.
2. İkinci cihazda **aynı hesapla giriş yapın**; ikinci hesap oluşturmayın.
3. Önceki yerel verileri taşımak için **Ayarlar > Ortak Hesap > … Kaydı Ortak Hesaba Ekle** seçeneğini kullanın. Bu işlem mevcut bulut kayıtlarını korur; aynı UUID'ye sahip kayıtlar çoğaltılmaz. Yerel kayıtlar kendiliğinden yüklenmez ve önceki yerel yedek değiştirilmez.
4. Bir cihazda gelir, diğerinde gider ekleyin. Her iki kayıt iki cihazda da görünmelidir. Birini silip diğer cihazdaki sonucu kontrol edin.
5. Uçak modunda uygulama internet ekranına geçmelidir. Bağlantıyı açınca sunucudan güncel kayıtlar alınır.

## Davranış ve sınırlar

- Sunucu tek doğruluk kaynağıdır. Başarılı sunucu yanıtı gelmeden kayıt eklenmiş/silinmiş gösterilmez. Belirsiz bağlantı hataları tekrar denenebilir.
- Açık uygulama yaklaşık **3 saniyede bir** tam hesap görüntüsü alır. Bu bir WebSocket/Realtime bağlantısı değildir; yansıma süresi ağ ve sunucu yanıtına da bağlıdır. Arka planda sorgulama durur, öne gelince yeniden güncellenir.
- Ekleme/silme kayıt bazında yapılır. Bir cihaz bütün listeyi diğer cihazın üzerine yazmaz. Silme işaretleri, geç gelen ekleme tekrarlarının silinmiş kayıtları geri getirmesini önler.
- Kategoriler ortak hesaba aittir. Görünüm, hedef, titreşim, bildirim ve biyometrik kilit tercihleri cihaza özeldir.
- Bulut kayıtları ve bunlardan türetilen ekstreler uygulama tarafından yerel kayıtlara yazılmaz. Oturum anahtarları bu cihaza özgü Keychain kaydında tutulur. Önceki sürümün yerel kayıtları, aktarım için ayrıca korunur.
- Çıkış yalnızca bu cihazın oturumunu kapatır. Hesap silme onay gerektirir; hesabın tüm etkin bulut kayıtlarını siler ve ortak kullanıcıyı da etkiler. Önceki yerel kayıtları, dışa alınmış kopyaları ve sağlayıcı yedeklerini silmez.
- RLS kuralları her isteği `auth.uid()` ile sınırlar. Hesap silme fonksiyonu hedef kullanıcı parametresi kabul etmez.
- Tam hesap görüntüsü alındığı için çok büyük hesaplarda ağ kullanımı artar. Daha yüksek ölçek için değişiklik akışı/sayfalama ayrıca tasarlanmalıdır.
- Supabase projesinin oluşturulması, SQL'nin canlı projeye uygulanması ve iki gerçek cihazdaki doğrulama tamamlanmadan canlı kurulum hazır sayılmaz.

## Doğrulama

```sh
bash verification/check-income-expenses.sh
bash verification/check-shared-account.sh
bash verification/check-shared-account-integration.sh
```

İlk kontrol yerel kayıtların korunmasını, ortak sunucu görüntüsünün uygulanmasını ve eski içe aktarma yolunun bulut verilerini değiştirememesini kontrol eder. İkinci kontrol ağ isteği biçimini, oturum hatalarını ve çevrimdışı hataları sahte HTTP yanıtlarıyla doğrular. İki bağımsız uygulama istemcisiyle ek kontrol; eş zamanlı ekleme/silme, sunucu yanıtının kaybolması, çevrimdışı yeniden deneme, öne gelme, cihazdan çıkış ve hesap silme akışlarını sahte HTTP sunucusuyla çalıştırır. Kullanıcı verilerine ve Keychain'e yazmazlar.

SQL kontrolü PGlite'ın bellek içi PostgreSQL motorunu kullanır; gerçek Supabase projesine bağlanmaz:

```sh
npm install --prefix /tmp/shared-account-sql-check --no-save --ignore-scripts @electric-sql/pglite
PGLITE_MODULE=/tmp/shared-account-sql-check/node_modules/@electric-sql/pglite/dist/index.js node verification/check-shared-account-sql.mjs
```

Bu kontrol şemayı çalıştırır, iki cihazın eski görüntülerden yaptığı ekleme/silmeleri, güvenli tekrarları, silme işaretlerini, veri birleştirmeyi, RLS yalıtımını, anonim erişim reddini ve hesap silme kapsamını doğrular. Yönetilen Supabase Auth hizmetini veya iki gerçek ağ bağlantısının gecikmesini simüle etmez.

İmzalama gerektirmeyen iOS derlemesi:

```sh
xcodebuild -scheme ozkan -configuration Debug -sdk iphoneos -destination 'generic/platform=iOS' -derivedDataPath /tmp/shared-account-device-build CODE_SIGNING_ALLOWED=NO build
```
