# Java-saml slovensko.sk login

Tomcat aplikácii s websso prihlasovaním voči Slovensko.sk s použitím knižnice [SAML Java Toolkit od Onelogin](https://github.com/SAML-Toolkits/java-saml). 

## Rozdielne kľúče pre podpisovanie a šifrovanie

Pri tejto integrácii je možné v NASESe zaregistrovať service provider metadáta s rozdielnymi kľúčmi a certifikátmi pre podpisovanie a šifrovanie.
Java-saml od Onelogin ale podporuje nastavenie iba jedného rovnakého certifikátu a kľúča pre obe použitia, čo je pri SAML bežné.

Ak ale naozaj potrebujete nastaviť rozdielne certifikáty a kľúče, použite tento [fork java-saml od Archimetes](https://github.com/archimetes/java-saml).
Fork si potrebujete lokálne zbuildovať, nainštalovať a v `pom.xml` použiť túto verziu `2.9.1-SK`.
Konfigurácia pre tento scenár je uložená v: [two_keys_setup.webssodemo.saml.properties](bind-mounts/portal/usr/local/tomcat/conf/two_keys_setup.webssodemo.saml.properties)

## Inštalácia

Zbuildujte si tento projekt:
```
mnv clean install
```
Výsledný artefakt ROOT.war sa nakopíruje do bind-mounts/portal/usr/local/tomcat/webapps kde už je dostupný pre Tomcat. 
Stačí už len spustiť kontajner s Tomcatom:
```
docker compose up -d 
```
A aby sme si to celé mohli aj pozrieť, treba ešte nasledovné veci:
* VPN na FIX prostredie Slovensko.sk
* otvoriť si stránku https://127.0.0.1:3001/ a pridať si jej certifikát medzi dôveryhodné

## Konfigurácia pre java-saml

Konfiguračný súbor pre SAML toolkit býva uložený na classpath pod názvom *onelogin.saml.properties*. 
Pre účely testovania a vývoja je ale výhodnejšie, keď je uložný mimo classpath a načítava vždy nanovo. Od toho máme triedu [Settings.java](src/main/java/com/archimetes/cgpcon/websso/Settings.java), 
ktorá číta konfiguráciu z [webssodemo.saml.properties](bind-mounts/portal/usr/local/tomcat/conf/webssodemo.saml.properties). 
Tento súbor obsahuje veľmi dobré komentára od autora takže to tu nemusím podrobne vysvetlovať. Hodnoty, ktoré budete meniť sú:

Pre service providera, teda pre vás:

* onelogin.saml2.sp.entityid - Je to síce URL adresa ale táto konkrétne nemusí existovať
* onelogin.saml2.sp.assertion_consumer_service.url - tak toto už musí existovať, host musí byť dostupný z klienta a cesta je hodnota pre servlet mapping samlcallbackServlet vo [web.xml](src/main/webapp/WEB-INF/web.xml)
* onelogin.saml2.sp.single_logout_service.url - to isté pre SingleLogoutServiceServlet. 
* onelogin.saml2.sp.x509cert_enc - Base64 kódovaný certifikát pre šifrovanie 
* onelogin.saml2.sp.privatekey_enc - jeho klúč
* onelogin.saml2.sp.x509cert - certifikát pre podpisovanie
* onelogin.saml2.sp.privatekey - a klúč

Pre Identity providera okopírujete hodnoty, čo ste dostali od NASESu v metadátovom XML
* onelogin.saml2.idp.entityid=https://prihlasenie.upvsfix.gov.sk/oam/fed
* onelogin.saml2.idp.single_sign_on_service.url=https://prihlasenie.upvsfix.gov.sk/oamfed/idp/samlv20
* onelogin.saml2.idp.single_logout_service.url=https://prihlasenie.upvsfix.gov.sk/oamfed/idp/samlv20
* onelogin.saml2.idp.single_logout_service.response.url=https://prihlasenie.upvsfix.gov.sk/oamfed/idp/samlv20
* onelogin.saml2.idp.x509cert=MIIDXjCCAkagAwIBA................=

## Prihlásenie cez Keycloak (lokálne demo)

Okrem produkčného IdP (UPVS FIX) je možné sa prihlasovať aj voči lokálnemu **Keycloaku**,
ktorý vystupuje ako SAML 2.0 Identity Provider. Postup a princíp sú podrobne popísané v [plan.md](plan.md).

Aplikácia a Keycloak sú **dva samostatné kontajnery** spúšťané osobitne. Poradie: najprv
**Tomcat (SP)**, potom **Keycloak (IdP)**. Všetky príkazy používajú prepínač `-f` a pred každým
je komentár, **na ktorom počítači sa spúšťa**.

### Deployment – prehľad režimov

| Režim | App URL (SP) | Keycloak URL (IdP) |
|---|---|---|
| **local** | `https://127.0.0.1:3001/` | `http://127.0.0.1:8081/` |
| **kistest** | `https://kistest:3001/` | `http://kistest:8081/` |
| **hybrid** | `https://127.0.0.1:3001/` (lokálne) | `http://kistest:8081/` (na serveri) |

#### 1) LOCAL – všetko na tvojom počítači

```bash
# === TVOJ POCITAC (localhost) ===
# 1) SP – demo appka (Tomcat)
docker compose -f docker-compose.demo-local.yml up -d

# === TVOJ POCITAC (localhost) ===
# 2) IdP – Keycloak
docker compose -f docker-compose.keycloak.yml up -d
```

#### 2) KISTEST – všetko na serveri kistest

```bash
# === SERVER kistest ===
# 1) SP – demo appka (Tomcat)
docker compose -f docker-compose.demo-kistest.yml up -d

# === SERVER kistest ===
# 2) IdP – Keycloak
docker compose -f docker-compose.keycloak.yml up -d
```

#### 3) HYBRID – appka lokálne, Keycloak na kisteste

```bash
# === TVOJ POCITAC (localhost) ===
# 1) SP – demo appka (Tomcat, prihlasuje sa proti Keycloaku na kisteste)
docker compose -f docker-compose.demo-kistest.hybrid.yml up -d

# === SERVER kistest ===
# 2) IdP – Keycloak (beží na serveri)
docker compose -f docker-compose.keycloak.yml up -d
```

> Pozn.: Prostredie sa neprepína env premennými, ale **výberom compose súboru**, ktorý namountuje
> príslušný SP properties profil na `.../conf/keycloak.webssodemo.saml.properties`. `docker-compose.yml`
> je len alias (`include`) na `docker-compose.demo-local.yml`, takže `docker compose up -d` bez `-f`
> robí to isté ako **local**.

**Hybridný režim** = SP beží lokálne, ale prihlasuje sa proti Keycloaku na **kisteste**. Keycloak
sa lokálne nespúšťa (beží už na kisteste). Predpoklad: `kistest` sa z tvojho prehliadača rozloží
na IP kistest servera (VPN + `hosts`/DNS). Realm na kisteste už obsahuje aj klienta
`https://127.0.0.1.slovensko.sk.login`, takže lokálna SP funguje bez ďalšej registrácie.
Podrobnosti v [plan.md](plan.md) (sekcia 4b).

> Pri zmene Java kódu najprv `mvn clean install` a pred reštartom zmazať rozbalený
> priečinok `bind-mounts/portal/usr/local/tomcat/webapps/ROOT`, aby Tomcat nasadil novú `ROOT.war`.

Konfigurácia SP pre tento scenár je v
[keycloak.webssodemo.saml.properties](bind-mounts/portal/usr/local/tomcat/conf/keycloak.webssodemo.saml.properties)
(sekcia `idp.*` mieri na Keycloak). Ktorý profil sa načíta, sa prepína v
[Settings.java](src/main/java/com/archimetes/cgpcon/websso/Settings.java).

Keycloak realm (klient, mappery, test používateľ aj **fixný IdP podpisový kľúč/cert**) je
definovaný v [keycloak/realm-export.json](keycloak/realm-export.json) a importuje sa pri štarte
(`--import-realm`). Vďaka fixnému RS256 kľúču sa `onelogin.saml2.idp.x509cert` pri reštarte nemení.

### Prístupy

| Účel | URL | Prihlasovacie údaje |
|---|---|---|
| Demo aplikácia (prihlásenie cez SAML) | https://127.0.0.1:3001/login | `testuser` / `test123` |
| Admin konzola Keycloaku (master realm) | http://kistest:8081/ | `admin` / `admin` |
| SAML metadáta IdP (Keycloak) | http://kistest:8081/realms/webssodemo/protocol/saml/descriptor | – |
| SAML metadáta SP | https://127.0.0.1:3001/sp-metadata.xml | – |

> Pozn.: `admin/admin` je len pre **administráciu Keycloaku**, do demo aplikácie sa prihlasuje
> používateľom **`testuser` / `test123`** (existuje v realme `webssodemo`).
