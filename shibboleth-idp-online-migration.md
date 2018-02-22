# Online migrace Shibboleth IdP

## Úvod

Tento dokument popisuje, jak *bezvýpadkově* přemigrovat Shibboleth IdP z jednoho serveru na jiný, aniž by si toho uživatelé všimli. Zároveň není potřeba žádné spolupráce s operátorem federace, protože nedochází ke změně metadat. Využijete toho např. v situaci, kdy chcete migrovat ze starší verze operačního systému na novější. Na CESNETu jsme takto migrovali z Debianu verze 8 *"Jessie"* na Debian verze 9 *"Stretch"*.

## Předpoklady

Pro lepší orientaci v textu zavedeme následující konvenci.

### Stávající server

* DNS: `idp.example.org`
* IPv4: `192.0.2.10`
* IPv6: `2001:db8::10`
* Název serveru (hostname): `idp.example.org`
* entityID: `https://idp.example.org/idp/shibboleth`

### Nový server

* DNS: není třeba, pro instalaci stačí přístup přes IP adresu
* IPv4: `192.0.2.20`
* IPv6: `2001:db8::20`
* Název serveru (hostname): `idp.example.org`
* entityID: `https://idp.example.org/idp/shibboleth`

## Postup

Nainstalujeme a nakonfigurujeme nový server jako kdyby se jednalo o stávající server `idp.example.org`.

Při konfiguraci Shibboleth IdP na novém serveru použijeme *entityID* stávajícího serveru, tedy `https://idp.example.org/idp/shibboleth`, použijeme také stávající "ceritifikáty" (ve skutečnosti páry veřejných a odpovídajících soukromých klíčů) `idp-encryption.{crt,key}` a `idp-signing.{crt,key}`, které se standardně nachází v adresáři `/opt/shibboleth-idp/credentials/`. S největší pravděpodobností můžeme zapomenout na "certifikáty" takzvaného backchannelu, tedy `backchannel.{crt,p12}`, jelikož ho v dnešní době již téměř nikdo nepoužívá.

Samozřejmostí je identická konfigurace např. atributů (`attribute-resolver.xml`), ale i pravidel jejich uvolňování (`attribute-filter.xml`) a tak dále. Nezapomeneme ani na přenos databáze s persistentními identifikátory a samozřejmě použít stejnou sůl.

## Otestování

Nyní si vše otestujeme pomocí jednoduchého triku. V konfiguračním souboru `/etc/hosts` (Linux, BSD, macOS) *na naší pracovní stanici* si upravíme DNS následujícím způsobem:

```
192.0.2.20    idp.example.org
2001:db8::20  idp.example.org
```

Tím jsme docílili toho, že při zadání adresy `https://idp.example.org` do webového prohlížeče budeme ve skutečnosti komunikovat s novým serverem, kterému jsme v DNS zatím nedali žádné jméno. Jelikož tento server zná privátní klíče k veřejným klíčům uvedených v metadatech federace, bude vystupovat v rámci federace jako stávající server `idp.example.org`.

Tento trik funguje, protože již není potřeba používat "backchannel" a veškerá komunikace (přihlášení, atributy, ...) funguje pomocí cookies webového prohlížeče.

Můžeme se tedy přihlásit na libovolnou službu ve federaci a otestovat, zda služba funguje správně i s nově nakonfigurovaným IdP.

## Přepnutí

Po úspěšném otestování můžeme přistoupit k samotnému přepnutí. To provedeme ve třech krocích pomocí změn DNS záznamů:

1. Snížíme TTL hodnoty u DNS záznamů A (IPv4) i AAAA (IPv6) pro `idp.example.org` na co nejnižší hodnotu, která je možná, abychom dokázali přepnout uživatele co nejrychleji po vypršení platnosti nakešovaných DNS záznamů. Nyní vyčkáme, až se staré nakešované DNS záznamy nahradí novými záznamy.

2. Necháme starý server v DNS přejmenovat z `idp.example.org` na `idp-old.example.org` a nový server následně pojmenujeme jako `idp.example.org`. TTL ponecháme na snížených hodnotách, abychom se v případě potřeby mohli změnou DNS záznamů vrátit zpět a používat i nadále IdP na původním serveru. Ten, kdo má stále nakešované staré DNS záznamy, se připojí na staré IdP, přihlásí se a vše funguje. Ten, kdo má již nové DNS záznamy, se připojí na nové IdP, přihlásí se a také vše funguje.

3. Nyní již jen necháme zvýšit hodnoty TTL na standardní úroveň a migrace je hotova. Starý server (včetně DNS záznamu `idp-old.example.org` a alokované IP adresy) můžeme zrušit. Samozřejmostí je také odebrání záznamů z `/etc/hosts` pro doménu `idp.example.org`.

## Kontakt

V případě jakýchkoliv nejasností, dotazů či tipů, pište na e-mail info@eduid.cz.

