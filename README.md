# Headless WooCommerce + Next.js: autentikacija, košarica i checkout

> Prilagođeno monorepo strukturi `apps/storefront`, `packages/commerce`, `packages/ui`, `packages/configuration`, `infra/` i `docs/`. Vodič se oslanja na javni Store API WooCommercea i namjerno ne pretpostavlja koji se WordPress auth plugin ili payment gateway koristi.

## 1. Ciljna arhitektura

Next.js aplikacija služi kao **Backend for Frontend (BFF)**. Browser razgovara samo s vlastitim `/api/*` rutama. Server-side commerce adapter poziva WooCommerce Store API. Tako se `Cart-Token`, auth sesija i eventualni provider credentials ne izlažu React kodu.

```text
Browser
  │ same-origin /api/*
  ▼
apps/storefront (Next.js App Router, BFF)
  ├── auth routes ── provider-neutral auth adapter ── WordPress / OIDC provider
  └── cart/checkout routes ── packages/commerce ── WooCommerce Store API
                                                     │
                                                     └── cart, totals, shipping, orders
```

### Uloge i povjerenje

| Dio | Odgovornost |
|---|---|
| `apps/storefront` | BFF route handlers, čitanje/pisanje browser cookieja, auth session, request validacija, CSRF/Origin provjere i response policy |
| `packages/commerce` | Framework-agnostični Store API client, transportni modeli, parsiranje/normalizacija WooCommerce odgovora i commerce operacije |
| `packages/ui` | Prikaz proizvoda, košarice i checkouta te UX stanja; nema WooCommerce tajni ni pristupa session-token vrijednostima |
| `packages/configuration` | Zajednički TypeScript, ESLint i Prettier postavke; ne sprema aplikacijske tajne |
| `infra/` | Reverse proxy, deployment, TLS, tajne u runtimeu, health checks i CI |
| `docs/architecture/` | ADR-ovi za auth provider, cart merge i payment flow |
| `docs/runbooks/` | Deploy, rollback, incidenti, provider/webhook operacije |

Važno je izbjeći ovo arhitekturno miješanje: `packages/commerce` **ne smije** uvoziti `next/headers`, `NextRequest` ili Next.js cookie helpers. Next.js sloj čita cookie, prosljeđuje token adapteru kao argument i upisuje eventualno obnovljeni token u response cookie.

## 2. Predložena struktura monorepoa

```text
apps/storefront/
  app/
    api/
      auth/
        login/route.ts
        callback/route.ts
        logout/route.ts
        me/route.ts
      cart/route.ts
      cart/items/route.ts
      cart/items/[key]/route.ts
      cart/coupons/route.ts
      cart/shipping-rates/route.ts
      cart/shipping-rates/select/route.ts
      checkout/route.ts
    (shop)/
      cart/page.tsx
      checkout/page.tsx
  src/
    auth/                 # session, provider adapter, guards
    http/                 # Origin/CSRF provjere, response/error helpers
    server/               # server-only config i BFF orchestration
packages/commerce/
  src/
    store-api/
      client.ts
      cart.ts
      checkout.ts
      types.ts
    index.ts
packages/ui/
  src/
    components/cart/
    components/checkout/
    tokens/
packages/configuration/
infra/
  docker/
  proxy/
  ci/
docs/
  architecture/
  runbooks/
```

Prilagodi točne `src/` i alias putanje postojećim workspace konvencijama. Zajednički package importira se preko internog workspace imena, npr. `@repo/commerce`, a ne preko relativnog importa iz `apps/storefront` u izvorne datoteke drugog packagea.

## 3. Konfiguracija i deployment varijable

Next.js tajne postavi u runtime okruženje aplikacije (lokalno u `apps/storefront/.env.local`, ako je to konvencija monorepoa; produkcijski preko secret managera/deployment platforme). Primjer:

```env
WC_STORE_API_URL=https://cms.example.com/wp-json/wc/store/v1
APP_ORIGIN=https://shop.example.com
AUTH_PROVIDER_ISSUER=https://identity.example.com
# Provider client secret samo ako odabrani flow to zahtijeva:
AUTH_PROVIDER_CLIENT_SECRET=...
# Session storage konekcija/ključ po potrebi:
SESSION_STORE_URL=...
```

- `WC_STORE_API_URL` mora biti konfigurirana server-side konstanta; nikad ne uzimaj host ili puni URL iz browser requesta.
- Validiraj konfiguraciju pri pokretanju aplikacije i prekini startup s jasnom serverskom greškom ako je obavezna vrijednost neispravna ili nedostaje.
- Ne koristi `NEXT_PUBLIC_` prefiks za tajne, auth tokene ni interni backend URL koji ne treba browseru.
- U produkciji koristi HTTPS. Ograniči mrežni pristup WordPress adminu i privatnim endpointima; javni Store API je namjerno customer-facing.
- BFF poziv prema WooCommerceu je server-to-server. Browseru zato nije potreban CORS pristup WooCommerceu. Ako browser ipak izravno zove CMS, to je druga arhitektura i zahtijeva zasebnu CORS/CSRF procjenu.

## 4. WooCommerce Store API i košarica

Store API je javni API za customer-facing proizvode, košaricu i checkout. Ne koristi WooCommerce REST API consumer key/secret za kupčev cart. Navedeni Store API namespace je:

```text
{WC_STORE_API_URL}
```

Primjer:

```text
https://cms.example.com/wp-json/wc/store/v1
```

Osnovne Store API operacije:

| Namjena | Metoda i Store API putanja |
|---|---|
| Dohvati cart | `GET /cart` |
| Dodaj stavku | `POST /cart/add-item` |
| Promijeni količinu | `POST /cart/update-item` |
| Ukloni stavku | `POST /cart/remove-item` |
| Primijeni/ukloni kupon | `POST /cart/apply-coupon`, `POST /cart/remove-coupon` |
| Ažuriraj customer/shipping podatke | `POST /cart/update-customer` |
| Odaberi shipping rate | `POST /cart/select-shipping-rate` |
| Checkout podaci | `GET /checkout` |
| Ažuriraj checkout polja | `PUT /checkout` |
| Kreiraj narudžbu i obradi payment | `POST /checkout` |

Za POST cart operacije i sve checkout zahtjeve potreban je Store API `Nonce` ili `Cart-Token`. U ovoj BFF arhitekturi odabire se Cart-Token, pa Store API nonce ne treba slati uz te zahtjeve. Nemoj miješati ovaj nonce s WordPress REST API `X-WP-Nonce`.

### 4.1. Cart-Token nije login token

WooCommerce Cart-Token je credential koji omogućuje rad s odgovarajućom Store API cart sesijom. On:

- služi Store API cart/checkout endpointima i može se koristiti umjesto WooCommerce cookie-based sesije;
- nije WordPress login, ne dokazuje korisnikov identitet i sam po sebi ne povezuje cart s WordPress customer računom;
- mora se tretirati kao session credential: ne šalji ga Reactu, ne vraćaj ga u JSON, ne zapisuj ga u logove i ne spremaj ga u `localStorage`/`sessionStorage`.

Uspješan odgovor na cart endpoint vraća aktualni token u `Cart-Token` response headeru. Prosljeđuj ga u sljedećem zahtjevu kao request header `Cart-Token`. Ažuriraj spremljeni token iz uspješnih cart odgovora i iz bilo kojeg drugog odgovora koji taj header vrati. Ako dođe do greške bez tog headera, nemoj token tiho obrisati ili naslijepo započeti novu košaricu: klasificiraj WooCommerce grešku i primijeni eksplicitno definiranu recovery politiku.

Store API opisuje customer/session podatke kao podatke trenutnog korisnika, ali Cart-Token sam nije dokaz WordPress identiteta. Nemoj zaključiti da je Next.js `app_session` automatski prenio WordPress login u WooCommerce.

### 4.2. Commerce adapter: token se prosljeđuje kao argument

U `packages/commerce/src/store-api/client.ts` napravi framework-agnostični client koji:

1. prima samo poznate Store API putanje ili typed operacije, ne proizvoljan URL iz requesta;
2. koristi konfigurirani `WC_STORE_API_URL`, `Accept: application/json`, JSON `Content-Type` za JSON body i `cache: "no-store"`;
3. prima opcionalni `cartToken` argument i postavlja `Cart-Token` header;
4. vraća parsirane podatke i `response.headers.get("Cart-Token")` kako bi BFF mogao ažurirati HttpOnly cookie;
5. na neuspjehu baca typed commerce grešku s HTTP statusom i sanitiziranim WooCommerce `code/message`; nemoj sve greške pretvoriti u lažni uspjeh ili praznu košaricu.

Pojednostavljen ugovor packagea:

```ts
export type StoreApiResult<T> = {
  data: T;
  cartToken?: string;
};

export async function getCart(
  cartToken?: string
): Promise<StoreApiResult<Cart>>;

export async function addCartItem(
  input: AddCartItemInput,
  cartToken?: string
): Promise<StoreApiResult<Cart>>;
```

Tipovi `Cart` i input modeli žive u `packages/commerce`; adapter ne zna ništa o Next cookies. Nemoj vraćati sirovi `Response` do UI-ja niti prosljeđivati WooCommerce `Cart-Token` header browseru.

### 4.3. BFF cookie i inicijalizacija cart sesije

`GET /api/cart` poziva `getCart(cartToken?)`. Pri prvom pozivu nema tokena; uspješan Store API odgovor treba inicijalizirati sesiju i vratiti `Cart-Token`. Route Handler postavlja token u browserov HttpOnly cookie i vraća cart JSON bez tokena.

```ts
// apps/storefront/src/server/cookies.ts
export const cartCookieOptions = {
  httpOnly: true,
  secure: process.env.NODE_ENV === "production",
  sameSite: "lax" as const,
  path: "/",
  // Bez Domain: cookie ostaje host-only.
};
```

U Route Handleru koristi `await cookies()` (aktualni async API Next.js App Routera); cookie postavi/obriši samo u Route Handleru ili Server Actionu, ne tijekom Server Component renderiranja. `GET /api/cart` nakon čitanja ili inicijalizacije cart sesije mora odgovoriti s `Cache-Control: private, no-store`; commerce `fetch` također koristi `cache: "no-store"`.

Odaberi `Max-Age`/`Expires` namjerno: bez toga browser tretira cookie kao session cookie. Cart cookie ne smije trajati dulje od odgovarajuće WooCommerce cart sesije; app session cookie treba biti usklađen s idle/absolute istekom server-side sessiona.

