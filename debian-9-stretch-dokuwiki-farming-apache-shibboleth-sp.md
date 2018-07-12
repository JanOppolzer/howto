# DokuWiki Farming s Apache a Shibboleth SP na Debianu 9 (Stretch)

Tento návod navazuje na [předchozí variantu pro Debian 8 (Jessie)](https://github.com/JanOppolzer/howto/blob/master/debian-8-jessie-dokuwiki-farming-apache-shibboleth-sp.md), avšak liší se v tom, že v Debianu 9 (Stretch) již není DokuWiki v balíčkovacím systému a je tedy nutné nainstalovat ji ze zdrojových kódů.

## Úvod

V případě, že chceme provozovat několik různých DokuWiki instancí pro různé projekty, není potřeba mít pro každou DokuWiki vlastní [virtuální nebo fyzický] server a starat se tak o několik operačních systémů. Je možné použít tzv. [DokuWiki Farming](https://www.dokuwiki.org/farms).

Všechny instance sdílí jednu instalaci DokuWiki a také rozšíření. Mají však své vlastní stránky a také nastavení (přístupová práva, vzhled, použitá rozšíření, způsob přihlašování, ...).

Tento dokument počítá s tím, že všechny instance budou používat přihlašování pomocí [České akademické federace identit eduID.cz](https://www.eduid.cz). To je implementováno pomocí rozšíření [dokuwiki-shibboleth-auth](https://github.com/JanOppolzer/dokuwiki-shibboleth-auth). Zároveň hostující server bude z pohledu federace jeden poskytovatel služby neboli jedna služba (Service Provider, SP) s jedním *entityID*. Jednotlivé instance DokuWiki běžící na svých vlastních adresách (doménách) budou mít své "koncové body" (endpointy) uvedeny v metadatech jediné entity. V návodu je použit CESNETí "linker" (horní záhlaví/hlavička) a [jednotný vzhled pro DokuWiki](https://wiki.cesnet.cz/doku.php?id=groups:weby:dokuwiki:start).

Níže popsaný přístup sníží nároky jak na správu operačních systémů (bude nutné spravovat jen jeden operační systém), tak instalace DokuWiki i federativní autentizace. Jeden operační systém, jedna instalace DokuWiki, jeden Shibboleth SP s jedním certifikátem atd. Budeme-li mít certifikát pro webserver obsahující v alternativních jménech všechny potřebné domény (doporučený způsob), budeme mít i jeden SSL certifikát. Znamená to ovšem, že když vyprší SSL certifikát, vyprší pro všechny instance najednou (ano, vyplatí se monitorovat). Rozbije-li se DokuWiki např. při neopatrné aktualizaci, rozbije to všechny instance najednou. (*With great power comes great responsibility.*)

V krocích níže se předpokládá použití [Debianu 9](https://www.debian.org/releases/stretch/) s kódovým označením [Stretch](https://www.debian.org/releases/stretch/).

Jako první nainstalujeme a nakonfigurujeme Apache a zprovozníme si virtuální hosty včetně SSL konfigurace. Následně nainstalujeme a nakonfigurujeme Shibboleth SP a zprovozníme federativní přihlašování. Poté bude na řadě instalace a konfigurace DokuWiki a její nakonfigurování v Apachi. Posledním krokem, který bude v řadě případů nepotřebný, bude návod, jak převést stávající DokuWiki instance na novou instalaci -- to pro případ, že byste konsolidovali více existujících instancí DokuWiki na několika serverech na jeden server.

## Konvence

V návodu používám následující:

* DNS název fyzického serveru (hostname): `blackhole.cesnet.cz`
* entityID služby ve federaci: `https://blackhole.cesnet.cz/shibboleth`
* hostované weby/domény: `blackhole.cesnet.cz`, `blackhole1.cesnet.cz`, `blackhole2.cesnet.cz`
* SSL certifikát se všemi hostovanými doménami: `/etc/ssl/certs/blackhole.cesnet.cz.crt.pem`
* privátní klíč k SSL certifikátu: `/etc/ssl/private/blackhole.cesnet.cz.key.pem`
* řetěz certifikátů až ke kořenové CA: `/etc/ssl/certs/chain_TERENA_SSL_CA_3.pem`

## Aktualizace

Před začátkem nejprve aktualizujeme systém:

```bash
apt update
apt upgrade
```

V případě potřeby (např. když bude k dispozici nové jádro) restartujeme server:

```bash
shutdown -r now
```

## Apache

Instalaci Apache provedeme obvyklým způsobem pomocí příkazu `apt`.

```bash
apt install apache2
```

Nejprve nakonfigurujeme server `blackhole.cesnet.cz`. Vytvoříme konfigurační soubor `/etc/apache2/sites-available/blackhole.cesnet.cz.conf` s následujícím obsahem:

```apache
<VirtualHost *:80>
    ServerName              blackhole.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    Redirect                permanent / https://blackhole.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/error-blackhole.log
    CustomLog               ${APACHE_LOG_DIR}/access-blackhole.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName              blackhole.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    DocumentRoot            /var/www/blackhole.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/ssl-error-blackhole.log
    CustomLog               ${APACHE_LOG_DIR}/ssl-access-blackhole.log combined

    SSLEngine               On
    SSLProtocol             All -SSLv2 -SSLv3
    SSLHonorCipherOrder     On
    SSLCompression          Off
    Header                  always set Strict-Transport-Security "max-age=15768000"
    SSLCipherSuite          'EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH:+CAMELLIA256:+AES256:+CAMELLIA128:+AES128:+SSLv3:!aNULL:!eNULL:!LOW:!3DES:!MD5:!EXP:!PSK:!DSS:!RC4:!SEED:!ECDSA:CAMELLIA256-SHA:AES256-SHA:CAMELLIA128-SHA:AES128-SHA'

    SSLCertificateFile      /etc/ssl/certs/blackhole.cesnet.cz.crt.pem
    SSLCertificateKeyFile   /etc/ssl/private/blackhole.cesnet.cz.key.pem
    SSLCertificateChainFile /etc/ssl/certs/chain_TERENA_SSL_CA_3.pem
</VirtualHost>
```

**FIXME: HTTP hlavičky**


Vytvoříme adresář `/var/www/blackhole.cesnet.cz/` a v něm soubor `index.html` obsahující pouze text `blackhole.cesnet.cz`.

```bash
mkdir -p /var/www/blackhole.cesnet.cz/
echo 'blackhole.cesnet.cz' > /var/www/blackhole.cesnet.cz/index.html
```

SSL certifikát umístíme do souboru `/etc/ssl/certs/blackhole.cesnet.cz.crt.pem`, privátní klíč do `/etc/ssl/private/blackhole.cesnet.cz.key.pem` a řetěz certifikátů až ke kořenové CA do `/etc/ssl/certs/chain_TERENA_SSL_CA_3.pem`.

Nyní v konfiguraci Apache vypneme server `000-default.conf` (`/etc/apache2/sites-available/000-default.conf`), který je po instalaci Apache ve výchozím stavu povolený. Následně zapneme nově nadefinovaný server `blackhole.cesnet.cz.conf`.

Jelikož v konfiguraci jednotlivých serverů používáme _SSL certifikáty_ pro komunikaci přes _HTTPS_, je potřeba aktivovat modul `ssl` v Apachi. Navíc u _SSL_ nastavujeme _HSTS_ hlavičku, takže je potřeba aktivovat i modul `headers`. Pak již zbývá jen restartovat Apache.

```bash
a2dissite 000-default.conf
a2ensite blackhole.cesnet.cz.conf
a2enmod ssl headers
systemctl restart apache2
```

Vyzkoušíme se připojit webovým prohlížečem k serveru a měli bychom vidět text `blackhole.cesnet.cz`. Nyní můžeme přejít ke konfiguraci zbývajících virtuálních hostů `blackhole1.cesnet.cz` a `blackhole2.cesnet.cz`. Konfigurační soubory (`blackhole1.cesnet.cz.conf`, `blackhole2.cesnet.cz`) budou velice podobné výše uvedenému souboru `blackhole.cesnet.cz`. Změny budou jen v parametrech `ServerName`, `DocumentRoot`, `Redirect`, `ErrorLog` a `CustomLog`.

Konfigurační soubor `/etc/apache2/sites-available/blackhole1.cesnet.cz.conf`:

```apache
<VirtualHost *:80>
    ServerName              blackhole1.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    Redirect                permanent / https://blackhole1.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/error-blackhole1.log
    CustomLog               ${APACHE_LOG_DIR}/access-blackhole1.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName              blackhole1.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    DocumentRoot            /var/www/blackhole1.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/ssl-error-blackhole1.log
    CustomLog               ${APACHE_LOG_DIR}/ssl-access-blackhole1.log combined

    SSLEngine               On
    SSLProtocol             All -SSLv2 -SSLv3
    SSLHonorCipherOrder     On
    SSLCompression          Off
    Header                  always set Strict-Transport-Security "max-age=15768000"
    SSLCipherSuite          'EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH:+CAMELLIA256:+AES256:+CAMELLIA128

    SSLCertificateFile      /etc/ssl/certs/blackhole.cesnet.cz.crt.pem
    SSLCertificateKeyFile   /etc/ssl/private/blackhole.cesnet.cz.key.pem
    SSLCertificateChainFile /etc/ssl/certs/chain_TERENA_SSL_CA_3.pem
</VirtualHost>
```

Konfigurační soubor `/etc/apache2/sites-available/blackhole2.cesnet.cz.conf`:

```apache
<VirtualHost *:80>
    ServerName              blackhole2.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    Redirect                permanent / https://blackhole2.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/error-blackhole2.log
    CustomLog               ${APACHE_LOG_DIR}/access-blackhole2.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName              blackhole2.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    DocumentRoot            /var/www/blackhole2.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/ssl-error-blackhole2.log
    CustomLog               ${APACHE_LOG_DIR}/ssl-access-blackhole2.log combined

    SSLEngine               On
    SSLProtocol             All -SSLv2 -SSLv3
    SSLHonorCipherOrder     On
    SSLCompression          Off
    Header                  always set Strict-Transport-Security "max-age=15768000"
    SSLCipherSuite          'EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH:+CAMELLIA256:+AES256:+CAMELLIA128

    SSLCertificateFile      /etc/ssl/certs/blackhole.cesnet.cz.crt.pem
    SSLCertificateKeyFile   /etc/ssl/private/blackhole.cesnet.cz.key.pem
    SSLCertificateChainFile /etc/ssl/certs/chain_TERENA_SSL_CA_3.pem
</VirtualHost>
```

Vytvoříme si nové adresáře pro zbývající weby a v každém z nich vytvoříme soubor `index.html` s textem odpovídajícím názvu serveru pro daný web:

```bash
mkdir -p /var/www/blackhole{1,2}.cesnet.cz/
for i in `seq 1 2`; do echo blackhole$i.cesnet.cz >> /var/www/blackhole$i.cesnet.cz/index.html; done
```

Povolíme nově definované virtuální hosty a znovu nahrajeme (stačí `reload`, není třeba `restart`) konfiguraci Apache:

```bash
a2ensite blackhole1.cesnet.cz.conf blackhole2.cesnet.cz.conf
systemctl reload apache2
```

Nyní ověříme, že se dostaneme na všechny weby a to jak protokolem _HTTP_ (musí dojít k přesměrování), tak protokolem _HTTPS_: `blackhole.cesnet.cz`, `blackhole1.cesnet.cz` i `blackhole2.cesnet.cz`.

### PHP

Pro běh _DokuWiki_ budeme potřebovat interpret jazyku _PHP_, ten nainstalujeme standardním způsobem z balíčkovacího systému.

```bash
apt install php php-gd php-xml
```

Že vše v pořádku funguje vyzkoušíme triviálním skriptem obsahujícím pouze `<?php PHPInfo(); ?>`.

```bash
cat >> /var/www/blackhole.cesnet.cz/phpinfo.php << EOF
<?php PHPInfo(); ?>
EOF
```

Nyní se přesvědčíme, že PHP funguje. Podíváme se na stránku `https://blackhole.cesnet.cz/phpinfo.php` a pokud je vše v pořádku, můžeme skript smazat.

```bash
rm /var/www/blackhole.cesnet.cz/phpinfo.php
```

## Shibboleth Service Provider

Provozovat Shibboleth SP ve spolupráci s web serverem Apache je velice jednoduché, stačí doinstalovat modul do Apache, povolit ho a upravit nastavení virtuálních hostů, kde má být modul použit:

Instalace modulu Shibboleth SP do Apache:

```bash
apt install libapache2-mod-shib2
```

Nyní se můžeme pustit do konfigurace Shibboleth SP. Po instalaci výše uvedeného balíčku si vygenerujeme certifikát pro Shibboleth, který bude v metadatech entity. Následující příkaz vygeneruje self-signed certifikát, nastaví mu vhodná práva i jméno, abychom měli co nejsnazší nastavení. Tento certifikát bude mít platnost 10 let (pomocí parametru `-y` je možné ovlivnit jeho platnost v letech), bude vystaven pro server/hostname blackhole.cesnet.cz (parametr `-h`) s _entityID_ `https://blackhole.cesnet.cz/shibboleth` (parametr `-e`).

```bash
shib-keygen -h blackhole.cesnet.cz -e https://blackhole.cesnet.cz/shibboleth
```

Zbytek konfigurace se odehrává v konfiguračním souboru `/etc/shibboleth/shibboleth2.xml`.

Jako první nastavíme _entityID_ (`https://blackhole.cesnet.cz/shibboleth`), což je jedinečný identifikátor entity v rámci federace identit:

```xml
<ApplicationDefaults entityID="https://blackhole.cesnet.cz/shibboleth"
                     REMOTE_USER="eppn persistent-id targeted-id"
                     cipherSuites="ECDHE+AESGCM:ECDHE:!aNULL:!eNULL:!LOW:!EXPORT:!RC4:!SHA:!SSLv2">
```

Dále nakonfigurujeme SSL (změníme `handlerSSL` na _true_ a `cookieProps` na _https_):

```xml
<Sessions lifetime="28800" timeout="3600" relayState="ss:mem"
          checkAddress="false" handlerSSL="true" cookieProps="https">
```

Před konfigurací přihlašování, čili konfigurací WAYF/DS (nastavíme v `discoveryURL` na `https://ds.eduid.cz/wayf.php`), je vhodné projít [dokumentaci k WAYFu](https://www.eduid.cz/cs/tech/wayf/sp) a případně i stránku, kde se nachází [generátor filtru](https://ds.eduid.cz/filter.php):

```xml
<SSO discoveryProtocol="SAMLDS" discoveryURL="https://ds.eduid.cz/wayf.php">
     SAML2 SAML1
</SSO>
```

Metadata služby musíme obohatit o další informace -- přidáme atribut `template="metadata-template.xml"` a souboru `metadata-template.xml` se budeme věnovat níže:

```xml
<Handler type="MetadataGenerator" Location="/Metadata" signing="false" template="metadata-template.xml"/>
```

Můžeme si povolit zobrazení hodnot atributů na diagnostrické stránce (`https://blackhole.cesnet.cz/Shibboleth.sso/Session`):

```xml
<Handler type="Session" Location="/Session" showAttributeValues="true"/>
```

Je potřeba také vyplnit odpovídající kontakt na správce služby v `supportContact`:

```xml
<Errors supportContact="root@blackhole.cesnet.cz"
    helpLocation="/about.html"
    styleSheet="/shibboleth-sp/main.css"/>
```

Aby fungovalo přihlašování pomocí federace identit, je nezbytné, aby SP znalo metadata jednotlivých poskytovatelů identit:

```xml
<!-- eduID.cz -->
<MetadataProvider type="XML" uri="https://metadata.eduid.cz/entities/eduid+idp"
    backingFilePath="eduid+idp.xml" reloadInterval="600">
    <MetadataFilter type="Signature" certificate="/etc/ssl/certs/metadata.eduid.cz.crt.pem"/>
</MetadataProvider>
```

Jelikož jsme definovali, že se bude kontrolovat integrita federačních metadat, musíme dodat odpovídající certifikát (veřejný klíč):

```bash
wget -P /etc/ssl/certs/ https://www.eduid.cz/docs/eduid/metadata/metadata.eduid.cz.crt.pem
```

Budeme-li využívat dodatečné atributy z _[atributové autority](https://www.eduid.cz/cs/aa)_ (popis [konfigurace](https://www.eduid.cz/cs/tech/aa) je na stránkách [eduID.cz](https://www.eduid.cz/)) pro řízení přístupu k jednotlivým instancím DokuWiki, musí naše služba znát i metadata atributové autority:

```xml
<!-- Attribute authority / Perun -->
<AttributeResolver type="SimpleAggregation" attributeId="eppn"
    format="urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified">
    <Entity>https://aa.cesnet.cz/idp/shibboleth</Entity>
        <MetadataProvider type="XML" uri="https://metadata.eduid.cz/entities/aa.cesnet.cz"
            backingFilePath="aa-cesnet-cz.xml" reloadInterval="600">
        <MetadataFilter type="Signature" certificate="/etc/ssl/certs/metadata.eduid.cz.crt.pem" />
    </MetadataProvider>
</AttributeResolver>
```

Nyní se dostáváme k dodatečným informacím v metadatech služby. Vytvoříme tedy soubor `/etc/shibboleth/metadata-template.xml` s následujícím obsahem, který modifikujeme (zejména `<UIInfo>` a alternativní DNS názvy) dle svých potřeb. Nebudeme zadávat "endpointy" (elementy `<AssertionConsumerService>`) pro `blackhole.cesnet.cz`, ale pouze pro `blackhole1.cesnet.cz` a `blackhole2.cesnet.cz`.

```xml
<?xml version="1.0" encoding="utf-8" ?>
<md:EntityDescriptor xmlns:md="urn:oasis:names:tc:SAML:2.0:metadata">

  <md:SPSSODescriptor>

    <md:Extensions>
      <!-- UIInfo -->
        <mdui:UIInfo xmlns:mdui="urn:oasis:names:tc:SAML:metadata:ui">
        <mdui:DisplayName xml:lang="en">Black Hole</mdui:DisplayName>
        <mdui:DisplayName xml:lang="cs">Černá díra</mdui:DisplayName>
        <mdui:Description xml:lang="en">Black Hole -- testing Shibboleth SP</mdui:Description>
        <mdui:Description xml:lang="cs">Černá díra -- testovací Shibboleth SP</mdui:Description>
        <mdui:InformationURL xml:lang="en">https://blackhole.cesnet.cz/en</mdui:InformationURL>
        <mdui:InformationURL xml:lang="cs">https://blackhole.cesnet.cz/cs</mdui:InformationURL>
        <mdui:Logo height="128" width="128">https://blackhole.cesnet.cz/logo.png</mdui:Logo>
      </mdui:UIInfo>
    </md:Extensions>

    <!-- ------------------------- -->
    <!-- alternative DNS hostnames -->
    <!-- ------------------------- -->

    <!-- blackhole1.cesnet.cz -->
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST" Location="https://blackhole1.cesnet.cz/Shibboleth.sso/SAML2/POST" index="1"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST-SimpleSign" Location="https://blackhole1.cesnet.cz/Shibboleth.sso/SAML2/POST-SimpleSign" index="2"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Artifact" Location="https://blackhole1.cesnet.cz/Shibboleth.sso/SAML2/Artifact" index="3"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:PAOS" Location="https://blackhole1.cesnet.cz/Shibboleth.sso/SAML2/ECP" index="4"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:1.0:profiles:browser-post" Location="https://blackhole1.cesnet.cz/Shibboleth.sso/SAML/POST" index="5"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:1.0:profiles:artifact-01" Location="https://blackhole1.cesnet.cz/Shibboleth.sso/SAML/Artifact" index="6"/>

    <!-- blackhole2.cesnet.cz -->
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST" Location="https://blackhole2.cesnet.cz/Shibboleth.sso/SAML2/POST" index="7"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST-SimpleSign" Location="https://blackhole2.cesnet.cz/Shibboleth.sso/SAML2/POST-SimpleSign" index="8"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Artifact" Location="https://blackhole2.cesnet.cz/Shibboleth.sso/SAML2/Artifact" index="9"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:PAOS" Location="https://blackhole2.cesnet.cz/Shibboleth.sso/SAML2/ECP" index="10"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:1.0:profiles:browser-post" Location="https://blackhole2.cesnet.cz/Shibboleth.sso/SAML/POST" index="11"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:1.0:profiles:artifact-01" Location="https://blackhole2.cesnet.cz/Shibboleth.sso/SAML/Artifact" index="12"/>

  </md:SPSSODescriptor>
 
  <md:Organization>
    <md:OrganizationName xml:lang="en">CESNET, a. l. e.</md:OrganizationName>
    <md:OrganizationName xml:lang="cs">CESNET, z. s. p. o.</md:OrganizationName>
    <md:OrganizationDisplayName xml:lang="en">CESNET</md:OrganizationDisplayName>
    <md:OrganizationDisplayName xml:lang="cs">CESNET</md:OrganizationDisplayName>
    <md:OrganizationURL xml:lang="en">http://www.cesnet.cz/?lang=en</md:OrganizationURL>
    <md:OrganizationURL xml:lang="cs">http://www.cesnet.cz/</md:OrganizationURL>
  </md:Organization>

  <md:ContactPerson contactType="technical">
    <md:GivenName>Jan</md:GivenName>
    <md:SurName>Oppolzer</md:SurName>
    <md:EmailAddress>mailto:jan.oppolzer@cesnet.cz</md:EmailAddress>
  </md:ContactPerson>

</md:EntityDescriptor>
```

V konfiguračním souboru `/etc/shibboleth/attribute-map.xml` musíme odkomentovat několik řádků, abychom získali další uživatelské atributy nad rámec těch definovaných standardně. Jelikož některá IdP by stále nemusela podporovat nový protokol _SAML2_, budeme akceptovat i atributy kódované protokolem _SAML1_. Budeme očekávat následující atributy `cn` (celé jméno), `sn` (příjmení), `givenName` (křestní jméno), `displayName` (celé jméno), `mail` (e-mailová adresa), `telephoneNumber` (telefonní číslo), `o` (organizace), `ou` (organizační jednotka).

```xml
<!-- SAML1 encoded attributes -->
<Attribute name="urn:mace:dir:attribute-def:cn" id="cn"/>
<Attribute name="urn:mace:dir:attribute-def:sn" id="sn"/>
<Attribute name="urn:mace:dir:attribute-def:givenName" id="givenName"/>
<Attribute name="urn:mace:dir:attribute-def:displayName" id="displayName"/>
<Attribute name="urn:mace:dir:attribute-def:mail" id="mail"/>
<Attribute name="urn:mace:dir:attribute-def:telephoneNumber" id="telephoneNumber"/>
<Attribute name="urn:mace:dir:attribute-def:o" id="o"/>
<Attribute name="urn:mace:dir:attribute-def:ou" id="ou"/>

<!-- SAML2 encoded attributes -->
<Attribute name="urn:oid:2.5.4.3" id="cn"/>
<Attribute name="urn:oid:2.5.4.4" id="sn"/>
<Attribute name="urn:oid:2.5.4.42" id="givenName"/>
<Attribute name="urn:oid:2.16.840.1.113730.3.1.241" id="displayName"/>
<Attribute name="urn:oid:0.9.2342.19200300.100.1.3" id="mail"/>
<Attribute name="urn:oid:2.5.4.20" id="telephoneNumber"/>
<Attribute name="urn:oid:2.5.4.10" id="o"/>
<Attribute name="urn:oid:2.5.4.11" id="ou"/>
```

Budeme-li využívat služeb _[atributové autority](https://www.eduid.cz/cs/aa)_ (samostatný popis [konfigurace](https://www.eduid.cz/cs/tech/aa) atributové autority je na stránkách [eduID.cz](https://www.eduid.cz)), jejíž data spravuje systém [Perun](https://perun.cesnet.cz/web), je potřeba v Shibbolethu nakonfigurovat dodatečné atributy, jejichž hodnoty budou plněny právě z atributové autority.

```xml
<!-- Attribute authority / Perun attributes -->
<Attribute name="https://aa.cesnet.cz/attribute-def/perunEntryStatus"      id="perunEntryStatus" />
<Attribute name="https://aa.cesnet.cz/attribute-def/perunGroupDn"          id="perunGroupDn" />
<Attribute name="https://aa.cesnet.cz/attribute-def/perunGroupId"          id="perunGroupId" />
<Attribute name="https://aa.cesnet.cz/attribute-def/perunGroupName"        id="perunGroupName" />
<Attribute name="https://aa.cesnet.cz/attribute-def/perunOrganizationName" id="perunOrganizationName" />
<Attribute name="https://aa.cesnet.cz/attribute-def/perunPreferredMail"    id="perunPreferredMail" />
<Attribute name="https://aa.cesnet.cz/attribute-def/perunPrincipalName"    id="perunPrincipalName" />
<Attribute name="https://aa.cesnet.cz/attribute-def/perunUniqueGroupName"  id="perunUniqueGroupName" />
<Attribute name="https://aa.cesnet.cz/attribute-def/perunUserId"           id="perunUserId" />
<Attribute name="https://aa.cesnet.cz/attribute-def/perunVoId"             id="perunVoId" />
<Attribute name="https://aa.cesnet.cz/attribute-def/perunVoName"           id="perunVoName" />
```

Modul `libapache2-mod-shib2` je po instalaci automaticky aktivován, pokud ne, následující příkaz zařídí jeho nahrání do Apache:

```bash
a2enmod shib2
```

Po provedení úprav výše restartujeme Shibboleth (služba/proces `shibd`):

```bash
systemctl restart shibd
```

Stáhneme si ze služby metadata (URL: `https://blackhole.cesnet.cz/Shibboleth.sso/Metadata`) a zařídíme jejich vložení do federace identit. Zároveň požádáme o přístup k atributové autoritě / Perunovi, abychom mohli získat dodatečné uživatelské atributy. Po přidání metadat do federace identit je potřeba počkat na jejich propagaci. Nebude-li některý z použitých poskytovatelů identit (Identity Provider, IdP) mít metadata naší služby (Service Provider, SP), odmítne s ním komunikovat, takže nebude možné se přihlásit. Propagace metadat může trvat v závislosti na nastavení IdP až 24 hodin, avšak většina rozumně nastavených IdP aktualizuje metadata každé 2 hodiny nebo i častěji.

Vytvoříme si tedy adresáře `shib1/` a `shib2/` a v nich soubor `index.html`, který bude obsahovat text `shib1` anebo `shib2` dle adresáře. Oba adresáře následně zkopírujeme do všech virtuálních hostů v Apachi. Adresáře budou dostupné z rootu webu.

```bash
mkdir shib1 shib2
for i in `seq 1 2`; do echo shib$i >> shib$i/index.html; done
for i in `seq 1 2`; do cp -r shib[12] /var/www/blackhole$i.cesnet.cz; done
mv shib[12] /var/www/blackhole.cesnet.cz/
```

Nyní je potřeba do nastavení jednotlivých virutálních hostů přidat následující řádky, aby se o přístup k adresářům staral Shibboleth. Přístup do adresáře `shib1/` nevyžaduje aktivní sezení v Shibbolethu, adresář `shib2/` už ale ano, a tak budeme v případě neexistujícího sezení nejprve přesměrováni na WAYF, abychom se přihlásili u své domovské organizace -- teprve pak se zobrazí obsah adresáře `shib2/`.

```apache
<Location /shib1>
    AuthType shibboleth
    Require shibboleth
    ShibRequestSetting requireSession 0
</Location>

<Location /shib2>
    AuthType shibboleth
    Require shibboleth
    ShibRequestSetting requireSession 1
</Location>
```

Aby bylo nastavení respektováno, je nutné restartovat Apache:

```bash
systemctl restart apache2
```

Nyní, přistoupíme-li na `https://blackhole.cesnet.cz/shib1/` (případně `https://blackhole1.cesnet.cz/shib1/` nebo `https://blackhole2.cesnet.cz/shib1/`), uvidíme text `shib1`. V tomto adresáři, ač je aktivován Shibboleth, není vyžadováno, aby existovalo sezení. Jedná se o tzv. _přihlášení na vyžádání_ (v angličtině _lazy session_).

Pokud však přistoupíme na `https://blackhole.cesnet.cz/shib2/` (případně `https://blackhole1.cesnet.cz/shib2/` nebo `https://blackhole2.cesnet.cz/shib2/`), budeme přesměrováni na stránku, kde si vybereme svého poskytovatele identity, u něhož se chceme ověřit svým přihlašovacím jménem a heslem. Teprve po úspěšném přihlášení uvidíme na těchto stránkách text `shib2`. (Budeme-li se ověřovat pokaždé u stejné domovské organizace, vytvoříme si poprvé sezení na IdP, takže nebudeme muset v dalších případech zadávat jméno a heslo, budeme-li používat stále stejný webový prohlížeč a budeme-li mít stále cookies z předchozího přihlášení.)

Existenci sezení na námi hostovanách stránkách je možné ověřit na adrese `https://blackhole.cesnet.cz/Shibboleth.sso/Session` (případně `https://blackhole1.cesnet.cz/Shibboleth.sso/Session` nebo `https://blackhole2.cesnet.cz/Shibboleth.sso/Session`). Pokud sezení neexistuje, uvidíme pouze informaci `A valid session was not found.`, v opačném případě uvidíme např. tento výpis:

```
Miscellaneous
Session Expiration (barring inactivity): 479 minute(s)
Client Address: 2001:718:1:6::cafe:1
SSO Protocol: urn:oasis:names:tc:SAML:2.0:protocol
Identity Provider: https://whoami.cesnet.cz/idp/shibboleth
Authentication Time: 2017-02-09T07:45:44.591Z
Authentication Context Class: urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
Authentication Context Decl: (none)

Attributes
affiliation: 2 value(s)
cn: 1 value(s)
displayName: 1 value(s)
entitlement: 3 value(s)
eppn: 1 value(s)
givenName: 1 value(s)
mail: 3 value(s)
o: 1 value(s)
persistent-id: 1 value(s)
perunEntryStatus: 1 value(s)
perunGroupDn: 43 value(s)
perunGroupId: 43 value(s)
perunGroupName: 35 value(s)
perunOrganizationName: 1 value(s)
perunPreferredMail: 1 value(s)
perunPrincipalName: 4 value(s)
perunUniqueGroupName: 43 value(s)
perunUserId: 1 value(s)
perunVoId: 8 value(s)
perunVoName: 8 value(s)
sn: 1 value(s)
```

Pokud jsme si v konfiguraci Shibbolethu povolili zobrazení hodnot atributů, uvidíme i hodnoty, čili např. místo `cn: 1 value(s)` uvidíme `cn: Jan Oppolzer`.

Nyní, pokud vše funguje, můžeme adresáře `shib1/` a `shib2/` ze všech virtuálních hostů bezpečně smazat:

```bash
rm -rf /var/www/blackhole*.cesnet.cz/shib[12]
```

Můžeme také smazat konfigurační části ze všech virtuálních hostů z Apache týkající se těchto adresářů, čili následující bloky kódu:

```apache
<Location /shib1>
    AuthType shibboleth
    Require shibboleth
    ShibRequestSetting requireSession 0
</Location>

<Location /shib2>
    AuthType shibboleth
    Require shibboleth
    ShibRequestSetting requireSession 1
</Location>
```

Pak ještě načteme aktuální konfiguraci Apache:

```bash
systemctl reload apache2
```

## DokuWiki

Jelikož DokuWiki již není součástí balíčkovacího systému Debianu (alespoň ne ve verzi 9 s kódovým označením Stretch), nezbývá než stáhnout si archiv ze [stránek projektu](https://download.dokuwiki.org/).

### Stažení

V době psaní tohoto dokumentu (11. července 2018) je k dispozici [poslední stabilní](https://download.dokuwiki.org/src/dokuwiki/dokuwiki-stable.tgz) (a také doporučená) verze *2018-04-22a "Greebo"*.

```bash
mkdir -p /opt/src
wget -P /opt/src https://download.dokuwiki.org/src/dokuwiki/dokuwiki-stable.tgz
```

### Instalace

Instalaci provedeme jednoduchým rozbalením zdrojových kódů, vytvořením symbolického odkazu `/opt/dokuwiki` směrujícím na právě rozbalenou verzi a změnou práv ke všem souborům v DokuWiki pro uživatele *root* a skupinu *root*, aby Apache (uživatel *www-data*) nemohl modifikovat žádné soubory. Symbolický odkaz nám v budoucnu umožní snadnou aktualizaci, která bude spočívat ve třech krocích: stažení nové verze, rozbalení nové verze a úpravě symbolického odkazu. Zároveň se změnou symbolického odkazu na původní hodnotu budeme moci vrátit zpět k předešlé verzi DokuWiki, pokud by nová verze nefungovala správně.

```bash
tar -xzf /opt/src/dokuwiki-stable.tgz -C /opt
chown -R root:root /opt/dokuwiki-2018-04-22a/
ln -s /opt/dokuwiki-2018-04-22a /opt/dokuwiki
```

Do některých adresářů však naopak Apache (uživatel *www-data*) přístup potřebuje, proto u nich práva změníme:

```bash
chown -R www-data /opt/dokuwiki/data/
chown www-data /opt/dokuwiki/lib/plugins/
chown www-data /opt/dokuwiki/conf/
```

V konfiguraci Apache musíme dále provést následující změny, aby se zobrazovala DokuWiki:

```apache
<VirtualHost *:80>
    ServerName              blackhole.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    Redirect                permanent / https://blackhole.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/error-blackhole.log
    CustomLog               ${APACHE_LOG_DIR}/access-blackhole.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName              blackhole.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    DocumentRoot            /opt/dokuwiki

    ErrorLog                ${APACHE_LOG_DIR}/ssl-error-blackhole.log
    CustomLog               ${APACHE_LOG_DIR}/ssl-access-blackhole.log combined

    SSLEngine               On
    SSLProtocol             All -SSLv2 -SSLv3
    SSLHonorCipherOrder     On
    SSLCompression          Off
    Header                  always set Strict-Transport-Security "max-age=15768000"
    SSLCipherSuite          'EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH:+CAMELLIA256:+AES256:+CAMELLIA128:+AES128:+SSLv3:!aNULL:!eNULL:!LOW:!3DES:!MD5:!EXP:!PSK:!DSS:!RC4:!SEED:!ECDSA:CAMELLIA256-SHA:AES256-SHA:CAMELLIA128-SHA:AES128-SHA'

    SSLCertificateFile      /etc/ssl/certs/blackhole.cesnet.cz.crt.pem
    SSLCertificateKeyFile   /etc/ssl/private/blackhole.cesnet.cz.key.pem
    SSLCertificateChainFile /etc/ssl/certs/chain_TERENA_SSL_CA_3.pem

    <Location />
        AuthType shibboleth
        Require shibboleth
    </Location>

    <Directory /opt/dokuwiki>
        DirectoryIndex doku.php
        Require all granted
        AllowOverride All
              
        AuthType shibboleth
        Require shibboleth
        ShibRequestSetting  requireSession 0    # lazy sessions
        #ShibRequestSetting requireSession 1    # required sessions (only authenticated users)

        Order Deny,Allow
        Allow from all
        Deny from all

        RewriteEngine on
        RewriteBase /
        RewriteRule ^_media/(.*)              lib/exe/fetch.php?media=$1  [QSA,L]
        RewriteRule ^_detail/(.*)             lib/exe/detail.php?media=$1  [QSA,L]
        RewriteRule ^_export/([^/]+)/(.*)     doku.php?do=export_$1&id=$2  [QSA,L]
        RewriteRule ^$                        doku.php  [L]
        RewriteCond %{REQUEST_FILENAME}       !-f
        RewriteCond %{REQUEST_FILENAME}       !-d
        RewriteRule (.*)                      doku.php?id=$1  [QSA,L]
        RewriteRule ^index.php$               doku.php
    </Directory>
</VirtualHost>
```

Aby fungovaly "hezké adresy", které jsme nakonfigurovali výše, musíme povolit modul `rewrite` a restartovat Apache:

```bash
a2enmode rewrite
systemctl restart apache2
```

Nyní si vytvoříme adresář pro jednotlivé instance DokuWiki (v angličtině *animal*) a zároveň si připravíme soubor `preload.php`, kde zapneme tzv. "farming":

```bash
mkdir -p /var/www/farm/
cp /opt/dokuwiki/inc/preload.php.dist /opt/dokuwiki/inc/preload.php
```

V souboru `preload.php`, který jsme si zkopírovali z výchozího distribučního souboru `preload.php.dist`, musíme odkomentovat zakomentované řádky a zakážeme přístup k takzvanému "farmáři" (anglicky *farmer*), což je základní instance DokuWiki. Soubor `preload.php` bude tedy vypadat následujícím způsobem:

```php
<?php
// preload.php

// set this to your farm directory
if(!defined('DOKU_FARMDIR')) define('DOKU_FARMDIR', '/var/www/farm');

// include this after DOKU_FARMDIR if you want to use farms
include(fullpath(dirname(__FILE__)).'/farm.php');

// disable access to the farmer
if(DOKU_FARM == false) { nice_die('Access to the farmer denied'); }
```

### DokuWiki Shibboleth Auth

```bash
apt install git
mkdir -p /opt/src/
cd /opt/src/
git clone https://github.com/JanOppolzer/dokuwiki-shibboleth-auth
cp -r dokuwiki-shibboleth-auth/plugin/authshibboleth/ /opt/dokuwiki/lib/plugins/
```

### Skripty pro "farming"

Přestože existují rozšíření [Farmer Plugin](https://www.dokuwiki.org/plugin:farmer) (kvalitní dokumentace a poměrně hezké grafické rozhraní) nebo [farmsync Plugin](https://www.dokuwiki.org/plugin:farmsync), které "farming" výrazně zjednodušují, my půjdeme jinou cestou. V [dokumentaci](https://www.dokuwiki.org/farms) je velice kvalitně popsán ruční způsob, nad kterým byly v balíčku DokuWiki pro Debian 8 (Jessie) napsány dva skripty -- `dokuwiki-addsite` pro vytvoření nové instance a `dokuwiki-delsite` pro odstranění stávající instance. Tyto dva skripty z Debianu Jessie jsem lehce upravil pro naše použití a jsou k dispozici zde: [dokuwiki-addsite](https://github.com/JanOppolzer/howto/blob/master/dokuwiki-addsite), [dokuwiki-delsite](https://github.com/JanOppolzer/howto/blob/master/dokuwiki-delsite).

### Animal #1

```bash
dokuwiki-addsite blackhole.cesnet.cz
```

### Animal #2

```bash
dokuwiki-addsite blackhole1.cesnet.cz
```

### Animal #3

```bash
dokuwiki-addsite blackhole1.cesnet.cz
```

### Úpravy v DokuWiki

FIXME

#### Přihlašovací formulář

Nechceme, aby se na stránce zobrazoval přihlašovací formulář, pokud nepřihlášený uživatel přistoupí na chráněnou stránku. Přihlašování se má dít pomocí kliknutí na tlačítko "Přihlásit" v záhlaví (linkeru), výběru domovské organizace z WAYFu a následně vyplněním přihlašovacích údajů na stránce domovské organizace. Toho docílíme triviální úpravou, kdy zakomentujeme na příslušném místě řádek s funkcí `html_login()` a vypíšeme informaci *"Možná jste se zapomněl(a) přihlásit.* Úpravu provedeme v souboru `/var/lib/dokuwiki/inc/html.php` ve funkci `html_denied()` na řádcích 86 a 87 takto:

```php
82 function html_denied() {
83     print p_locale_xhtml('denied');
84
85     if(empty($_SERVER['REMOTE_USER'])){
86         //html_login();
87         print "<p>Možná jste se zapomněl(a) přihlásit.</p>";
88     }
89 }
```

#### Rozšíření XYZ

FIXME

### Migrace stávajících DokuWiki instancí

Chceme-li [přenést](https://www.dokuwiki.org/faq:servermove) stávající DokuWiki do námi právě vytvořené "farmy", stačí přenést **celý datový adresář** (na Debianu 8/Jessie se jedná o adresář `/var/lib/dokuwiki/data/`) ze starého serveru na nový. Musíme však pamatovat na správné umístění na novém serveru s farmou, tedy `/var/www/farm/<jméno_dokuwiki_instance>/data/`, přičemž novou instanci si nejprve vytvoříme příkazem `dokuwiki-addsite <jméno_dokuwiki_instance>`.

Aby nedošlo ke změně posledních úprav u jednotlivých souborů, je vhodné na starém serveru *datový adresář* zabalit do TAR archivu a po přesunu na nový server archiv rozbalit. Je třeba též dodržet správná přístupová práva.

Musíme také samozřejmě vytvořit odpovídající virtuální host v Apachi, zažádat si o nový SSL certifikát s dodatečnou doménou pro novou instanci DokuWiki a zařídit změnu DNS záznamů. Pak už opravdu stačí jen příkazem `dokuwiki-addsite` vytvořit tuto novou instanci a překopírovat datový adresář. Následně ještě musíme provést potřebné konfigurační úpravy DokuWiki nové instance a zkontrolovat všechna potřebná rozšíření.

