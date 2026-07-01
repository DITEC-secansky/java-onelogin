# Plán: Autentifikácia voči Keycloaku (namiesto UPVS FIX)

Tento dokument popisuje, ako prepojiť existujúce demo **webssodemo** (SAML Service Provider
postavený na [java-saml od OneLogin](https://github.com/SAML-Toolkits/java-saml)) s
**Keycloakom** vystupujúcim ako **SAML 2.0 Identity Provider (IdP)**, namiesto produkčného
IdP UPVS FIX (`prihlasenie.upvsfix.gov.sk`).

Princíp je jednoduchý: aplikácia ostáva tým istým SAML SP, mení sa len **IdP** – teda
hodnoty `onelogin.saml2.idp.*` v properties súbore a zaregistrovanie SP v Keycloaku.

---

## 0. Východiskový stav (čo už máme)

- SP aplikácia (`ROOT.war`) beží v Tomcate cez `docker-compose.yml` na `https://127.0.0.1:3001/`.
- SAML endpointy SP (z `web.xml`):
  - **ACS** (callback): `/auth/saml/callback` → `https://127.0.0.1:3001/auth/saml/callback`
  - **SLS** (single logout): `/auth/saml/logout` → `https://127.0.0.1:3001/auth/saml/logout`
  - **SP metadáta**: `/sp-metadata.xml`
  - login: `/login`, logout: `/logout`
- Konfigurácia sa načítava cez `Settings.java` zo súboru
  `conf/two_keys_setup.webssodemo.saml.properties` (mimo classpath, načítava sa pri každom requeste).
- SP entityID: `https://127.0.0.1.slovensko.sk.login`
- **Tomcat počúva iba cez HTTPS na porte 3001** (`server.xml`: `scheme="https" secure="true"`,
  keystore `localhost-rsa.jks`). Všetky SP URL teda musia byť `https://127.0.0.1:3001/...`.
- `ProtectedFilter` chráni `/protected/*`: ak v session nie je `nameId`, spustí
  `auth.login(requestURI)` – t.j. **deep-link** sa nesie cez **RelayState** a SP sa naň po
  prihlásení vráti (logika v `SamlServlet`).
- Logout je **SP-initiated SLO** (`LogoutServlet` pošle podpísaný `<LogoutRequest>` cez
  redirect binding; `SingleLogoutServiceServlet.processSLO()` spracuje `<LogoutResponse>`
  na tom istom endpointe `/auth/saml/logout`).

> Cieľ: vytvoriť nový profil konfigurácie (napr. `keycloak.webssodemo.saml.properties`),
> ktorý v sekcii `idp.*` ukazuje na Keycloak, a v Keycloaku zaregistrovať tento SP.

---

## 1. Spustenie Keycloaku (lokálne, cez Docker)

Pridať službu `keycloak` do `docker-compose.yml` (alebo spustiť samostatne):

```yaml
  keycloak:
    image: quay.io/keycloak/keycloak:26.0
    command: start-dev
    environment:
      - KC_BOOTSTRAP_ADMIN_USERNAME=admin
      - KC_BOOTSTRAP_ADMIN_PASSWORD=admin
      - KC_HTTP_ENABLED=true
      - KC_HOSTNAME_STRICT=false
    ports:
      - 8081:8080
```

- Admin konzola: `http://localhost:8081/` (admin / admin).
- Pre lokálny vývoj stačí HTTP. Pre HTTPS treba doplniť `KC_HTTPS_*` a certifikáty.

> Pozor na dostupnosť hostname medzi kontajnerom Tomcatu a Keycloakom.
> Tomcat (java-saml) komunikuje s Keycloakom iba nepriamo cez prehliadač používateľa
> (redirect/POST binding), takže pre SSO stačí, aby Keycloak URL boli dostupné **z prehliadača**.
> Validáciu podpisu robí SP lokálne pomocou IdP certifikátu – nepotrebuje sieťové spojenie na Keycloak.

> **HTTP vs HTTPS – dôležité kvôli cookies.** SP beží na **HTTPS** (`https://127.0.0.1:3001`).
> Pri POST binding Keycloak odošle `<Response>` na ACS ako **cross-site top-level POST**.
> Ak je Keycloak na `http://`, môže to spôsobiť problémy so `SameSite`/`Secure` cookies
> (strata session, prípadne varovania o mixed-content pri redirectoch). Preto pre stabilný
> beh **odporúčam Keycloak tiež cez HTTPS**, alebo aspoň počítať s týmto rizikom pri ladení.
> Pinovanie hostname/portu cez `KC_HOSTNAME` zabráni nekonzistentným entityID/endpointom.

### 1.1 Keycloak ako samostatný, predkonfigurovaný image (odporúčané)

Áno – Keycloak je už teraz **samostatný image/kontajner** (vlastná služba v compose, žiadna
väzba na `ROOT.war`). Manuálne klikanie z kapitoly 2 sa však dá úplne vynechať tak, že realm
(vrátane SAML klienta, mapperov a test používateľa) **napečieme do imagu / naimportujeme pri štarte**.
Tým je celé prostredie reprodukovateľné a verzionované v gite.

**Možnosť A – import realmu z JSON cez volume (bez vlastného Dockerfile):**

```yaml
  keycloak:
    image: quay.io/keycloak/keycloak:26.0
    command: start-dev --import-realm
    environment:
      - KC_BOOTSTRAP_ADMIN_USERNAME=admin
      - KC_BOOTSTRAP_ADMIN_PASSWORD=admin
      - KC_HTTP_ENABLED=true
      - KC_HOSTNAME_STRICT=false
    volumes:
      - ./keycloak/realm-export.json:/opt/keycloak/data/import/realm-webssodemo.json
    ports:
      - 8081:8080
```

- `--import-realm` pri štarte naimportuje všetky realmy z `/opt/keycloak/data/import/`.
- `realm-export.json` získame z bežiaceho Keycloaku: **Realm settings → Action → Partial export**
  (zaškrtnúť *Include clients* a *Include groups/roles*), alebo cez
  `kc.sh export --realm webssodemo --file ...`. Súbor uložíme do `keycloak/` v repe.

**Možnosť B – vlastný (self-contained) image s realmom „zapečeným" dovnútra:**

```dockerfile
# keycloak/Dockerfile
FROM quay.io/keycloak/keycloak:26.0 AS builder
ENV KC_HEALTH_ENABLED=true
RUN /opt/keycloak/bin/kc.sh build

FROM quay.io/keycloak/keycloak:26.0
COPY --from=builder /opt/keycloak/ /opt/keycloak/
COPY realm-export.json /opt/keycloak/data/import/realm-webssodemo.json
ENTRYPOINT ["/opt/keycloak/bin/kc.sh", "start-dev", "--import-realm"]
```

```yaml
  keycloak:
    build: ./keycloak
    environment:
      - KC_BOOTSTRAP_ADMIN_USERNAME=admin
      - KC_BOOTSTRAP_ADMIN_PASSWORD=admin
      - KC_HTTP_ENABLED=true
      - KC_HOSTNAME_STRICT=false
    ports:
      - 8081:8080
```

> **Čo musí byť v `realm-export.json`:** realm `webssodemo`, SAML klient s `clientId =
> sp.entityid`, ACS/SLO URL, zapnuté podpisy/šifrovanie, **SP signing (a príp. encryption)
> certifikát** v `attributes` klienta, atribútové mappery (email, meno…) a test používateľ.
> Citlivé veci (heslá, privátne kľúče) do gitu nedávať – pre lokál stačí test používateľ
> s dočasným heslom; SP **privátne** kľúče ostávajú iba v properties SP, do Keycloaku ide
> len **verejný** SP certifikát.

> **IdP certifikát je generovaný Keycloakom.** Ak realm necháte vygenerovať kľúče pri prvom
> štarte, `idp.x509cert` budete musieť vyčítať až potom (z descriptor URL). Aby bol IdP cert
> **deterministický** (a `keycloak.webssodemo.saml.properties` sa nemenil pri každom rebuilde),
> vložte realm RS256 kľúč/cert priamo do `realm-export.json` (komponent
> `org.keycloak.keys.KeyProvider` s `privateKey`/`certificate`). Potom `idp.x509cert` ostáva fixný.

> Tým sa celá kapitola 2 (manuálne vytváranie) stáva už len **dokumentáciou** toho, čo je
> v `realm-export.json`; reálne stačí `docker compose up -d`.

---

## 2. Vytvorenie realm-u a SAML klienta v Keycloaku

### 2.1 Realm
1. V admin konzole vytvoriť realm, napr. **`webssodemo`**.

### 2.2 SAML klient (reprezentuje náš SP)
V realme `webssodemo` vytvoriť klienta typu **SAML**:

| Nastavenie | Hodnota |
|---|---|
| Client type | SAML |
| **Client ID** | `https://127.0.0.1.slovensko.sk.login` (= `sp.entityid`) |
| Name | webssodemo |
| **Valid redirect URIs** | `https://127.0.0.1:3001/*` |
| **Master SAML Processing URL** (ACS) | `https://127.0.0.1:3001/auth/saml/callback` |
| Logout Service POST/Redirect Binding URL | `https://127.0.0.1:3001/auth/saml/logout` |
| Name ID format | `unspecified` (musí sedieť s `sp.nameidformat`) |

Najjednoduchšia alternatíva: v Keycloaku použiť **Import client** a nahrať `sp-metadata.xml`
z `https://127.0.0.1:3001/sp-metadata.xml` – Keycloak si predvyplní endpointy a certifikáty.

### 2.3 Nastavenia podpisov a šifrovania klienta
Tieto musia byť **konzistentné** s `security.*` v properties (viď tabuľka v sekcii 4):

| Keycloak (Client → Settings / Keys) | Hodnota | Zodpovedá v properties |
|---|---|---|
| Sign documents | ON | IdP podpisuje `<Response>` → `want_messages_signed=true` |
| Sign assertions | ON | `want_assertions_signed=true` |
| Client signature required | ON | SP podpisuje AuthnRequest → `authnrequest_signed=true` |
| Encrypt assertions | ON/OFF | `want_assertions_encrypted` (viď nižšie) |
| Signature algorithm | RSA_SHA256 | `signature_algorithm=...rsa-sha256` |

- Ak je **Client signature required = ON**, treba do klienta v Keycloaku nahrať
  **podpisový certifikát SP** (`sp.x509cert`) – záložka **Keys → Signing**.
- Ak má Keycloak **šifrovať aserty** (`want_assertions_encrypted=true`), treba do klienta
  nahrať **šifrovací certifikát SP** (`sp.x509cert_enc`) – záložka **Keys → Encryption**.

> Aktuálne demo (`two_keys_setup...`) má `want_assertions_encrypted=true` a používa **rozdielne**
> kľúče pre podpis a šifrovanie (preto fork java-saml `2.9.1-SK`). Keycloak vie nahrať
> osobitný encryption a osobitný signing certifikát – tento scenár teda podporuje.
> **Tip pre prvý rozbeh:** šifrovanie najprv vypnúť (`want_assertions_encrypted=false`,
> v Keycloaku Encrypt assertions = OFF), rozbehať SSO, a šifrovanie zapnúť až potom.

### 2.4 Single Logout (SLO) – špecifiká Keycloaku
Demo robí **SP-initiated SLO** s **redirect bindingom** a podpísaným `<LogoutRequest>`
(`logoutrequest_signed=true`, `logoutresponse_signed=true`). V Keycloak klientovi preto:

- Nastaviť **Logout Service Redirect Binding URL** = `https://127.0.0.1:3001/auth/saml/logout`.
- **Force POST Binding = OFF** (inak Keycloak pošle LogoutResponse cez POST a vznikne
  binding mismatch voči `idp.single_logout_service.binding=...HTTP-Redirect`).
- **Force Name ID Format / Name ID format** zladiť s `sp.nameidformat` (= `unspecified`),
  inak môže byť NameID v LogoutRequeste pre Keycloak neznámy a SLO zlyhá.
- Keycloak musí mať **SP signing cert** (na overenie podpisu LogoutRequestu) – rovnaký ako pre AuthnRequest.

### 2.5 Testovací používateľ
V realme vytvoriť používateľa (Users → Add user), nastaviť heslo (Credentials),
prípadne e-mail/meno pre atribúty.

---

## 3. Získanie IdP údajov z Keycloaku

Keycloak SAML IdP descriptor (metadáta) sú na:

```
http://localhost:8081/realms/webssodemo/protocol/saml/descriptor
```

Z neho (alebo z realm Keys) získať hodnoty pre SP konfiguráciu:

| Properties kľúč | Hodnota z Keycloaku |
|---|---|
| `idp.entityid` | `http://localhost:8081/realms/webssodemo` |
| `idp.single_sign_on_service.url` | `http://localhost:8081/realms/webssodemo/protocol/saml` |
| `idp.single_logout_service.url` | `http://localhost:8081/realms/webssodemo/protocol/saml` |
| `idp.single_logout_service.response.url` | `http://localhost:8081/realms/webssodemo/protocol/saml` |
| `idp.x509cert` | RSA signing certifikát realm-u (z descriptor XML, element `<ds:X509Certificate>`, alebo Realm settings → Keys → RS256 → Certificate) |

> Pri produkčnom/HTTPS Keycloaku použiť `https://<host>/realms/webssodemo/...`.

> **Rotácia kľúčov Keycloaku.** Keycloak môže mať aktívnych viac signing kľúčov a v descriptor
> XML vypísať viac `<ds:X509Certificate>`. `idp.x509cert` drží **iba jeden** certifikát –
> musí to byť **aktívny RS256 signing** kľúč. Ak chcete podporiť viac kľúčov (kvôli rotácii),
> použite v java-saml multi-cert variant:
> ```ini
> onelogin.saml2.idp.x509certMulti.signing.0=<cert A>
> onelogin.saml2.idp.x509certMulti.signing.1=<cert B>
> ```
> Po rotácii kľúča v Keycloaku treba certifikát v properties aktualizovať, inak prestane
> sedieť podpis a SP odpovede zamietne.

---

## 4. Nový konfiguračný súbor pre SP

Vytvoriť `bind-mounts/portal/usr/local/tomcat/conf/keycloak.webssodemo.saml.properties`
ako kópiu `two_keys_setup.webssodemo.saml.properties` a upraviť **iba** sekciu IdP:

```ini
# --- IdP = Keycloak ---
onelogin.saml2.idp.entityid=http://localhost:8081/realms/webssodemo
onelogin.saml2.idp.single_sign_on_service.url=http://localhost:8081/realms/webssodemo/protocol/saml
onelogin.saml2.idp.single_sign_on_service.binding=urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect
onelogin.saml2.idp.single_logout_service.url=http://localhost:8081/realms/webssodemo/protocol/saml
onelogin.saml2.idp.single_logout_service.response.url=http://localhost:8081/realms/webssodemo/protocol/saml
onelogin.saml2.idp.single_logout_service.binding=urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect
onelogin.saml2.idp.x509cert=<RS256 signing certifikát realm-u, Base64 bez hlavičiek>
```

SP sekcia (`sp.*`) a väčšina `security.*` ostáva nezmenená. Skontrolovať konzistenciu:

| Properties | Musí sedieť s Keycloak klientom |
|---|---|
| `security.authnrequest_signed=true` | Client signature required = ON + nahratý signing cert SP |
| `security.want_messages_signed=true` | Sign documents = ON |
| `security.want_assertions_signed=true` | Sign assertions = ON |
| `security.want_assertions_encrypted` | Encrypt assertions (ON ⇔ true) |
| `security.signature_algorithm=...rsa-sha256` | Signature algorithm = RSA_SHA256 |
| `sp.nameidformat=...unspecified` | Name ID format klienta |
| `strict=true`, `want_xml_validation=true` | (odporúčané ponechať) |

### Prepnutie aplikácie na nový profil
V `Settings.java` prepnúť načítavaný súbor:

```java
// conf.load(new FileInputStream(System.getProperty("catalina.base") + "/conf/two_keys_setup.webssodemo.saml.properties"));
conf.load(new FileInputStream(System.getProperty("catalina.base") + "/conf/keycloak.webssodemo.saml.properties"));
```

A pridať nový súbor ako volume do `docker-compose.yml`:

```yaml
      - ./bind-mounts/portal/usr/local/tomcat/conf/keycloak.webssodemo.saml.properties:/usr/local/tomcat/conf/keycloak.webssodemo.saml.properties
```

> Voliteľne lepšie riešenie: cestu k properties čítať z env premennej / system property,
> aby sa profil dal prepínať bez rekompilácie.

### Kompatibilita šifrovacích algoritmov (ak `want_assertions_encrypted=true`)
Keycloak vie aserty šifrovať modernými algoritmami (napr. **RSA-OAEP** pre kľúč a
**AES-GCM** pre dáta). Staršie java-saml/xmlsec s nimi mali problém, ale tento projekt už
má v `pom.xml` povýšený **`xmlsec 4.0.2`** (a vylúčený starý), takže by mali fungovať.
Ak by dešifrovanie zlyhávalo, v Keycloak klientovi skúste prepnúť **Encryption Algorithm**
(napr. na RSA-OAEP) a **Encryption Key Algorithm**, prípadne dočasne šifrovanie vypnúť
a overiť zvyšok toku. Sleduje sa to v logoch SP (`SamlServlet` loguje aj raw/decoded SAMLResponse).

---

## 5. Mapovanie atribútov (voliteľné)

Aby `auth.getAttributes()` v `SamlServlet` vracal použiteľné údaje, pridať klientovi
v Keycloaku **Client scopes → Mappers** (typu *User Property* / *User Attribute*), napr.:

- `email`, `firstName` (`given name`), `lastName` (`surname`), `username`.

Mená atribútov a SAML Attribute NameFormat si zvoliť podľa toho, čo appka očakáva zobraziť.

---

## 6. Postup nasadenia a testovanie

1. Spustiť Keycloak, vytvoriť realm `webssodemo`, SAML klienta a používateľa (sekcia 1–2).
2. Stiahnuť IdP descriptor a doplniť `idp.x509cert` + URL do nového properties (sekcia 3–4).
3. Nahrať podpisový (a príp. šifrovací) certifikát SP do Keycloak klienta.
4. Prepnúť `Settings.java` a doplniť volume v compose, potom:
   ```
   mvn clean install
   docker compose up -d --build
   ```
5. Otvoriť `https://127.0.0.1:3001/login` → presmerovanie na Keycloak login → prihlásiť sa
   testovacím používateľom → návrat na ACS `/auth/saml/callback` → úspešné prihlásenie.
6. Overiť SLO cez `/logout`.

### Odporúčaný „happy path" pre prvý rozbeh
Najprv zjednodušiť bezpečnosť, aby sa odladilo samotné SSO, potom postupne pritvrdzovať:

1. `want_assertions_encrypted=false`, Encrypt assertions = OFF.
2. (príp.) `authnrequest_signed=false`, Client signature required = OFF.
3. Po úspešnom SSO postupne zapnúť podpis AuthnRequestu a šifrovanie asertov.

---

## 7. Časté problémy (troubleshooting)

- **Invalid signature / signature validation failed** – nesprávny `idp.x509cert`
  (zlý realm key, alebo skopírovaný aj s hlavičkami/medzerami). Použiť RS256 signing cert.
- **Destination/Audience mismatch** – `sp.entityid` ≠ Client ID v Keycloaku, alebo ACS URL
  v Keycloaku ≠ `sp.assertion_consumer_service.url`.
- **AuthnRequest rejected (signature required)** – Keycloak vyžaduje podpísaný AuthnRequest,
  ale SP ho neposlal podpísaný, alebo chýba nahratý SP signing cert v Keycloaku.
- **Decryption failed** – Keycloak šifruje asert iným certifikátom než `sp.x509cert_enc`,
  alebo SP nemá zodpovedajúci `privatekey_enc`.
- **Clock skew / NotBefore** – nezosynchronizovaný čas medzi kontajnermi → `strict` zamietne odpoveď.
- **HTTP vs HTTPS hostname** – nekonzistentné `entityid`/URL (http vs https, port) spôsobia
  audience/destination chyby. URL musia presne sedieť na oboch stranách.
- **SLO binding mismatch** – Keycloak má zapnuté *Force POST Binding*, ale SP očakáva
  redirect binding (`idp.single_logout_service.binding=...HTTP-Redirect`) → vypnúť Force POST Binding.
- **SLO „Invalid NameID"** – NameID/format v `<LogoutRequest>` nesedí s tým, čo Keycloak
  očakáva → zladiť `sp.nameidformat` a Name ID format klienta.
- **Stratená session po POST z Keycloaku** – `SameSite`/`Secure` cookie pri cross-site POST
  na HTTPS ACS z HTTP Keycloaku → prejsť na HTTPS Keycloak.
- **Podpis nesedí po čase** – Keycloak zrotoval signing kľúč → aktualizovať `idp.x509cert`
  (alebo použiť `idp.x509certMulti.signing.N`).

---

## 8. Zhrnutie zmien v repozitári

- [ ] `docker-compose.yml` – pridať službu `keycloak` (+ volume nového properties súboru).
- [ ] `bind-mounts/.../conf/keycloak.webssodemo.saml.properties` – nový IdP profil (Keycloak).
- [ ] `Settings.java` – prepnúť načítavaný properties súbor (alebo zaviesť env premennú).
- [ ] Keycloak: realm `webssodemo`, SAML klient, používateľ, atribútové mappery, SP certifikáty.
- [ ] `keycloak/realm-export.json` (+ príp. `keycloak/Dockerfile`) – Keycloak ako samostatný,
      predkonfigurovaný image s automatickým importom realmu (viď sekcia 1.1).```