Za sve mutacije (`add`, `update`, `remove`, coupon, shipping, checkout):

- validiraj ulazni JSON i business pravila na serveru;
- pročitaj `cart_token` iz request cookieja, pozovi commerce adapter i proslijedi nazad odgovor bez tokena;
- ako odgovor nosi novi `Cart-Token`, ažuriraj cookie;
- vrati svježi cart/checkout prikaz, bez shared/public cachea.

Kad je proxy i browser na istoj javnoj domeni, cookie ostaje host-only za storefront. `Secure` je obavezan u produkciji; u lokalnom običnom HTTP developmentu može biti `false`. Razmotri `__Host-` prefiks u HTTPS produkciji (zahtijeva `Secure`, `Path=/` i izostavljen `Domain`), ali tada osiguraj da lokalna cookie konfiguracija koristi kompatibilno ime.

### 4.4. BFF rute i input validacija

Predloženi browser/BFF ugovor:

```text
GET    /api/cart
POST   /api/cart/items             { id, quantity, variation? }
PATCH  /api/cart/items/:key        { quantity }
DELETE /api/cart/items/:key
POST   /api/cart/coupons           { code }  # primjena kupona
DELETE /api/cart/coupons           { code }  # uklanjanje kupona
POST   /api/cart/shipping-rates    { shipping_address } # adresa i dohvat aktualnih rateova
POST   /api/cart/shipping-rates/select { packageId, rateId }
GET    /api/checkout               # WooCommerce checkout draft/podaci
PUT    /api/checkout               # payment method / order note update
POST   /api/checkout               # finalni Store API checkout
```

Rute su naš BFF API; nisu iste putanje kao WooCommerce Store API. Uskladi metode i inpute s postojećim frontend konvencijama, ali mapiraj ih eksplicitno u `packages/commerce`.

Validacije uključuju barem:

- `id` pozitivan cijeli broj, `quantity` cijeli broj prema ograničenjima kataloga i nepoznata polja odbijena ili eksplicitno filtrirana;
- variation atribute validirati po ugovoru Store API-ja; nemoj prihvatiti proizvoljan objekt i slijepo ga proslijediti;
- cart item key i coupon code provjeriti po veličini/formatu i ne koristiti ih kao URL bez encodinga;
- tijelo zahtjeva ograničiti po veličini, JSON parse greške vratiti kao `400`;
- klijentski iznosi, porezi, shipping total, product name i status plaćanja nikad nisu autoritativni.

WooCommerce vraća ažurirano stanje cijelog carta nakon uspješnih cart POST operacija. Kod `409` ili sličnog konflikta response može sadržavati ažurirani cart; mapiraj tu situaciju tako da UI može osvježiti prikaz, bez prikrivanja greške.

### 4.5. UI i novčani iznosi

`packages/ui` prikazuje `Cart` podatke koje dobiva iz storefront server/UI sloja. Client Component poziva samo `/api/cart/*`; ne uvozi `packages/commerce` server client i ne vidi cookie vrijednosti.

WooCommerce Store API iznose vraća kao stringove u najmanjoj jedinici valute, uz `currency_code` i `currency_minor_unit`. Nemoj tretirati string kao decimalnu cijenu bez konverzije, niti koristiti IEEE-754 `Number` za aritmetiku novca. Sačuvaj preciznost cijelog broja ili decimalnog tipa te formatiraj prema currency metadata. WooCommerce ostaje izvor istine za sve ukupne iznose.

UI treba prikazati loading/error/empty states, spriječiti duple submitove i nakon uspješne mutacije ažurirati stanje iz odgovora servera, ne optimistički izračunavati konačni total.

## 5. Autentikacija i korisnička sesija

Odvojeni su identitet korisnika i WooCommerce cart:

| Cookie/session | Značenje |
|---|---|
| `app_session` | Next.js aplikacijska sesija; veže zahtjev s autentificiranim identitetom kroz server-side session sloj |
| `cart_token` | WooCommerce Store API cart sesija; ne identificira WordPress korisnika |

Oba cookieja su `HttpOnly`, `Secure` u produkciji, `SameSite=Lax` ili strože gdje UX dopušta, `Path=/` i bez `Domain`, osim ako arhitektura izričito zahtijeva drugačije. Korisnički JavaScript ih ne čita.

### 5.1. Provider-neutralni auth adapter

WordPress core REST API nema generički headless username/password login endpoint za kupce. Odaberi i zabilježi provider/plugin prije implementacije login flowa; mogućnosti uključuju OIDC/OAuth identity providera s odgovarajućom WordPress integracijom ili specifičan auth plugin/custom WordPress endpoint koji održava tvoj tim.

Definiraj provider adapter u `apps/storefront/src/auth/` s ugovorom sličnim:

```ts
export type AuthenticatedIdentity = {
  subject: string; // stabilan ID iz pouzdanog providera, ne iz browsera
  email: string;
  displayName?: string;
};

export interface AuthProvider {
  authenticate(input: LoginInput): Promise<AuthenticatedIdentity | null>;
}
```

Ovaj `authenticate` oblik odgovara credential-based adapteru, ne svim OIDC flowovima. Za OIDC/OAuth obično implementiraj login-start i callback korake: generiraj i provjeri `state`/`nonce`, koristi PKCE gdje je predviđeno, validiraj redirect URI, issuer, audience, potpis i rok trajanja tokena te tek tada uspostavi aplikacijsku sesiju. Adapter mora vratiti minimalan identitet i nikad ne vjerovati `userId` koji šalje browser. Ne logiraj password, provider token ni cijelo auth tijelo. Generički primjer ne može zamijeniti pluginovu dokumentaciju i ugovor.

WooCommerce REST API consumer key/secret su integracijski credentials, ne customer login. WordPress Application Passwords služe autentikaciji integracija/REST klijenata i nisu mehanizam za customer sign-in. WordPress REST cookie authentication zahtijeva WordPress auth cookie i odgovarajući `X-WP-Nonce`; Next.js `app_session` nije zamjena za njih.

### 5.2. Session lifecycle

Preporučeni početni model je neproziran, kriptografski nasumičan session ID u `HttpOnly` cookieju, a session sadržaj i provider refresh tokeni u server-side storeu (npr. Redis ili baza). Vrijednost u cookieju ne sadrži podatke korisnika. Implementiraj:

- rotaciju session ID-a nakon uspješnog login-a i promjene privilegija;
- idle i absolute expiry, server-side opoziv i brisanje sessiona pri logoutu;
- redakciju session ID-ja iz logova i kontrolu session store pristupa;
- re-authentication/step-up pravilo za osjetljive account akcije;
- zaštitu login, registracije i password reset endpointa rate limitingom.

Potpisani JWT može biti alternativa, ali sam potpisani stateless JWT ne omogućuje trenutno opozivanje bez denylist/session-version sloja. Ako se bira, definiraj kratko trajanje, issuer/audience/algorithm allowlist, rotaciju ključa i jasan logout/expiry ugovor. Nemoj koristiti neprovjereni `AUTH_SECRET!` fallback ili hardkodiran tajni ključ.

### 5.3. Auth BFF rute

```text
GET|POST /api/auth/login     # GET može započeti redirect-based flow
GET      /api/auth/callback  # samo ako provider koristi redirect/callback
POST     /api/auth/login     # credential-based flow
POST /api/auth/logout
GET  /api/auth/me
```

- Login body validiraj, primijeni rate limiting i Origin/CSRF zaštitu; vrati generičku grešku za netočne vjerodajnice.
- Tek nakon potvrde providera izdaj novu app session i postavi cookie.
- `/me` dohvaća trenutni identitet iz provjerene sesije; gdje treba, osvježi profil iz autoritativnog providera.
- Logout opozove server-side sesiju i obriše `app_session`. Odvojeno definiraj zadržava li se guest cart; uobičajeno je logoutom ne brisati cart bez izričite korisničke namjere.
- Reset lozinke, registracija i account recovery ovise o provideru; ne simuliraj ih lokalno ako je identity provider njihov izvor istine.

## 6. Cart nakon login-a i merge politika

Guest cart ostaje dostupna preko postojećeg Cart-Tokena nakon što Next.js izda `app_session`. Samo kreiranje Next.js sesije ne mijenja WooCommerce identitet, ne prenosi WordPress login i ne garantira spajanje s postojećom customer košaricom.

Prije nego podržiš prijavljeni customer cart ili merge:

1. izaberi kako se pouzdano veže provider `subject` s WooCommerce customer ID-jem;
2. implementiraj/odaberi WooCommerce integraciju koja uspostavlja podržanu autentificiranu customer sesiju;
3. definiraj konfliktna pravila (duplicate product, variation, količine, rasprodani proizvodi, kuponi, shipping);
4. merge izvedeš server-side, zatim ponovno dohvatiš konačni cart iz WooCommercea;
5. testiraš login/logout, više uređaja i istovremene zahtjeve.

Ne čitaj niti mijenjaj WooCommerce session tablice direktno i ne tretiraj Cart-Token kao dokaz vlasništva nad korisničkim računom. Ako nema podržane customer-session integracije, dokumentiraj da login utječe na account funkcije, a cart ostaje guest/session cart.

## 7. Checkout i plaćanje

Checkout nije samo proxy koji prima proizvoljan JSON. Store API ugovor uključuje:

- `GET /checkout`: vraća checkout podatke i draft order za trenutni cart;
- `PUT /checkout`: ažurira checkout podatke kao što su `payment_method`, `order_notes` i plugin-specific `additional_fields`; `__experimental_calc_totals` je označen kao experimental i ne treba ga tretirati kao stabilan dugoročni ugovor;
- `POST /checkout`: finalni zahtjev s obaveznim `billing_address`, `shipping_address` i `payment_method`; podržava `payment_data` array, opcionalne `customer_note`, `customer_password`, `expected_total` i plugin extension podatke.

Preporučeni slijed:

1. Dohvati trenutni cart i checkout podatke.
2. Validiraj adrese na BFF-u i pošalji ih odgovarajućoj Store API operaciji za customer/cart podatke.
3. Dohvati shipping rates, prikaži ih iz WooCommercea i pošalji odabir putem Store API-ja.
4. Prikaži WooCommerceov osvježeni cart total, poreze, popuste i shipping.
5. Prikaži samo payment metode koje checkout/provider podržava.
6. Pozovi `POST /checkout` s validiranim payloadom; `expected_total` može se koristiti kao mismatch zaštita kada UI zna potvrđeni total (vrijednost je string u minor units), ali ne zamjenjuje server-side izračun ni payment confirmation.
7. Obradi gatewayov rezultat, redirect/3DS flow i asinkrono konačno stanje prema ugovoru konkretnog gatewaya.

