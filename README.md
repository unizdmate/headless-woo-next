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
POST   /api/cart/shipping-rates    { packageId, rateId } # prema Store API inputu
GET    /api/checkout               # WooCommerce checkout draft/podaci
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