`payment_data` nije univerzalan format: svaki gateway može zahtijevati vlastita polja i browser-side SDK/tokenizaciju. Koristi gatewayovu službenu integracijsku dokumentaciju; nikad ne prosljeđuj PAN/CVC kroz Next.js ili vlastite logove ako gateway nudi tokenizirani/hosted flow. Ne prihvaćaj payment method ID samo zato što je klijent poslao string — validiraj ga prema aktivnim/ponuđenim metodama na serveru.

Ne računaj total, porez ili shipping u UI-ju kao autoritativne iznose. `expected_total` služi otkrivanju promjene total-a, ne dokazuje da je uplata izvršena. Redirect URL ni browserov success ekran nisu dokaz plaćanja: konačan status provjeri preko pouzdanog server-to-server gateway odgovora ili webhooka, uz provjeru webhook potpisa, događaja i narudžbe.

Za timeout/retry POST checkouta definiraj idempotency/reconciliation ponašanje prema mogućnostima WooCommercea i gatewaya. Nemoj automatski ponoviti payment request ako ne znaš je li prvi zahtjev već kreirao narudžbu ili pokrenuo naplatu.

## 8. Sigurnosni zahtjevi za BFF

### CSRF, CORS i Origin

`SameSite=Lax` je koristan sloj, ali nije potpuna CSRF zaštita. Za sve mutacijske same-origin BFF rute, uključujući login/logout i checkout:

- provjeri `Origin` prema točnoj allowlisti (`APP_ORIGIN`), uz pažljivo definirano ponašanje za zahtjeve bez `Origin`;
- koristi CSRF token/double-submit mehanizam gdje threat model ili klijentski flow to traži;
- odbij neočekivane cross-origin zahtjeve i ne reflektiraj proizvoljan CORS origin uz credentials;
- ne baziraj sigurnosnu provjeru samo na `Referer` ili `Host` koji nije validiran kroz deployment konfiguraciju.

### Ostala pravila

- Svaki user-controlled input provjeri na granici BFF-a; commerce package dodatno tipizira outbound input.
- Postavi `Cache-Control: private, no-store` za cart, checkout, auth, account i personalizirane odgovore. Ne dopusti CDN/reverse proxy cache za te rute.
- Ne šalji cart token, app session, auth/password ili payment data u logove, analytics, tracing baggage ili error reportove.
- Razlikuj invalid input (`400`), neautoriziran zahtjev (`401`), zabranjen pristup (`403`), WooCommerce conflict (`409`) i downstream grešku (`502/503`). Klijentu vrati sanitiziran error; server log zadrži correlation ID i tehnički uzrok bez tajni/PII.
- Koristi timeout za CMS/provider pozive i eksplicitno obradi timeout, neispravan JSON i downstream unavailable slučajeve.
- Server-side pozive ograniči na konfigurirani CMS host, HTTPS u produkciji; nikad ne sastavljaj backend host iz browser inputa (SSRF).
- Ograniči payload veličinu i rate-limit auth, cart abuse te checkout prema volumenu i infrastrukturi.
- WC REST API admin credentials, OIDC client secret i session store pristup ne pripadaju browser bundleu.
- Ne izlaži `Cart-Token` u JSON, custom browser-readable headeru ili UI erroru.

## 9. Reverse proxy i mrežna topologija

Jedna javna storefront domena olakšava first-party cookie politiku:

```text
https://shop.example.com/          -> Next.js
https://cms.example.com/            -> WordPress (prema javnosti i potrebi)
```

Reverse proxy može proslijediti CMS zahtjeve prema internom WordPress hostu, ali browser i dalje treba koristiti Next.js `/api/*` za cart/checkout. Nemoj izložiti privatne WordPress/admin rute samo zato što je `/wp-json` proxyan. Ograniči koje CMS putanje su javne, podesi TLS i forwarding headers, zaštiti admin i provjeri da proxy ne cacheira auth/cart/checkout.

Odvojene `shop` i `cms` domene ne zahtijevaju browser CORS za server-to-server BFF pozive. Cookieji se postavljaju na storefront host; ne postavljaj široki cookie `Domain` bez potrebe. U `infra/` dokumentiraj runtime egress, health check, TLS renewal, secret injection i rollback korake.

## 10. Testiranje

### `packages/commerce`

- Unit testiraj URL sastavljanje, Cart-Token request/response header, JSON/error parsiranje i timeout ponašanje.
- Contract testiraj cart add/update/remove, coupon, shipping i checkout payload prema stvarnoj WooCommerce verziji/pluginima.
- Provjeri da nema importova iz Next.js/React/browser slojeva.

### `apps/storefront`

- Testiraj cookie absent → inicijalizacija → HttpOnly Set-Cookie → nastavak cart sesije.
- Testiraj token refresh iz svih relevantnih cart/checkout odgovora i da token nikad nije u JSON/logu.
- Testiraj schema validation, Origin/CSRF deny/allow, rate limit, cache headers i downstream error mapping.
- Testiraj login session rotation/revocation/logout, expiry i autorizaciju korisničkih account endpointa.
- End-to-end testiraj guest cart refresh, login/logout ponašanje, shipping/coupon i gateway sandbox payment.
- Testiraj race/retry scenarije checkouta, uključujući timeout nakon što je downstream možda već kreirao order.

### `packages/ui`

- Testiraj loading/error/empty states, inline validation, dupli submit i prikaz currency precision.
- Provjeri da UI ne importa server-only commerce/auth module i da u browser bundleu nema session/cart tokena.

## 11. Runbooks i produkcijska kontrolna lista

Dodaj u `docs/architecture/` ADR-ove za odabrani identity provider, session store, guest/customer cart pravila i payment flow. U `docs/runbooks/` dokumentiraj deploy/rollback, provider credentials rotation, WooCommerce outage, Cart-Token/session oporavak i webhook incident.

Prije produkcije potvrdi:

- [ ] Store API i gateway verzije/pluginovi su podržani i testirani zajedno.
- [ ] Cart radi za gosta nakon refreshanja i token ostaje HttpOnly.
- [ ] Cart/checkout/auth odgovori imaju `no-store`; proxy/CDN ih ne cacheira.
- [ ] Cart-Token se prosljeđuje samo server-side i response token se rotira/ažurira.
- [ ] Auth provider je odabran; login, expiry, logout i server-side revoke su testirani.
- [ ] Dokumentirano je da app auth i WooCommerce customer session nisu automatski ista sesija.
- [ ] Cart merge pravila su implementirana i testirana ili je jasno navedeno da merge nije podržan.
- [ ] Login/reset/checkout imaju rate-limit i Origin/CSRF zaštitu.
- [ ] Checkout totals dolaze iz WooCommercea; selected payment method je server-side validiran.
- [ ] Gateway redirect/webhook signature/status provjera je testirana; browser redirect nije jedini dokaz plaćanja.
- [ ] Logovi, tracing i error monitoring redaktiraju tokene, vjerodajnice, payment podatke i osjetljive osobne podatke.
- [ ] TLS, secret management, health checks, backup i rollback su provjereni u `infra/` runbooku.

## 12. Ispravci u odnosu na početni nacrt

| Početna tvrdnja/primjer | Preciznija uputa |
|---|---|
| Jedan generički `app_session` JWT je dovoljan za produkcijsku auth sesiju. | Provider potvrđuje identitet; preporučena je opaque server-side sesija s rotacijom i opozivom. Stateless JWT ima eksplicitne trade-offe i nije sam po sebi opoziv. |
| Cart-Token i WordPress user auth mogu se držati odvojeno, ali prijava je dovoljna za account cart. | Cart-Token identificira cart sesiju, ne WordPress korisnika. Za customer cart treba podržana WordPress/WooCommerce identity integracija. |
| Svi checkout pozivi su GET/POST. | Checkout podržava `GET`, `PUT` i `POST`; `POST` obrađuje narudžbu/plaćanje. `PUT` ugovor i eksperimentalna polja moraju se koristiti prema aktualnoj dokumentaciji. |
| Proxy samo prosljeđuje checkout JSON. | Server mora validirati adrese, odabranu payment metodu, gateway-specific payment data, total mismatch i greške/retry. |
| Token cookie se ažurira samo u get/add primjerima. | Sačuvaj novi `Cart-Token` iz svakog relevantnog cart/checkout odgovora, bez slanja tokena browseru. |
| `try/catch` bez logiranja vraća isti 500 za sve greške. | Klasificiraj validacijske/downstream/timeout greške, logiraj sanitiziran tehnički uzrok s correlation ID-jem i vrati siguran status. |
| Istoj javnoj domeni treba proxyirati `/wp-json/*`. | To je opcionalna topologija; BFF može zvati CMS server-side. Ograniči izložene CMS putanje i zadrži browser na `/api/*`. |
| `Number(total_price) / 10**minorUnit` je dobar način financijskog računanja. | Sačuvaj minor-unit preciznost; `Number` koristi samo uz dokazanu sigurnu granicu ili decimalni/integer pristup. |

## 13. Službena dokumentacija

- [WooCommerce Store API — overview](https://developer.woocommerce.com/docs/apis/store-api/)
- [WooCommerce Store API — Cart](https://developer.woocommerce.com/docs/apis/store-api/resources-endpoints/cart/)
- [WooCommerce Store API — Cart Tokens](https://developer.woocommerce.com/docs/apis/store-api/cart-tokens/)
- [WooCommerce Store API — Nonce Tokens](https://developer.woocommerce.com/docs/apis/store-api/nonce-tokens/)
- [WooCommerce Store API — Checkout](https://developer.woocommerce.com/docs/apis/store-api/resources-endpoints/checkout/)
- [WordPress REST API — Authentication](https://developer.wordpress.org/rest-api/using-the-rest-api/authentication/)
- [Next.js `cookies`](https://nextjs.org/docs/app/api-reference/functions/cookies)
- [Next.js Route Handlers](https://nextjs.org/docs/app/api-reference/file-conventions/route)

API ugovore i gateway behavior provjeri ponovno pri nadogradnji WooCommercea ili payment/auth plugina; Store API ekstenzije i gateway payloadi ovise o instaliranim verzijama.

## 14. Implementacija korak po korak: funkcionalni primjeri koda

Primjeri ispod zajedno čine minimalan BFF za guest cart i bankovni transfer (`bacs`). U monorepu prilagodi samo workspace package name i konfigurirane alias putanje. Primjer namjerno ne uključuje gateway-specifičan frontend SDK: kartična plaćanja i OIDC zahtijevaju dodatni provider/plugin ugovor, opisan u odgovarajućim koracima.

### 14.1. Env konfiguracija i tipovi

**Datoteka: `apps/storefront/src/server/config.ts`**

```ts
const storeApiUrl = process.env.WC_STORE_API_URL;
const appOrigin = process.env.APP_ORIGIN;

if (!storeApiUrl || !appOrigin) {
  throw new Error("WC_STORE_API_URL and APP_ORIGIN must be configured");
}

const parsedStoreApiUrl = new URL(storeApiUrl);
const parsedAppOrigin = new URL(appOrigin);

if (
  parsedStoreApiUrl.protocol !== "https:" &&
  process.env.NODE_ENV === "production"
) {
  throw new Error("WC_STORE_API_URL must use HTTPS in production");
}
if (
  !["https:", "http:"].includes(parsedAppOrigin.protocol) ||
  (process.env.NODE_ENV === "production" && parsedAppOrigin.protocol !== "https:")
) {
  throw new Error("APP_ORIGIN must use HTTPS in production");
}

export const config = {
  storeApiUrl: parsedStoreApiUrl.toString().replace(/\/+$/, ""),
  appOrigin: parsedAppOrigin.origin,
};
```

`.env.local` lokacija je `apps/storefront/.env.local`:

```env
WC_STORE_API_URL=https://cms.example.com/wp-json/wc/store/v1
APP_ORIGIN=http://localhost:3000
AUTH_SECRET=replace-with-at-least-32-random-bytes
WP_HEADLESS_AUTH_URL=https://cms.example.com/wp-json/headless/v1/login
```

`AUTH_SECRET` se koristi u primjeru JWT sessiona ispod. Generiraj vrijednost lokalno pomoću `openssl rand -base64 32`; produkcijsku vrijednost dostavi kroz secret manager, nikad je ne commitaj.

**Datoteka: `apps/storefront/tsconfig.json` — relevantan dio**

Zadrži postojeći `extends` prema `packages/configuration` i spoji ove path alias-e s postojećim compiler options:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

**Datoteka: `packages/commerce/package.json` — relevantan dio**

```json
{
  "name": "@repo/commerce",
  "private": true,
  "exports": {
    ".": "./src/index.ts"
  }
}
```

Package name koristi i u aplikacijskim importima. Ako monorepo builda `dist/`, izvezi `dist/index.js` i odgovarajuće type declarations umjesto `.ts` izvora.

**Datoteka: `apps/storefront/next.config.ts`**

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  transpilePackages: ["@repo/commerce", "@repo/ui"],
};

export default nextConfig;
```

Ako `packages/ui` ne koristi Next.js/React source transpilation ili već ima ispravno buildani output, prilagodi `transpilePackages` postojećem build procesu.

**Datoteka: `packages/commerce/src/store-api/types.ts`**

```ts
export type CartItem = {
  key: string;
  id: number;
  name: string;
  quantity: number;
};

export type Cart = {
  items: CartItem[];
  items_count: number;
  totals: {
    total_price: string;
    currency_code: string;
    currency_minor_unit: number;
  };
  payment_methods?: string[];
  shipping_rates?: Array<{
    package_id: number;
    shipping_rates: Array<{
      rate_id: string;
      name: string;
      selected: boolean;
    }>;
  }>;
};

export type CartVariation = {
  attribute: string;
  value: string;
};

export type AddCartItemInput = {
  id: number;
  quantity: number;
  variation?: CartVariation[];
};

export type Address = {
  first_name: string;
  last_name: string;
  address_1: string;
  address_2?: string;
  city: string;
  state?: string;
  postcode: string;
  country: string;
  email?: string;
  phone?: string;
};

export type CheckoutInput = {
  billing_address: Address;
  shipping_address: Address;
  payment_method: string;
  payment_data?: Array<{ key: string; value: string | number | boolean }>;
  customer_note?: string;
  expected_total?: string;
};

export type Checkout = {
  order_id: number;
  status: string;
  payment_method: string;
  payment_result: {
    payment_status: string;
    redirect_url: string;
  } | null;
};
```

Ovi su namjerno minimalni tipovi za polja koja UI/BFF koristi; proširi ih kada implementiraš dodatna Store API polja. Runtime validacija ispod ostaje obavezna jer TypeScript tipovi ne validiraju HTTP JSON.

### 14.2. WooCommerce Store API client

**Datoteka: `packages/commerce/src/store-api/client.ts`**

```ts
export type StoreApiResult<T> = {
  data: T;
  cartToken?: string;
};

type StoreApiErrorBody = {
  code?: string;
  message?: string;
};

export class StoreApiError extends Error {
  constructor(
    message: string,
    readonly status: number,
    readonly code?: string,
    readonly responseBody?: unknown,
    readonly cartToken?: string
  ) {
    super(message);
    this.name = "StoreApiError";
  }
}

type StoreApiRequest = {
  baseUrl: string;
  path: string;
  method?: "GET" | "POST" | "PUT";
  cartToken?: string;
  body?: unknown;
};

export async function storeApiRequest<T>({
  baseUrl,
  path,
  method = "GET",
  cartToken,
  body,
}: StoreApiRequest): Promise<StoreApiResult<T>> {
  const normalizedBase = `${baseUrl.replace(/\/+$/, "")}/`;
  const normalizedPath = path.replace(/^\/+/, "");
  const url = new URL(normalizedPath, normalizedBase);

  if (url.origin !== new URL(normalizedBase).origin) {
    throw new Error("Store API path escaped configured origin");
  }

  const headers = new Headers({ Accept: "application/json" });
  if (cartToken) headers.set("Cart-Token", cartToken);
  if (body !== undefined) headers.set("Content-Type", "application/json");

  const response = await fetch(url, {
    method,
    headers,
    body: body === undefined ? undefined : JSON.stringify(body),
    cache: "no-store",
    signal: AbortSignal.timeout(10_000),
  });

  const responseCartToken = response.headers.get("Cart-Token") ?? undefined;
  const raw = await response.text();
  let data: unknown;
  try {
    data = raw ? JSON.parse(raw) : null;
  } catch {
    throw new StoreApiError(
      "WooCommerce returned invalid JSON",
      502,
      undefined,
      undefined,
      responseCartToken
    );
  }

  if (!response.ok) {
    const record =
      typeof data === "object" && data !== null && !Array.isArray(data)
        ? (data as Record<string, unknown>)
        : undefined;
    const errorBody: StoreApiErrorBody = {
      ...(typeof record?.code === "string" ? { code: record.code } : {}),
      ...(typeof record?.message === "string"
        ? { message: record.message }
        : {}),
    };
    throw new StoreApiError(
      errorBody?.message ?? "WooCommerce Store API request failed",
      response.status,
      errorBody?.code,
      data,
      responseCartToken
    );
  }

  return { data: data as T, cartToken: responseCartToken };
}
```

Ovdje je cast ograničen na granicu nepouzdanog vanjskog JSON-a. U produkciji dodaj runtime response schema validaciju, npr. Zod, prije nego podatke izložiš UI-ju. Nemoj logirati cijeli `StoreApiError` body.

**Datoteka: `packages/commerce/src/store-api/cart.ts`**

```ts
import { storeApiRequest } from "./client";
import type { AddCartItemInput, Cart } from "./types";

export function getCart(baseUrl: string, cartToken?: string) {
  return storeApiRequest<Cart>({
    baseUrl,
    path: "cart",
    cartToken,
  });
}

export function addCartItem(
  baseUrl: string,
  input: AddCartItemInput,
  cartToken?: string
) {
  return storeApiRequest<Cart>({
    baseUrl,
    path: "cart/add-item",
    method: "POST",
    cartToken,
    body: input,
  });
}

export function updateCartItem(
  baseUrl: string,
  input: { key: string; quantity: number },
  cartToken?: string
) {
  return storeApiRequest<Cart>({
    baseUrl,
    path: "cart/update-item",
    method: "POST",
    cartToken,
    body: input,
  });
}

export function removeCartItem(
  baseUrl: string,
  key: string,
  cartToken?: string
) {
  return storeApiRequest<Cart>({
    baseUrl,
    path: "cart/remove-item",
    method: "POST",
    cartToken,
    body: { key },
  });
}

export function updateCustomer(
  baseUrl: string,
  input: { billing_address?: unknown; shipping_address?: unknown },
  cartToken?: string
) {
  return storeApiRequest<Cart>({
    baseUrl,
    path: "cart/update-customer",
    method: "POST",
    cartToken,
    body: input,
  });
}

export function selectShippingRate(
  baseUrl: string,
  input: { packageId: number; rateId: string },
  cartToken?: string
) {
  const query = new URLSearchParams({
    package_id: String(input.packageId),
    rate_id: input.rateId,
  });

  return storeApiRequest<Cart>({
    baseUrl,
    path: `cart/select-shipping-rate?${query}`,
    method: "POST",
    cartToken,
  });
}
```

**Datoteka: `packages/commerce/src/store-api/checkout.ts`**

```ts
import { storeApiRequest } from "./client";
import type { Checkout, CheckoutInput } from "./types";

export function getCheckout(baseUrl: string, cartToken?: string) {
  return storeApiRequest<Checkout>({
    baseUrl,
    path: "checkout",
    cartToken,
  });
}

export function submitCheckout(
  baseUrl: string,
  input: CheckoutInput,
  cartToken?: string
) {
  return storeApiRequest<Checkout>({
    baseUrl,
    path: "checkout",
    method: "POST",
    cartToken,
    body: input,
  });
}

export function updateCheckout(
  baseUrl: string,
  input: {
    payment_method?: string;
    order_notes?: string;
    additional_fields?: Record<string, unknown>;
  },
  cartToken?: string
) {
  return storeApiRequest<Checkout>({
    baseUrl,
    path: "checkout",
    method: "PUT",
    cartToken,
    body: input,
  });
}
```

**Datoteka: `packages/commerce/src/index.ts`**

```ts
export * from "./store-api/cart";
export * from "./store-api/checkout";
export * from "./store-api/client";
export type {
  AddCartItemInput,
  Address,
  Cart,
  CartItem,
  CartVariation,
  Checkout,
  CheckoutInput,
} from "./store-api/types";
```

Postavi package export u `packages/commerce/package.json` tako da Next.js aplikacija uvozi javni API paketa, primjerice `"exports": { ".": "./src/index.ts" }` u monorepu koji transpajlira workspace TypeScript. Ako package gradi `dist/`, exportiraj build output umjesto `src/`.

### 14.3. BFF pomoćne funkcije: cookie, JSON, Origin i odgovori

Pretpostavka primjera: Next.js `apps/storefront/app` koristi alias `@/*` za `apps/storefront/src/*`, a commerce package ime je `@repo/commerce`.

**Datoteka: `apps/storefront/src/server/cart-cookie.ts`**

```ts
import { cookies } from "next/headers";

export const CART_COOKIE = "cart_token";

const cartCookieOptions = {
  httpOnly: true,
  secure: process.env.NODE_ENV === "production",
  sameSite: "lax" as const,
  path: "/",
  maxAge: 60 * 60 * 24 * 2,
};

export async function readCartToken() {
  return (await cookies()).get(CART_COOKIE)?.value;
}

export async function writeCartToken(token?: string) {
  if (!token) return;
  (await cookies()).set(CART_COOKIE, token, cartCookieOptions);
}
```

`maxAge` primjeru je 48 sati kao WooCommerce session expiry default u tipičnoj konfiguraciji; provjeri stvarnu konfiguraciju i uskladi je. Ako cart-token cookie treba ukloniti, pozovi `(await cookies()).delete(CART_COOKIE)` samo u Route Handleru/Server Actionu i samo prema eksplicitnoj recovery/logout politici.

**Datoteka: `apps/storefront/src/http/api.ts`**

```ts
import { NextResponse } from "next/server";
import { config } from "@/server/config";
import { writeCartToken } from "@/server/cart-cookie";
import { StoreApiError } from "@repo/commerce";

export function noStoreJson(data: unknown, status = 200) {
  return NextResponse.json(data, {
    status,
    headers: { "Cache-Control": "private, no-store" },
  });
}

export async function readJsonObject(
  request: Request
): Promise<Record<string, unknown> | null> {
  const contentLength = Number(request.headers.get("content-length") ?? 0);
  if (contentLength > 32_000) return null;

  let text: string;
  try {
    text = await request.text();
  } catch {
    return null;
  }
  if (text.length > 32_000) return null;

  try {
    const value: unknown = JSON.parse(text);
    return typeof value === "object" && value !== null && !Array.isArray(value)
      ? (value as Record<string, unknown>)
      : null;
  } catch {
    return null;
  }
}

export function requireSameOrigin(request: Request) {
  const origin = request.headers.get("origin");
  return origin !== null && origin === config.appOrigin;
}

export async function downstreamError(error: unknown) {
  if (error instanceof StoreApiError) {
    await writeCartToken(error.cartToken);
    if (error.status === 409) {
      const body = error.responseBody;
      const bodyData =
        typeof body === "object" && body !== null
          ? (body as Record<string, unknown>).data
          : undefined;
      const cart =
        typeof bodyData === "object" && bodyData !== null
          ? (bodyData as Record<string, unknown>).cart
          : undefined;
      return noStoreJson(
        {
          error: "conflict",
          ...(typeof cart === "object" && cart !== null ? { cart } : {}),
        },
        409
      );
    }

    const status =
      error.status >= 400 && error.status < 500
          ? error.status
          : 502;
    return noStoreJson(
      { error: "commerce_error" },
      status
    );
  }

  const requestId = crypto.randomUUID();
  console.error("BFF request failed", {
    requestId,
    errorName: error instanceof Error ? error.name : "UnknownError",
  });
  return noStoreJson({ error: "internal_error", requestId }, 500);
}
```

Ograničenje tijela provjerava i `Content-Length` i stvarnu duljinu teksta, jer header može nedostajati. Origin guard je za mutacijske browser rute; redirect callback iz OIDC providera treba validirati preko `state`/`nonce`, ne preko istog same-origin pravila.

**Datoteka: `apps/storefront/src/server/commerce.ts`**

```ts
import {
  addCartItem,
  getCart,
  getCheckout,
  removeCartItem,
  selectShippingRate,
  submitCheckout,
  updateCheckout,
  updateCartItem,
  updateCustomer,
} from "@repo/commerce";
import { config } from "./config";

export const commerce = {
  getCart: (token?: string) => getCart(config.storeApiUrl, token),
  addCartItem: (input: Parameters<typeof addCartItem>[1], token?: string) =>
    addCartItem(config.storeApiUrl, input, token),
  updateCartItem: (
    input: Parameters<typeof updateCartItem>[1],
    token?: string
  ) => updateCartItem(config.storeApiUrl, input, token),
  removeCartItem: (key: string, token?: string) =>
    removeCartItem(config.storeApiUrl, key, token),
  updateCustomer: (
    input: Parameters<typeof updateCustomer>[1],
    token?: string
  ) => updateCustomer(config.storeApiUrl, input, token),
  selectShippingRate: (
    input: Parameters<typeof selectShippingRate>[1],
    token?: string
  ) => selectShippingRate(config.storeApiUrl, input, token),
  getCheckout: (token?: string) => getCheckout(config.storeApiUrl, token),
  submitCheckout: (
    input: Parameters<typeof submitCheckout>[1],
    token?: string
  ) => submitCheckout(config.storeApiUrl, input, token),
  updateCheckout: (
    input: Parameters<typeof updateCheckout>[1],
    token?: string
  ) => updateCheckout(config.storeApiUrl, input, token),
};
```

**Datoteka: `apps/storefront/src/server/cart-session.ts`**

POST Store API cart operacije trebaju Cart-Token/Nonce. Zato BFF mora inicijalizirati sesiju i za slučaj kada kupac prvi put klikne "Dodaj u košaricu" bez prethodnog otvaranja `/api/cart`.

```ts
import { commerce } from "./commerce";
import { readCartToken, writeCartToken } from "./cart-cookie";

export async function ensureCartToken(): Promise<string> {
  const existing = await readCartToken();
  if (existing) return existing;

  const initialized = await commerce.getCart();
  if (!initialized.cartToken) {
    throw new Error("WooCommerce did not initialize a cart token");
  }
  await writeCartToken(initialized.cartToken);
  return initialized.cartToken;
}
```

### 14.4. Cart GET i dodavanje stavke

**Datoteka: `apps/storefront/app/api/cart/route.ts`**

```ts
import { commerce } from "@/server/commerce";
import { readCartToken, writeCartToken } from "@/server/cart-cookie";
import { downstreamError, noStoreJson } from "@/http/api";

export const dynamic = "force-dynamic";

export async function GET() {
  try {
    const result = await commerce.getCart(await readCartToken());
    await writeCartToken(result.cartToken);
    return noStoreJson(result.data);
  } catch (error) {
    return downstreamError(error);
  }
}
```

**Datoteka: `apps/storefront/app/api/cart/items/route.ts`**

```ts
import { commerce } from "@/server/commerce";
import { writeCartToken } from "@/server/cart-cookie";
import { ensureCartToken } from "@/server/cart-session";
import {
  downstreamError,
  noStoreJson,
  readJsonObject,
  requireSameOrigin,
} from "@/http/api";
import type { CartVariation } from "@repo/commerce";

function isVariation(value: unknown): value is CartVariation[] {
  return (
    Array.isArray(value) &&
    value.every(
      (item) =>
        typeof item === "object" &&
        item !== null &&
        typeof item.attribute === "string" &&
        item.attribute.length <= 128 &&
        typeof item.value === "string" &&
        item.value.length <= 200
    ) &&
    value.length <= 20
  );
}

export async function POST(request: Request) {
  if (!requireSameOrigin(request)) return noStoreJson({ error: "forbidden" }, 403);

  const body = await readJsonObject(request);
  const id = body?.id;
  const quantity = body?.quantity ?? 1;
  const variation = body?.variation;

  if (
    typeof id !== "number" ||
    !Number.isInteger(id) ||
    id <= 0 ||
    typeof quantity !== "number" ||
    !Number.isInteger(quantity) ||
    quantity <= 0 ||
    (variation !== undefined && !isVariation(variation))
  ) {
    return noStoreJson({ error: "invalid_cart_item" }, 400);
  }

  try {
    const result = await commerce.addCartItem(
      {
        id,
        quantity,
        ...(variation ? { variation } : {}),
      },
      await ensureCartToken()
    );
    await writeCartToken(result.cartToken);
    return noStoreJson(result.data);
  } catch (error) {
    return downstreamError(error);
  }
}
```

Validacija runtimea ostaje obavezna jer `Request.json()` vraća nepouzdan sadržaj. Za produkciju proširi schema validatorom maksimalne string/array opsege i pravila varijacija prema katalogu.

### 14.5. Količina i uklanjanje stavke

**Datoteka: `apps/storefront/app/api/cart/items/[key]/route.ts`**

```ts
import { commerce } from "@/server/commerce";
import { writeCartToken } from "@/server/cart-cookie";
import { ensureCartToken } from "@/server/cart-session";
import {
  downstreamError,
  noStoreJson,
  readJsonObject,
  requireSameOrigin,
} from "@/http/api";

type RouteContext = { params: Promise<{ key: string }> };

export async function PATCH(request: Request, context: RouteContext) {
  if (!requireSameOrigin(request)) return noStoreJson({ error: "forbidden" }, 403);

  const body = await readJsonObject(request);
  const quantity = body?.quantity;
  const { key } = await context.params;

  if (
    !key ||
    key.length > 128 ||
    typeof quantity !== "number" ||
    !Number.isInteger(quantity) ||
    quantity <= 0
  ) {
    return noStoreJson({ error: "invalid_cart_item" }, 400);
  }

  try {
    const result = await commerce.updateCartItem(
      { key, quantity },
      await ensureCartToken()
    );
    await writeCartToken(result.cartToken);
    return noStoreJson(result.data);
  } catch (error) {
    return downstreamError(error);
  }
}

export async function DELETE(request: Request, context: RouteContext) {
  if (!requireSameOrigin(request)) return noStoreJson({ error: "forbidden" }, 403);

  const { key } = await context.params;
  if (!key || key.length > 128) {
    return noStoreJson({ error: "invalid_cart_item" }, 400);
  }

  try {
    const result = await commerce.removeCartItem(key, await ensureCartToken());
    await writeCartToken(result.cartToken);
    return noStoreJson(result.data);
  } catch (error) {
    return downstreamError(error);
  }
}
```

U Next.js aktualnog App Routera `params` je Promise; ako projekt koristi stariju Next.js verziju, provjeri njezin route typing i prilagodi potpis.

### 14.6. Ažuriranje adrese i odabir shipping ratea

**Datoteka: `apps/storefront/app/api/cart/shipping-rates/route.ts`**

`GET` vraća shipping opcije iz cart stanja. `POST` u ovom BFF ugovoru ažurira dostavnu adresu i osvježava cart; klijent odabire rate u zasebnoj ruti.

```ts
import { commerce } from "@/server/commerce";
import { readCartToken, writeCartToken } from "@/server/cart-cookie";
import { ensureCartToken } from "@/server/cart-session";
import {
  downstreamError,
  noStoreJson,
  readJsonObject,
  requireSameOrigin,
} from "@/http/api";

export async function GET() {
  try {
    const result = await commerce.getCart(await readCartToken());
    await writeCartToken(result.cartToken);
    return noStoreJson({ shipping_rates: result.data.shipping_rates ?? [] });
  } catch (error) {
    return downstreamError(error);
  }
}

export async function POST(request: Request) {
  if (!requireSameOrigin(request)) return noStoreJson({ error: "forbidden" }, 403);
  const body = await readJsonObject(request);
  const rawAddress = body?.shipping_address;

  if (
    typeof rawAddress !== "object" ||
    rawAddress === null ||
    Array.isArray(rawAddress)
  ) {
    return noStoreJson({ error: "invalid_shipping_address" }, 400);
  }

  const source = rawAddress as Record<string, unknown>;
  const fields = [
    "first_name",
    "last_name",
    "address_1",
    "address_2",
    "city",
    "state",
    "postcode",
    "country",
  ] as const;
  const shippingAddress: Record<string, string> = {};
  for (const field of fields) {
    const value = source[field];
    if (value === undefined) continue;
    if (typeof value !== "string" || value.length > 200) {
      return noStoreJson({ error: "invalid_shipping_address" }, 400);
    }
    shippingAddress[field] = value;
  }
  if (!shippingAddress.country || !shippingAddress.postcode) {
    return noStoreJson({ error: "invalid_shipping_address" }, 400);
  }

  try {
    const result = await commerce.updateCustomer(
      { shipping_address: shippingAddress },
      await ensureCartToken()
    );
    await writeCartToken(result.cartToken);
    return noStoreJson(result.data);
  } catch (error) {
    return downstreamError(error);
  }
}
```

**Datoteka: `apps/storefront/app/api/cart/shipping-rates/select/route.ts`**

```ts
import { commerce } from "@/server/commerce";
import { writeCartToken } from "@/server/cart-cookie";
import { ensureCartToken } from "@/server/cart-session";
import {
  downstreamError,
  noStoreJson,
  readJsonObject,
  requireSameOrigin,
} from "@/http/api";

export async function POST(request: Request) {
  if (!requireSameOrigin(request)) return noStoreJson({ error: "forbidden" }, 403);
  const body = await readJsonObject(request);

  if (
    typeof body?.packageId !== "number" ||
    !Number.isInteger(body.packageId) ||
    body.packageId < 0 ||
    typeof body.rateId !== "string" ||
    body.rateId.length === 0 ||
    body.rateId.length > 200
  ) {
    return noStoreJson({ error: "invalid_shipping_rate" }, 400);
  }

  try {
    const result = await commerce.selectShippingRate(
      { packageId: body.packageId, rateId: body.rateId },
      await ensureCartToken()
    );
    await writeCartToken(result.cartToken);
    return noStoreJson(result.data);
  } catch (error) {
    return downstreamError(error);
  }
}
```

WooCommerce prihvaća `package_id` i `rate_id` za `/cart/select-shipping-rate`. Odabir uvijek mora biti jedan od rateova koje je Store API vratio za aktualni cart; WooCommerce je konačni autoritet i može odbiti zastarjeli rate.

### 14.7. Kuponi

**Datoteka: `packages/commerce/src/store-api/coupons.ts`**

```ts
import { storeApiRequest } from "./client";
import type { Cart } from "./types";

export function applyCoupon(
  baseUrl: string,
  code: string,
  cartToken?: string
) {
  return storeApiRequest<Cart>({
    baseUrl,
    path: "cart/apply-coupon",
    method: "POST",
    cartToken,
    body: { code },
  });
}

export function removeCoupon(
  baseUrl: string,
  code: string,
  cartToken?: string
) {
  return storeApiRequest<Cart>({
    baseUrl,
    path: "cart/remove-coupon",
    method: "POST",
    cartToken,
    body: { code },
  });
}
```

Dodaj `export * from "./store-api/coupons";` u `packages/commerce/src/index.ts`.

**Datoteka: `apps/storefront/app/api/cart/coupons/route.ts`**

```ts
import { applyCoupon, removeCoupon } from "@repo/commerce";
import { config } from "@/server/config";
import { writeCartToken } from "@/server/cart-cookie";
import { ensureCartToken } from "@/server/cart-session";
import {
  downstreamError,
  noStoreJson,
  readJsonObject,
  requireSameOrigin,
} from "@/http/api";

async function mutateCoupon(request: Request, remove: boolean) {
  if (!requireSameOrigin(request)) return noStoreJson({ error: "forbidden" }, 403);
  const body = await readJsonObject(request);
  const code = body?.code;
  if (typeof code !== "string" || code.trim().length === 0 || code.length > 128) {
    return noStoreJson({ error: "invalid_coupon" }, 400);
  }

  try {
    const operation = remove ? removeCoupon : applyCoupon;
    const result = await operation(
      config.storeApiUrl,
      code.trim(),
      await ensureCartToken()
    );
    await writeCartToken(result.cartToken);
    return noStoreJson(result.data);
  } catch (error) {
    return downstreamError(error);
  }
}

export async function POST(request: Request) {
  return mutateCoupon(request, false);
}

export async function DELETE(request: Request) {
  return mutateCoupon(request, true);
}
```

### 14.8. UI koji koristi isključivo BFF

**Datoteka: `packages/ui/src/components/cart/Cart.tsx`**

```tsx
"use client";

import { useEffect, useState } from "react";

type CartView = {
  items: Array<{ key: string; name: string; quantity: number }>;
  items_count: number;
  totals: {
    total_price: string;
    currency_code: string;
    currency_minor_unit: number;
  };
};

async function readCart(): Promise<CartView> {
  const response = await fetch("/api/cart", { cache: "no-store" });
  if (!response.ok) throw new Error("Košaricu nije moguće učitati.");
  return response.json() as Promise<CartView>;
}

export function Cart() {
  const [cart, setCart] = useState<CartView | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    void readCart()
      .then(setCart)
      .catch(() => setError("Košaricu nije moguće učitati."));
  }, []);

  async function removeItem(key: string) {
    setError(null);
    const response = await fetch(
      `/api/cart/items/${encodeURIComponent(key)}`,
      { method: "DELETE" }
    );
    if (!response.ok) {
      const failure: unknown = await response.json();
      if (
        response.status === 409 &&
        typeof failure === "object" &&
        failure !== null &&
        "cart" in failure
      ) {
        setCart((failure as { cart: CartView }).cart);
      }
      setError("Stavku nije moguće ukloniti.");
      return;
    }
    setCart((await response.json()) as CartView);
  }

  if (error && !cart) return <p role="alert">{error}</p>;
  if (!cart) return <p>Učitavanje košarice…</p>;
  if (cart.items.length === 0) return <p>Košarica je prazna.</p>;

  const formattedTotal = formatMinorUnits(
    cart.totals.total_price,
    cart.totals.currency_code,
    cart.totals.currency_minor_unit
  );

  return (
    <section>
      <h1>Košarica</h1>
      {error && <p role="alert">{error}</p>}
      <ul>
        {cart.items.map((item) => (
          <li key={item.key}>
            {item.name} × {item.quantity}
            <button type="button" onClick={() => void removeItem(item.key)}>
              Ukloni
            </button>
          </li>
        ))}
      </ul>
      <p>Broj komada: {cart.items_count}</p>
      <p>Ukupno: {formattedTotal}</p>
    </section>
  );
}

function formatMinorUnits(value: string, currency: string, minorUnit: number) {
  if (
    !/^-?\d+$/.test(value) ||
    !Number.isInteger(minorUnit) ||
    minorUnit < 0 ||
    minorUnit > 3
  ) {
    throw new Error("Invalid currency amount from cart API");
  }
  const amount = BigInt(value);
  const negative = amount < 0n;
  const absolute = negative ? -amount : amount;
  const scale = 10n ** BigInt(minorUnit);
  const whole = absolute / scale;
  const fraction = (absolute % scale).toString().padStart(minorUnit, "0");
  const formatter = new Intl.NumberFormat(undefined, {
    style: "currency",
    currency,
    minimumFractionDigits: minorUnit,
    maximumFractionDigits: minorUnit,
  });
  const wholeForPattern = negative && whole === 0n ? -1n : negative ? -whole : whole;
  const parts = formatter.formatToParts(wholeForPattern);
  const digitFormatter = new Intl.NumberFormat(undefined, { useGrouping: false });
  const localizedFraction = [...fraction]
    .map((digit) => digitFormatter.format(Number(digit)))
    .join("");

  return parts
    .map((part) => {
      if (part.type === "integer" && negative && whole === 0n) return "0";
      if (part.type === "fraction") return localizedFraction;
      return part.value;
    })
    .join("");
}
```

Formatter koristi `BigInt` za minor-unit vrijednosti i ne radi floating-point aritmetiku. Komponenta ne prima i ne čita Cart-Token.

### 14.9. Aplikacijska sesija i WordPress credential adapter

Sljedeći session primjer koristi potpisani JWT radi samostalnog rada bez dodatnog session-store packagea. Zbog stateless dizajna logout ne opoziva već izdani token prije isteka; za trenutačni revoke koristi server-side session store i zamijeni ovaj primjer. `jose` dodaj kao dependency u `apps/storefront`.

**Datoteka: `apps/storefront/src/auth/session.ts`**

```ts
import { SignJWT, jwtVerify } from "jose";
import { cookies } from "next/headers";

const COOKIE_NAME = "app_session";
const SESSION_SECONDS = 60 * 60 * 24 * 7;
const rawSecret = process.env.AUTH_SECRET;

if (!rawSecret || new TextEncoder().encode(rawSecret).length < 32) {
  throw new Error("AUTH_SECRET must contain at least 32 bytes");
}

const secret = new TextEncoder().encode(rawSecret);

export type AppIdentity = {
  subject: string;
  email: string;
  displayName: string;
};

export async function createSession(identity: AppIdentity) {
  const token = await new SignJWT({
    email: identity.email,
    name: identity.displayName,
  })
    .setProtectedHeader({ alg: "HS256" })
    .setSubject(identity.subject)
    .setIssuer("storefront")
    .setAudience("storefront-web")
    .setIssuedAt()
    .setExpirationTime(`${SESSION_SECONDS}s`)
    .sign(secret);

  (await cookies()).set(COOKIE_NAME, token, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
    path: "/",
    maxAge: SESSION_SECONDS,
  });
}

export async function getSession(): Promise<AppIdentity | null> {
  const token = (await cookies()).get(COOKIE_NAME)?.value;
  if (!token) return null;

  try {
    const { payload } = await jwtVerify(token, secret, {
      algorithms: ["HS256"],
      issuer: "storefront",
      audience: "storefront-web",
    });
    if (
      typeof payload.sub !== "string" ||
      typeof payload.email !== "string" ||
      typeof payload.name !== "string"
    ) {
      return null;
    }
    return {
      subject: payload.sub,
      email: payload.email,
      displayName: payload.name,
    };
  } catch {
    return null;
  }
}

export async function destroySession() {
  (await cookies()).delete(COOKIE_NAME);
}
```

Payload potpisanog JWT-a nije šifriran; u cookie stavi samo minimalne, neosjetljive claimove. Ne stavljaj password, provider access/refresh tokene ili payment podatke u JWT.

**Primjer WordPress endpointa — putanja u WordPress repozitoriju: `wp-content/mu-plugins/headless-auth.php`**

Ovaj mali endpoint je konkretna credential-based opcija, ne WordPress core login API. Stavi ga samo na WordPress instancu kojoj vjeruješ, koristi HTTPS i rate limitiraj `/wp-json/headless/v1/login` na reverse proxyju/WAF: primjer koda namjerno ne implementira distribuirani rate limiter.

```php
<?php
/**
 * Plugin Name: Headless Storefront Authentication
 */

add_action('rest_api_init', function () {
    register_rest_route('headless/v1', '/login', [
        'methods'             => 'POST',
        'permission_callback' => '__return_true',
        'callback'            => function (WP_REST_Request $request) {
            $params = $request->get_json_params();
            $email_param = is_array($params) ? ($params['email'] ?? null) : null;
            $password_param = is_array($params) ? ($params['password'] ?? null) : null;

            if (
                !is_string($email_param) ||
                !is_string($password_param)
            ) {
                return new WP_Error(
                    'invalid_credentials',
                    'Invalid email or password.',
                    ['status' => 401]
                );
            }

            $email = sanitize_email($email_param);
            if (
                !$email ||
                !is_email($email) ||
                $password_param === '' ||
                strlen($password_param) > 1024
            ) {
                return new WP_Error(
                    'invalid_credentials',
                    'Invalid email or password.',
                    ['status' => 401]
                );
            }

            $user = wp_authenticate($email, $password_param);
            if (is_wp_error($user)) {
                return new WP_Error(
                    'invalid_credentials',
                    'Invalid email or password.',
                    ['status' => 401]
                );
            }

            $allowed_roles = apply_filters(
                'headless_storefront_login_allowed_roles',
                ['customer', 'subscriber']
            );
            if (!array_intersect($allowed_roles, (array) $user->roles)) {
                return new WP_Error(
                    'invalid_credentials',
                    'Invalid email or password.',
                    ['status' => 401]
                );
            }

            return new WP_REST_Response([
                'id'           => (string) $user->ID,
                'email'        => (string) $user->user_email,
                'display_name' => (string) $user->display_name,
            ], 200);
        },
    ]);
});

add_filter('rest_pre_serve_request', function ($served, $result, $request) {
    if ($request->get_route() === '/headless/v1/login') {
        header('Cache-Control: private, no-store');
        header('Pragma: no-cache');
    }
    return $served;
}, 10, 3);
```

Ovaj endpoint ne stvara WordPress browser cookie niti WooCommerce customer cart sesiju; samo provjerava lozinku i vraća identitet za Next.js session. Zadani allowlist prihvaća role `customer` i `subscriber`; promijeni ga kroz filter samo nakon odluke koji računi smiju u storefront. Ako WordPress konfiguracija ne dopušta prijavu emailom kroz `wp_authenticate`, prilagodi login funkciju i testiraj je na stvarnoj WP verziji. Nemoj slati password kroz URL ili log.

**Datoteka: `apps/storefront/src/auth/wordpress.ts`**

```ts
import type { AppIdentity } from "./session";

const endpoint = process.env.WP_HEADLESS_AUTH_URL;
if (!endpoint || new URL(endpoint).protocol !== "https:") {
  throw new Error("WP_HEADLESS_AUTH_URL must be configured with HTTPS");
}

export async function authenticateWithWordPress(
  email: string,
  password: string
): Promise<AppIdentity | null> {
  const response = await fetch(endpoint, {
    method: "POST",
    headers: { "Content-Type": "application/json", Accept: "application/json" },
    body: JSON.stringify({ email, password }),
    cache: "no-store",
    signal: AbortSignal.timeout(8_000),
  });

  if (response.status === 401) return null;
  if (!response.ok) throw new Error("WordPress authentication unavailable");

  const body: unknown = await response.json();
  if (
    typeof body !== "object" ||
    body === null ||
    typeof (body as Record<string, unknown>).id !== "string" ||
    typeof (body as Record<string, unknown>).email !== "string" ||
    typeof (body as Record<string, unknown>).display_name !== "string"
  ) {
    throw new Error("WordPress returned an invalid identity response");
  }

  const identity = body as {
    id: string;
    email: string;
    display_name: string;
  };
  return {
    subject: identity.id,
    email: identity.email,
    displayName: identity.display_name,
  };
}
```

**Datoteka: `apps/storefront/app/api/auth/login/route.ts`**

```ts
import { authenticateWithWordPress } from "@/auth/wordpress";
import { createSession } from "@/auth/session";
import {
  downstreamError,
  noStoreJson,
  readJsonObject,
  requireSameOrigin,
} from "@/http/api";

export async function POST(request: Request) {
  if (!requireSameOrigin(request)) return noStoreJson({ error: "forbidden" }, 403);
  const body = await readJsonObject(request);
  const email = body?.email;
  const password = body?.password;

  if (
    typeof email !== "string" ||
    email.length > 254 ||
    typeof password !== "string" ||
    password.length === 0 ||
    password.length > 1024
  ) {
    return noStoreJson({ error: "invalid_credentials" }, 400);
  }

  try {
    const identity = await authenticateWithWordPress(email.trim(), password);
    if (!identity) return noStoreJson({ error: "invalid_credentials" }, 401);
    await createSession(identity);
    return noStoreJson({
      user: {
        email: identity.email,
        displayName: identity.displayName,
      },
    });
  } catch (error) {
    return downstreamError(error);
  }
}
```

Rate limiting mora se izvršavati prije slanja login pokušaja WordPressu (gateway/WAF ili shared store middleware) i na samom javno dostupnom WordPress login endpointu. Ne ograničavaj pokušaje samo u memoriji Next.js procesa jer više instanci ne dijeli stanje.

**Datoteka: `apps/storefront/app/api/auth/me/route.ts`**

```ts
import { getSession } from "@/auth/session";
import { noStoreJson } from "@/http/api";

export async function GET() {
  const user = await getSession();
  if (!user) return noStoreJson({ user: null }, 401);
  return noStoreJson({
    user: { email: user.email, displayName: user.displayName },
  });
}
```

**Datoteka: `apps/storefront/app/api/auth/logout/route.ts`**

```ts
import { destroySession } from "@/auth/session";
import { noStoreJson, requireSameOrigin } from "@/http/api";

export async function POST(request: Request) {
  if (!requireSameOrigin(request)) return noStoreJson({ error: "forbidden" }, 403);
  await destroySession();
  return noStoreJson({ success: true });
}
```

Ako odabereš OIDC, nemoj prilagođavati `authenticateWithWordPress` da prima korisničku lozinku. Umjesto toga dodaj `GET /api/auth/login` start rutu i `GET /api/auth/callback` s provider SDK-om; obavezno validiraj `state`, `nonce`, PKCE, issuer, audience, redirect URI i token signature. Točan kod ovisi o odabranom provideru/SDK-u i mora se izraditi prema njegovoj dokumentaciji.

### 14.10. Checkout GET i BACS primjer

Primjer omogućuje checkout samo preko `bacs` ako ga WooCommerce vraća kao aktivnu metodu. To je namjerno zatvoren primjer; za Stripe/PayPal/ostale metode implementiraj gateway-specifično generiranje i validaciju `payment_data` te frontend SDK tokenizaciju.

**Datoteka: `apps/storefront/app/api/checkout/route.ts`**

```ts
import { commerce } from "@/server/commerce";
import { readCartToken, writeCartToken } from "@/server/cart-cookie";
import {
  downstreamError,
  noStoreJson,
  readJsonObject,
  requireSameOrigin,
} from "@/http/api";
import type { Address, CheckoutInput } from "@repo/commerce";

function parseAddress(value: unknown): Address | null {
  if (typeof value !== "object" || value === null || Array.isArray(value)) return null;
  const address = value as Record<string, unknown>;
  const firstName = address.first_name;
  const lastName = address.last_name;
  const address1 = address.address_1;
  const city = address.city;
  const postcode = address.postcode;
  const country = address.country;

  if (
    typeof firstName !== "string" ||
    typeof lastName !== "string" ||
    typeof address1 !== "string" ||
    typeof city !== "string" ||
    typeof postcode !== "string" ||
    typeof country !== "string" ||
    firstName.length > 100 ||
    lastName.length > 100 ||
    address1.length > 200 ||
    city.length > 100 ||
    postcode.length > 32 ||
    !/^[A-Z]{2}$/.test(country)
  ) {
    return null;
  }

  const optionalString = (key: string, maxLength: number) => {
    const item = address[key];
    if (item === undefined) return undefined;
    return typeof item === "string" && item.length <= maxLength ? item : null;
  };
  const address2 = optionalString("address_2", 200);
  const state = optionalString("state", 100);
  const email = optionalString("email", 254);
  const phone = optionalString("phone", 64);
  if ([address2, state, email, phone].includes(null)) return null;

  return {
    first_name: firstName,
    last_name: lastName,
    address_1: address1,
    city,
    postcode,
    country,
    ...(typeof address2 === "string" ? { address_2: address2 } : {}),
    ...(typeof state === "string" ? { state } : {}),
    ...(typeof email === "string" ? { email } : {}),
    ...(typeof phone === "string" ? { phone } : {}),
  };
}

export async function GET() {
  try {
    const cart = await commerce.getCart(await readCartToken());
    await writeCartToken(cart.cartToken);
    const result = await commerce.getCheckout(cart.cartToken);
    await writeCartToken(result.cartToken);
    return noStoreJson(result.data);
  } catch (error) {
    return downstreamError(error);
  }
}

export async function POST(request: Request) {
  if (!requireSameOrigin(request)) return noStoreJson({ error: "forbidden" }, 403);
  const body = await readJsonObject(request);
  const billingAddress = parseAddress(body?.billing_address);
  const shippingAddress = parseAddress(body?.shipping_address);
  const paymentMethod = body?.payment_method;
  const expectedTotal = body?.expected_total;
  const customerNote = body?.customer_note;

  if (
    !billingAddress ||
    !shippingAddress ||
    typeof billingAddress.email !== "string" ||
    !billingAddress.email.includes("@") ||
    paymentMethod !== "bacs" ||
    (expectedTotal !== undefined &&
      (typeof expectedTotal !== "string" || !/^\d+$/.test(expectedTotal))) ||
    (customerNote !== undefined &&
      (typeof customerNote !== "string" || customerNote.length > 1000))
  ) {
    return noStoreJson({ error: "invalid_checkout" }, 400);
  }

  try {
    const token = await readCartToken();
    const cart = await commerce.getCart(token);
    await writeCartToken(cart.cartToken);
    if (!cart.data.payment_methods?.includes("bacs")) {
      return noStoreJson({ error: "payment_method_unavailable" }, 400);
    }

    const input: CheckoutInput = {
      billing_address: billingAddress,
      shipping_address: shippingAddress,
      payment_method: "bacs",
      ...(typeof customerNote === "string"
        ? { customer_note: customerNote }
        : {}),
      ...(typeof expectedTotal === "string"
        ? { expected_total: expectedTotal }
        : {}),
    };
    const result = await commerce.submitCheckout(input, cart.cartToken ?? token);
    await writeCartToken(result.cartToken);
    return noStoreJson(result.data);
  } catch (error) {
    return downstreamError(error);
  }
}

export async function PUT(request: Request) {
  if (!requireSameOrigin(request)) return noStoreJson({ error: "forbidden" }, 403);
  const body = await readJsonObject(request);
  const paymentMethod = body?.payment_method;
  const orderNotes = body?.order_notes;

  if (
    (paymentMethod !== undefined && paymentMethod !== "bacs") ||
    (orderNotes !== undefined &&
      (typeof orderNotes !== "string" || orderNotes.length > 1000))
  ) {
    return noStoreJson({ error: "invalid_checkout_update" }, 400);
  }

  try {
    const token = await readCartToken();
    const cart = await commerce.getCart(token);
    await writeCartToken(cart.cartToken);
    if (
      paymentMethod === "bacs" &&
      !cart.data.payment_methods?.includes("bacs")
    ) {
      return noStoreJson({ error: "payment_method_unavailable" }, 400);
    }

    const result = await commerce.updateCheckout(
      {
        ...(paymentMethod ? { payment_method: paymentMethod } : {}),
        ...(typeof orderNotes === "string" ? { order_notes: orderNotes } : {}),
      },
      cart.cartToken ?? token
    );
    await writeCartToken(result.cartToken);
    return noStoreJson(result.data);
  } catch (error) {
    return downstreamError(error);
  }
}
```

Store API `POST /checkout` zahtijeva billing/shipping adresu i payment method. `PUT /checkout` sprema odabranu metodu/napomenu prema podržanom Store API ugovoru. Poslije potvrde checkouta tretiraj WooCommerce/gateway rezultat kao odgovor procesa; ne proglasi uplatu uspješnom samo zbog UI redirecta. Za gateway koji redirecta, validiraj order status server-side ili primitak potpisanog webhooka.

**Datoteka: `packages/ui/src/components/checkout/CheckoutForm.tsx`**

```tsx
"use client";

import { useState } from "react";

export function CheckoutForm() {
  const [message, setMessage] = useState("");
  const [pending, setPending] = useState(false);

  async function submit(formData: FormData) {
    if (pending) return;
    setPending(true);
    setMessage("");

    try {
      const address = {
        first_name: String(formData.get("first_name") ?? ""),
        last_name: String(formData.get("last_name") ?? ""),
        address_1: String(formData.get("address_1") ?? ""),
        city: String(formData.get("city") ?? ""),
        postcode: String(formData.get("postcode") ?? ""),
        country: String(formData.get("country") ?? "").toUpperCase(),
        email: String(formData.get("email") ?? ""),
      };

      const response = await fetch("/api/checkout", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          billing_address: address,
          shipping_address: address,
          payment_method: "bacs",
        }),
      });

      const result: unknown = await response.json();
      if (!response.ok) {
        setMessage("Narudžbu nije moguće poslati. Provjerite podatke i pokušajte ponovno.");
        return;
      }
      const orderId =
        typeof result === "object" && result !== null
          ? (result as Record<string, unknown>).order_id
          : undefined;
      setMessage(`Narudžba ${String(orderId ?? "")} je zaprimljena.`);
    } catch {
      setMessage("Veza s trgovinom trenutno nije dostupna.");
    } finally {
      setPending(false);
    }
  }

  return (
    <form action={submit}>
      <label>
        Ime <input name="first_name" required />
      </label>
      <label>
        Prezime <input name="last_name" required />
      </label>
      <label>
        Adresa <input name="address_1" required />
      </label>
      <label>
        Grad <input name="city" required />
      </label>
      <label>
        Poštanski broj <input name="postcode" required />
      </label>
      <label>
        Država (ISO-3166-1 alpha-2) <input name="country" minLength={2} maxLength={2} required />
      </label>
      <label>
        Email <input name="email" type="email" required />
      </label>
      <button type="submit" disabled={pending}>
        {pending ? "Slanje…" : "Naruči uz bankovni transfer"}
      </button>
      {message && <p role="status">{message}</p>}
    </form>
  );
}
```

Ovaj jednostavni UI koristi isti billing i shipping address radi preglednosti; produkcijski checkout treba odvojeno omogućiti različitu adresu dostave, validirati sva polja prema WooCommerce country/state pravilima i prije slanja prikazati aktualne shipping/tax/totals podatke iz Store API-ja. UI validacija je samo UX; BFF/WooCommerce validacija ostaje autoritativna.

### 14.11. Test jednog važnog ugovora commerce klijenta

Ako workspace već koristi Vitest, dodaj test u postojeći test target; ako ne, koristi postojeći runner umjesto da uvodiš drugi.

**Datoteka: `packages/commerce/src/store-api/client.test.ts`**

```ts
import { afterEach, describe, expect, it, vi } from "vitest";
import { storeApiRequest } from "./client";

afterEach(() => vi.unstubAllGlobals());

describe("storeApiRequest", () => {
  it("prosljeđuje Cart-Token i vraća obnovljeni token", async () => {
    const fetchMock = vi.fn().mockResolvedValue(
      new Response(JSON.stringify({ items: [], items_count: 0 }), {
        status: 200,
        headers: {
          "Content-Type": "application/json",
          "Cart-Token": "rotated-token",
        },
      })
    );
    vi.stubGlobal("fetch", fetchMock);

    const result = await storeApiRequest<{ items: unknown[] }>({
      baseUrl: "https://cms.example.com/wp-json/wc/store/v1",
      path: "cart",
      cartToken: "old-token",
    });

    const [, init] = fetchMock.mock.calls[0] as [URL, RequestInit];
    expect(new Headers(init.headers).get("Cart-Token")).toBe("old-token");
    expect(result.cartToken).toBe("rotated-token");
    expect(result.data.items).toEqual([]);
  });
});
```

### 14.12. Reverse proxy primjer

Ovaj Nginx primjer terminira TLS uz postojeće certifikate i šalje samo storefront traffic u Next.js; CMS host je server-side upstream u aplikacijskoj konfiguraciji, ne browser proxy route.

**Datoteka: `infra/proxy/nginx.conf`**

```nginx
# Uključi ovaj snippet unutar Nginx http {} konteksta.
limit_req_zone $binary_remote_addr zone=store_login:10m rate=5r/m;

server {
    listen 443 ssl http2;
    server_name shop.example.com;

    ssl_certificate     /etc/ssl/certs/shop.crt;
    ssl_certificate_key /etc/ssl/private/shop.key;

    location = /api/auth/login {
        limit_req zone=store_login burst=3 nodelay;
        limit_req_status 429;

        proxy_pass http://storefront:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        proxy_pass http://storefront:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_no_cache 1;
        proxy_cache_bypass 1;
    }
}
```

U deploymentu povjeri `X-Forwarded-*` vrijednosti samo proxyju koji kontroliraš, nemoj vjerovati proizvoljnim client-supplied forwarding headerima i provjeri da aplikacija/CDN ne cacheira `/api/cart`, `/api/checkout` ili `/api/auth`.
Ako je Nginx iza load balancera, konfiguriraj `real_ip` samo za poznate proxy IP/CIDR raspone kako rate limit ne bi koristio istu adresu za sve kupce niti vjerovao lažnom `X-Forwarded-For`.

### 14.13. Što ovisi o konkretnom provideru

Sljedeće granice ne mogu biti univerzalno implementirane bez izbora dodatne tehnologije:

1. **OIDC callback** — SDK, discovery URL, client authentication, PKCE i provider-specific claim mapping.
2. **Customer cart** — način na koji WooCommerce dobiva provjeren WordPress customer identitet; Next.js app session i Cart-Token to ne uspostavljaju sami.
3. **Kartično plaćanje** — browser SDK/tokenizacija i `payment_data` polja konkretnog gatewaya.
4. **Webhook** — signature header/algoritam, replay protection i event model ovise o payment provideru.
5. **Session revoke i shared rate limit** — zahtijevaju odabrani Redis/DB/WAF ili auth/session servis.

Za svaku od tih stavki napravi zaseban ADR u `docs/architecture/` i provider-specific contract/integration test. Nemoj zamijeniti nedostajući provider ugovor generičkim success responseom.
