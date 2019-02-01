# DokuWiki Farming s Apache, Shibboleth SP a ProxyIdP na Debianu 9 (Stretch)

Tento návod navazuje na [předchozí variantu pro Debian 9 (Stretch)](https://github.com/JanOppolzer/howto/blob/master/debian-9-stretch-dokuwiki-farming-apache-shibboleth-sp.md), která využívala [Atributovou autoritu](https://www.eduid.cz/cs/aa), avšak liší se v tom, že nyní využívá ProxyIdP.

## Úvod

V případě, že chceme provozovat několik různých DokuWiki instancí pro různé projekty, není potřeba mít pro každou DokuWiki vlastní [virtuální nebo fyzický] server a starat se tak o několik operačních systémů. Je možné použít tzv. [DokuWiki Farming](https://www.dokuwiki.org/farms).

Všechny instance sdílí jednu instalaci DokuWiki a také rozšíření. Mají však své vlastní stránky a také nastavení (přístupová práva, vzhled, použitá rozšíření, způsob přihlašování, ...).

Tento dokument počítá s tím, že všechny instance budou používat přihlašování přes ProxyIdP. Přihlašování je implementováno pomocí rozšíření [dokuwiki-shibboleth-auth](https://github.com/JanOppolzer/dokuwiki-shibboleth-auth). Zároveň hostující server bude z pohledu federace jeden poskytovatel služby neboli jedna služba (Service Provider, SP) s jedním *entityID*. Jednotlivé instance DokuWiki běžící na svých vlastních adresách (doménách) budou mít své "koncové body" (endpointy) uvedeny v metadatech jediné entity. V návodu je použit CESNETí "linker" (horní záhlaví/hlavička) a [jednotný vzhled pro DokuWiki](https://wiki.cesnet.cz/doku.php?id=groups:weby:dokuwiki:start).

Metadata služby musí být zaregistrována v ProxyIdP. Registrace a jakékoliv změny metadat se provádí zasláním metadat na adresu <login@cesnet.cz>. **Metadata služby by NEMĚLA být ve federaci [eduID.cz](http://eduID.cz) a vřele doporučuji, abyste si zkontrolovali, že tam metadata nemáte!**

*Pokud by nám nevyhovovalo, že všechny instance budou "schovány" pod jedním `entityID`, je možné nakonfigurovat Shibboleth SP tak, aby každá instance měla svá vlastní metadata, své vlastní `entityID` a podobně. Po každém přidání takové instance DokuWiki je pak ale potřeba udělat ruční konfigurační změny v `/etc/shibboleth/shibboleth2.xml`, které však nejsou nijak složité. Tato varianta zde NENÍ zdokumentovaná. V případě potřeby se ozvěte na e-mail <jan.oppolzer@cesnet.cz>.*

Níže popsaný přístup sníží nároky jak na správu operačních systémů (bude nutné spravovat jen jeden operační systém), tak instalace DokuWiki i federativní autentizace. Jeden operační systém, jedna instalace DokuWiki, jeden Shibboleth SP s jedním certifikátem atd. Budeme-li mít certifikát pro webserver obsahující v alternativních jménech všechny potřebné domény (doporučený způsob), budeme mít i jeden SSL certifikát. Znamená to ovšem, že když vyprší SSL certifikát, vyprší pro všechny instance najednou (ano, vyplatí se monitorovat). Rozbije-li se DokuWiki např. při neopatrné aktualizaci, rozbije to všechny instance najednou. (*With great power comes great responsibility.*)

V krocích níže se předpokládá použití [Debianu 9](https://www.debian.org/releases/stretch/) s kódovým označením [Stretch](https://www.debian.org/releases/stretch/).

Jako první nainstalujeme a nakonfigurujeme Apache a zprovozníme si virtuální hosty včetně SSL konfigurace. Následně nainstalujeme a nakonfigurujeme Shibboleth SP a zprovozníme federativní přihlašování. Poté bude na řadě instalace a konfigurace DokuWiki a její nakonfigurování v Apachi. Posledním krokem, který bude v řadě případů nepotřebný, bude návod, jak převést stávající DokuWiki instance na novou instalaci — to pro případ, že byste konsolidovali více existujících instancí DokuWiki na několika serverech na jeden server.

## Konvence

V návodu používám následující:

* DNS název fyzického serveru (hostname): `devnull.cesnet.cz`
* entityID služby: `https://devnull.cesnet.cz/shibboleth`
* hostované weby/domény: `devnull.cesnet.cz`, `devnull1.cesnet.cz`, `devnull2.cesnet.cz`
* SSL certifikát se všemi hostovanými doménami: `/etc/ssl/certs/devnull.cesnet.cz.crt.pem`
* privátní klíč k SSL certifikátu: `/etc/ssl/private/devnull.cesnet.cz.key.pem`
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

Instalaci Apache provedeme obvyklým způsobem pomocí příkazu `apt`:

```bash
apt install apache2
```

Nejprve nakonfigurujeme server `devnull.cesnet.cz`. Vytvoříme konfigurační soubor `/etc/apache2/sites-available/devnull.cesnet.cz.conf` s následujícím obsahem:

```apache
<VirtualHost *:80>
    ServerName              devnull.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    Redirect                permanent / https://devnull.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/error-devnull.log
    CustomLog               ${APACHE_LOG_DIR}/access-devnull.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName              devnull.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    DocumentRoot            /var/www/devnull.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/ssl-error-devnull.log
    CustomLog               ${APACHE_LOG_DIR}/ssl-access-devnull.log combined

    SSLEngine               On
    SSLProtocol             All -SSLv2 -SSLv3
    SSLHonorCipherOrder     On
    SSLCompression          Off
    Header                  always set Strict-Transport-Security "max-age=15768000"
    Header                  always set X-Content-Type-Options    "nosniff"
    Header                  always set X-Xss-Protection          "1; mode=block"
    Header                  always set X-Frame-Options           "DENY"
    Header                  always set Referrer-Policy           "no-referrer-when-downgrade"
    SSLCipherSuite          'EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH:+CAMELLIA256:+AES256:+CAMELLIA128:+AES128:+SSLv3:!aNULL:!eNULL:!LOW:!3DES:!MD5:!EXP:!PSK:!DSS:!RC4:!SEED:!ECDSA:CAMELLIA256-SHA:AES256-SHA:CAMELLIA128-SHA:AES128-SHA'

    SSLCertificateFile      /etc/ssl/certs/devnull.cesnet.cz.crt.pem
    SSLCertificateKeyFile   /etc/ssl/private/devnull.cesnet.cz.key.pem
    SSLCertificateChainFile /etc/ssl/certs/chain_TERENA_SSL_CA_3.pem
</VirtualHost>
```

*Bylo by vhodné nastudovat si "HTTP Content-Security-Policy" hlavičku a odpovídajícím způsobem ji v Apachi nastavit. Nicméně to je nad rámec tohoto dokumentu.*

Vytvoříme adresář `/var/www/devnull.cesnet.cz/` a v něm soubor `index.html` obsahující pouze text `devnull.cesnet.cz`.

```bash
mkdir -p /var/www/devnull.cesnet.cz/
echo "devnull.cesnet.cz" > /var/www/devnull.cesnet.cz/index.html
```

SSL certifikát umístíme do souboru `/etc/ssl/certs/devnull.cesnet.cz.crt.pem`, privátní klíč do `/etc/ssl/private/devnull.cesnet.cz.key.pem` a řetěz certifikátů až ke kořenové CA do `/etc/ssl/certs/chain_TERENA_SSL_CA_3.pem`.

Nyní v konfiguraci Apache vypneme server `000-default.conf` (`/etc/apache2/sites-available/000-default.conf`), který je po instalaci Apache ve výchozím stavu zapnutý. Následně zapneme nově nadefinovaný server `devnull.cesnet.cz.conf`.

Jelikož v konfiguraci jednotlivých serverů používáme *SSL certifikáty* pro komunikaci přes *HTTPS*, je potřeba aktivovat modul `ssl` v Apachi. Navíc u *SSL* nastavujeme *HSTS* a jiné hlavičky, takže je potřeba aktivovat i modul `headers`. Pak již zbývá jen restartovat Apache.

```bash
a2dissite 000-default.conf
a2ensite devnull.cesnet.cz.conf
a2enmod ssl headers
systemctl restart apache2
```

Vyzkoušíme se připojit webovým prohlížečem k serveru a měli bychom vidět text `devnull.cesnet.cz`. Nyní můžeme přejít ke konfiguraci zbývajících virtuálních hostů `devnull1.cesnet.cz` a `devnull2.cesnet.cz`. Konfigurační soubory (`devnull1.cesnet.cz.conf`, `devnull2.cesnet.cz.conf`) budou velice podobné výše uvedenému souboru `devnull.cesnet.cz.conf`. Změny budou jen v parametrech `ServerName`, `DocumentRoot`, `Redirect`, `ErrorLog` a `CustomLog`.

Konfigurační soubor `/etc/apache2/sites-available/devnull1.cesnet.cz.conf`:

```apache
<VirtualHost *:80>
    ServerName              devnull1.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    Redirect                permanent / https://devnull1.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/error-devnull1.log
    CustomLog               ${APACHE_LOG_DIR}/access-devnull1.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName              devnull1.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    DocumentRoot            /var/www/devnull1.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/ssl-error-devnull1.log
    CustomLog               ${APACHE_LOG_DIR}/ssl-access-devnull1.log combined

    SSLEngine               On
    SSLProtocol             All -SSLv2 -SSLv3
    SSLHonorCipherOrder     On
    SSLCompression          Off
    Header                  always set Strict-Transport-Security "max-age=15768000"
    Header                  always set X-Content-Type-Options    "nosniff"
    Header                  always set X-Xss-Protection          "1; mode=block"
    Header                  always set X-Frame-Options           "DENY"
    Header                  always set Referrer-Policy           "no-referrer-when-downgrade"
    SSLCipherSuite          'EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH:+CAMELLIA256:+AES256:+CAMELLIA128

    SSLCertificateFile      /etc/ssl/certs/devnull.cesnet.cz.crt.pem
    SSLCertificateKeyFile   /etc/ssl/private/devnull.cesnet.cz.key.pem
    SSLCertificateChainFile /etc/ssl/certs/chain_TERENA_SSL_CA_3.pem
</VirtualHost>
```

Konfigurační soubor `/etc/apache2/sites-available/devnull2.cesnet.cz.conf`:

```apache
<VirtualHost *:80>
    ServerName              devnull2.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    Redirect                permanent / https://devnull2.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/error-devnull2.log
    CustomLog               ${APACHE_LOG_DIR}/access-devnull2.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName              devnull2.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    DocumentRoot            /var/www/devnull2.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/ssl-error-devnull2.log
    CustomLog               ${APACHE_LOG_DIR}/ssl-access-devnull2.log combined

    SSLEngine               On
    SSLProtocol             All -SSLv2 -SSLv3
    SSLHonorCipherOrder     On
    SSLCompression          Off
    Header                  always set Strict-Transport-Security "max-age=15768000"
    Header                  always set X-Content-Type-Options    "nosniff"
    Header                  always set X-Xss-Protection          "1; mode=block"
    Header                  always set X-Frame-Options           "DENY"
    Header                  always set Referrer-Policy           "no-referrer-when-downgrade"
    SSLCipherSuite          'EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH:+CAMELLIA256:+AES256:+CAMELLIA128

    SSLCertificateFile      /etc/ssl/certs/devnull.cesnet.cz.crt.pem
    SSLCertificateKeyFile   /etc/ssl/private/devnull.cesnet.cz.key.pem
    SSLCertificateChainFile /etc/ssl/certs/chain_TERENA_SSL_CA_3.pem
</VirtualHost>
```

Vytvoříme si nové adresáře pro zbývající weby a v každém z nich vytvoříme soubor `index.html` s textem odpovídajícím názvu serveru pro daný web:

```bash
mkdir -p /var/www/devnull{1,2}.cesnet.cz/
for i in `seq 1 2`; do echo "devnull$i.cesnet.cz" >> /var/www/devnull$i.cesnet.cz/index.html; done
```

Povolíme nově definované virtuální hosty a znovu nahrajeme (stačí `reload`, není třeba `restart`) konfiguraci Apache:

```bash
a2ensite devnull1.cesnet.cz.conf devnull2.cesnet.cz.conf
systemctl reload apache2
```

Nyní ověříme, že se dostaneme na všechny weby a to jak protokolem *HTTP* (musí dojít k přesměrování), tak protokolem *HTTPS*: `devnull.cesnet.cz`, `devnull1.cesnet.cz` i `devnull2.cesnet.cz`.

### PHP

Pro běh *DokuWiki* budeme potřebovat interpret jazyku *PHP* (s podporou *GD* a *XML*), ten nainstalujeme standardním způsobem z balíčkovacího systému.

```bash
apt install php php-gd php-xml
```

Že vše v pořádku funguje, vyzkoušíme triviálním skriptem obsahujícím pouze `<?php PHPInfo(); ?>`.

```bash
cat >> /var/www/devnull.cesnet.cz/phpinfo.php << EOF
<?php PHPInfo(); ?>
EOF
```

Nyní se přesvědčíme, že PHP funguje. Podíváme se na stránku `https://devnull.cesnet.cz/phpinfo.php` a pokud je vše v pořádku, můžeme skript smazat a pokračovat v dalších krocích.

```bash
rm /var/www/devnull.cesnet.cz/phpinfo.php
```

## Shibboleth Service Provider

Provozovat Shibboleth SP ve spolupráci s webovým serverem Apache je velice jednoduché, stačí doinstalovat modul do Apache, povolit ho a upravit nastavení virtuálních hostů, kde má být modul použit:

Instalace modulu Shibboleth SP do Apache:

```bash
apt install libapache2-mod-shib2
```

Nyní se můžeme pustit do konfigurace Shibboleth SP. Po instalaci výše uvedeného balíčku si vygenerujeme certifikát pro Shibboleth, který bude v metadatech entity. Následující příkaz vygeneruje self-signed certifikát, nastaví mu vhodná práva i jméno, abychom měli co nejsnazší nastavení. Tento certifikát bude mít platnost 10 let (pomocí parametru `-y` je možné ovlivnit jeho platnost v letech), bude vystaven pro server/hostname [devnull.cesnet.cz](http://devnull.cesnet.cz) (parametr `-h`) s *entityID* `https://devnull.cesnet.cz/shibboleth` (parametr `-e`).

```bash
shib-keygen -h devnull.cesnet.cz -e https://devnull.cesnet.cz/shibboleth
```

Nejdůležitější část konfigurace Shibboleth SP se odehrává v konfiguračním souboru `/etc/shibboleth/shibboleth2.xml`. Standardně se po přihlášení uživatele do proměnné prostředí `REMOTE_USER` dostávají atributy v tomto pořadí: _eduPersonPrincipalName_ (`eppn`) a _eduPersonTargetedID_ (`persistent-id`). Chceme-li mezi nimi i _eduPersonUniqueId_ (`uniqueid`) anebo forwardované _eduPersonPrincipalName_ (`forwardedPrincipalName`), musíme si je doplnit do atributu `REMOTE_USER` elementu `<ApplicationDefaults>`. _Bude použita první existující hodnota_:
```xml
<ApplicationDefaults entityID="https://devnull.cesnet.cz/shibboleth"
                     REMOTE_USER="eppn uniqueid forwardedPersonPrincipalName persistent-id targeted-id"
                     cipherSuites="ECDHE+AESGCM:ECDHE:!aNULL:!eNULL:!LOW:!EXPORT:!RC4:!SHA:!SSLv2">
```

Dále nakonfigurujeme SSL (změníme `handlerSSL` na *true* a `cookieProps` na *https*):

```xml
<Sessions lifetime="28800" timeout="3600" relayState="ss:mem"
          checkAddress="false" handlerSSL="true" cookieProps="https">

```

Konfigurace pro potřeby ProxyIdP je poněkud složitější, pokud chceme, aby ProxyIdP při zobrazení WAYFu používala filtry, které si definujeme v konfiguračním souboru `shibboleth2.xml`. Pokud žádný filtr nedefinujeme, zobrazí se všechna IdP z federací [eduID.cz](http://eduID.cz) a eduGAIN.

Nejprve zakomentujeme (případně smažeme) elementy `<SSO>` a `<Logout>`, jejichž existence by jinak způsobila nefunkčnost konfigurace po přidání níže uvedených řádků:

```xml
<!--
<SSO entityID="https://idp.example.org/idp/shibboleth"
    discoveryProtocol="SAMLDS" discoveryURL="https://ds.example.org/DS/WAYF">
    SAML2 SAML1
</SSO>
-->

<!--
<Logout>SAML2 Local</Logout>
-->
```

Dále si uvnitř elementu `<Sessions>` nadefinujeme `<SessionInitiator>`, v němž nastavíme `entityID` na `https://login.cesnet.cz/idp/` a `authnContextClassRef` na hodnotu filtru pro WAYF, který si vygenerujeme na stránce [generátoru filtru](https://ds.eduid.cz/filter.php). Nesmíme také zapomenout na elementy `<AssertionConsumerService>`:

```xml
<SessionInitiator Location="/Login" isDefault="true" id="Login"
    type="SAML2" template="bindingTemplate.html"
    entityID="https://login.cesnet.cz/idp/"
    authnContextClassRef="urn:cesnet:proxyidp:filter:eyJ2ZXIiOiIyIiwiYWxsb3dGZWVkcyI6eyJlZHVJRC5jeiI6e319LCJhbGxvd0hvc3RlbCI6dHJ1ZSwiYWxsb3dIb3N0ZWxSZWciOmZhbHNlfQ==" />

<md:AssertionConsumerService Location="/SAML2/POST" index="1"
    Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"/>
<md:AssertionConsumerService Location="/SAML2/POST-SimpleSign" index="2"
    Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST-SimpleSign"/>
<md:AssertionConsumerService Location="/SAML2/Artifact" index="3"
    Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Artifact"/>
```

Metadata služby musíme obohatit o další informace — přidáme atribut `template="metadata-template.xml"` a souboru `metadata-template.xml` se budeme věnovat níže:

```xml
<Handler type="MetadataGenerator" Location="/Metadata" signing="false" template="metadata-template.xml"/>
```

Můžeme si povolit zobrazení hodnot atributů na diagnostrické stránce (`https://devnull.cesnet.cz/Shibboleth.sso/Session`):

```xml
<Handler type="Session" Location="/Session" showAttributeValues="true"/>
```

Je potřeba také vyplnit odpovídající kontakt na správce služby v `supportContact`:

```xml
<Errors supportContact="jan.oppolzer@cesnet.cz"
    helpLocation="/about.html"
    styleSheet="/shibboleth-sp/main.css"/>
```

Aby fungovalo přihlašování přes ProxyIdP, je nutné stahovat metadata ProxyIdP. **Ne, metadata federace eduID.cz nepotřebujete, takže je nestahujte.**

```xml
<!-- ProxyIdP -->
<MetadataProvider type="XML" uri="https://login.cesnet.cz/proxy/saml2/idp/metadata.php"
    backingFilePath="proxyidp.xml" reloadInterval="600">
    <MetadataFilter type="Signature" certificate="/etc/ssl/certs/proxyidp.crt.pem"/>
</MetadataProvider>
```

Jelikož jsme definovali, že se bude kontrolovat integrita federačních metadat, musíme dodat odpovídající certifikát (veřejný klíč):

```bash
wget -O /etc/ssl/certs/proxyidp.crt.pem https://login.cesnet.cz/proxy/module.php/saml/idp/certs.php/idp.crt
```

Nyní se dostáváme k dodatečným informacím v metadatech služby. Vytvoříme tedy soubor `/etc/shibboleth/metadata-template.xml` s následujícím obsahem, který modifikujeme (zejména `<UIInfo>` a alternativní DNS názvy) dle svých potřeb. Nebudeme zadávat "endpointy" (elementy `<AssertionConsumerService>`) pro `devnull.cesnet.cz`, ale pouze pro `devnull1.cesnet.cz` a `devnull2.cesnet.cz`.

```xml
<?xml version="1.0" encoding="utf-8" ?>
<md:EntityDescriptor xmlns:md="urn:oasis:names:tc:SAML:2.0:metadata">

  <md:SPSSODescriptor>

    <md:Extensions>
      <!-- UIInfo -->
        <mdui:UIInfo xmlns:mdui="urn:oasis:names:tc:SAML:metadata:ui">
        <mdui:DisplayName xml:lang="en">DevNull</mdui:DisplayName>
        <mdui:DisplayName xml:lang="cs">DevNull</mdui:DisplayName>
        <mdui:Description xml:lang="en">DevNull -- testing Shibboleth SP</mdui:Description>
        <mdui:Description xml:lang="cs">DevNull -- testovací Shibboleth SP</mdui:Description>
        <mdui:InformationURL xml:lang="en">https://devnull.cesnet.cz/en</mdui:InformationURL>
        <mdui:InformationURL xml:lang="cs">https://devnull.cesnet.cz/cs</mdui:InformationURL>
        <mdui:Logo height="128" width="128">https://devnull.cesnet.cz/logo.png</mdui:Logo>
      </mdui:UIInfo>
    </md:Extensions>

    <!-- ------------------------- -->
    <!-- alternative DNS hostnames -->
    <!-- ------------------------- -->

    <!-- devnull1.cesnet.cz -->
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST" Location="https://devnull1.cesnet.cz/Shibboleth.sso/SAML2/POST" index="1"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST-SimpleSign" Location="https://devnull1.cesnet.cz/Shibboleth.sso/SAML2/POST-SimpleSign" index="2"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Artifact" Location="https://devnull1.cesnet.cz/Shibboleth.sso/SAML2/Artifact" index="3"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:PAOS" Location="https://devnull1.cesnet.cz/Shibboleth.sso/SAML2/ECP" index="4"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:1.0:profiles:browser-post" Location="https://devnull1.cesnet.cz/Shibboleth.sso/SAML/POST" index="5"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:1.0:profiles:artifact-01" Location="https://devnull1.cesnet.cz/Shibboleth.sso/SAML/Artifact" index="6"/>

    <!-- devnull2.cesnet.cz -->
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST" Location="https://devnull2.cesnet.cz/Shibboleth.sso/SAML2/POST" index="7"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST-SimpleSign" Location="https://devnull2.cesnet.cz/Shibboleth.sso/SAML2/POST-SimpleSign" index="8"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Artifact" Location="https://devnull2.cesnet.cz/Shibboleth.sso/SAML2/Artifact" index="9"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:PAOS" Location="https://devnull2.cesnet.cz/Shibboleth.sso/SAML2/ECP" index="10"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:1.0:profiles:browser-post" Location="https://devnull2.cesnet.cz/Shibboleth.sso/SAML/POST" index="11"/>
    <md:AssertionConsumerService Binding="urn:oasis:names:tc:SAML:1.0:profiles:artifact-01" Location="https://devnull2.cesnet.cz/Shibboleth.sso/SAML/Artifact" index="12"/>

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

V konfiguračním souboru `/etc/shibboleth/attribute-map.xml` musíme odkomentovat několik řádků, abychom získali další uživatelské atributy nad rámec těch definovaných standardně. Budeme očekávat následující atributy `cn` (celé jméno), `displayName` (zobrazované jméno), `mail` (e-mailová adresa) a `eduPersonUniqueId` (neměnný a nerecyklovatelný uživatelský identifikátor). _Stačí SAML2 enkódování._

Dále potřebujeme definovat ještě několik atributů specifických pro ProxyIdP, čili `loa` (úroveň ověření účtu), `isCesnetEligibleLastSeen`, `forwardedPersonPrincipalName` (přeposlané hodnoty atributu `eduPersonPrincipalName` od IdP, kterým se uživatel přihlásil), `forwardedScopedAffiliation` (přeposlané hodnoty atributu `eduPersonScopedAffiliation` od IdP, kterým se uživatel přihlásil), `sourceIdPEntityID` (`entityID` IdP, kterým se uživatel přihlásil) a `IdPOrganizationName` (hodnota atributu `o` od IdP, kterým se uživatel přihlásil).

```xml
<!-- SAML2 encoded attributes -->
<Attribute name="urn:oid:2.5.4.3" id="cn"/>
<Attribute name="urn:oid:2.16.840.1.113730.3.1.241" id="displayName"/>
<Attribute name="urn:oid:0.9.2342.19200300.100.1.3" id="mail"/>

<!-- ProxyIdP -->
<Attribute name="urn:oid:1.3.6.1.4.1.5923.1.1.1.13"                             id="uniqueid" />
<Attribute name="urn:oid:1.3.6.1.4.1.8057.2.1"                                  id="loa" />
<Attribute name="urn:cesnet:proxyidp:attribute:isCesnetEligibleLastSeen"        id="isCesnetEligibleLastSeen" />
<Attribute name="urn:cesnet:proxyidp:attribute:forwardedPersonPrincipalName"    id="forwardedPersonPrincipalName" />
<Attribute name="urn:cesnet:proxyidp:attribute:forwardedScopedAffiliation"      id="forwardedScopedAffiliation" />
<Attribute name="urn:cesnet:proxyidp:attribute:sourceIdPEntityID"               id="sourceIdPEntityID" />
<Attribute name="urn:cesnet:proxyidp:attribute:IdPOrganizationName"             id="IdPOrganizationName" />
```

Modul `libapache2-mod-shib2` je po instalaci automaticky aktivován, pokud ne, následující příkaz zařídí jeho nahrání do Apache:

```bash
a2enmod shib2
```

Po provedení úprav výše restartujeme Shibboleth (služba/proces `shibd`):

```bash
systemctl restart shibd
```

Stáhneme si ze služby metadata (URL: `https://devnull.cesnet.cz/Shibboleth.sso/Metadata`) a zařídíme jejich vložení do ProxyIdP jejich zasláním na adresu login@cesnet.cz.

Po přidání metadat do ProxyIdP je potřeba počkat na jejich propagaci. Nebude-li mít ProxyIdP metadata naší služby (Service Provider, SP), odmítne s ní komunikovat, takže nebude možné se přihlásit. Propagace metadat může trvat až několik hodin.

Vytvoříme si tedy adresáře `shib1/` a `shib2/` a v nich soubor `index.html`, který bude obsahovat text `shib1` anebo `shib2` dle adresáře. Oba adresáře následně zkopírujeme do všech virtuálních hostů v Apachi. Adresáře budou dostupné z rootu webu.

```bash
mkdir shib1 shib2
for i in `seq 1 2`; do echo shib$i >> shib$i/index.html; done
for i in `seq 1 2`; do cp -r shib[12] /var/www/devnull$i.cesnet.cz; done
mv shib[12] /var/www/devnull.cesnet.cz/
```

Nyní je potřeba do nastavení jednotlivých virutálních hostů přidat následující řádky, aby se o přístup k adresářům staral Shibboleth. Přístup do adresáře `shib1/` nevyžaduje aktivní sezení v Shibbolethu, adresář `shib2/` už ale ano, a tak budeme v případě neexistujícího sezení nejprve přesměrováni na WAYF, abychom se přihlásili u své domovské organizace — teprve pak se zobrazí obsah adresáře `shib2/`.

```apache
<VirtualHost *:443>

    ServerName              devnull.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    #
    # ... Apache configuration ...
    #

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

</VirtualHost>
```

Výše uvedenou konfiguraci vložíme do konfiguračních souborů všech virtuálních hostů. Aby bylo nastavení respektováno, je nutné restartovat Apache:

```bash
systemctl restart apache2
```

Nyní, přistoupíme-li na `https://devnull.cesnet.cz/shib1/` (případně `https://devnull1.cesnet.cz/shib1/` nebo `https://devnull2.cesnet.cz/shib1/`), uvidíme text `shib1`. V tomto adresáři, ač je aktivován Shibboleth, není vyžadováno, aby existovalo sezení. Jedná se o tzv. *přihlášení na vyžádání* (v angličtině *lazy session*).

Pokud však přistoupíme na `https://devnull.cesnet.cz/shib2/` (případně `https://devnull1.cesnet.cz/shib2/` nebo `https://devnull2.cesnet.cz/shib2/`), budeme přesměrováni na stránku, kde si vybereme svého poskytovatele identity, u něhož se chceme ověřit svým přihlašovacím jménem a heslem. Teprve po úspěšném přihlášení uvidíme na těchto stránkách text `shib2`. (Budeme-li se ověřovat pokaždé u stejné domovské organizace, vytvoříme si poprvé sezení na IdP, takže nebudeme muset v dalších případech zadávat jméno a heslo, budeme-li používat stále stejný webový prohlížeč a budeme-li mít stále cookies z předchozího přihlášení.)

Existenci sezení na námi hostovanách stránkách je možné ověřit na adrese `https://devnull.cesnet.cz/Shibboleth.sso/Session` (případně `https://devnull1.cesnet.cz/Shibboleth.sso/Session` nebo `https://devnull2.cesnet.cz/Shibboleth.sso/Session`). Pokud sezení neexistuje, uvidíme pouze informaci `A valid session was not found.`, v opačném případě uvidíme např. tento výpis:

```
Miscellaneous
Session Expiration (barring inactivity): 479 minute(s)
Client Address: 2001:718:1:13:1c1f:797e:9774:911b
SSO Protocol: urn:oasis:names:tc:SAML:2.0:protocol
Identity Provider: https://login.cesnet.cz/idp/
Authentication Time: 2018-11-09T12:02:35Z
Authentication Context Class: urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport
Authentication Context Decl: (none)

Attributes
IdPOrganizationName: 1 value(s)
affiliation: 1 value(s)
cn: 1 value(s)
displayName: 1 value(s)
entitlement: 5 value(s)
eppn: 1 value(s)
forwardedPersonPrincipalName: 6 value(s)
isCesnetEligibleLastSeen: 1 value(s)
loa: 1 value(s)
mail: 1 value(s)
persistent-id: 1 value(s)
sourceIdPEntityID: 1 value(s)
uniqueid: 1 value(s)
```

Pokud jsme si v konfiguraci Shibbolethu povolili zobrazení hodnot atributů, uvidíme i hodnoty, čili např. místo `cn: 1 value(s)` uvidíme `cn: Jan Oppolzer`.

Nyní, pokud vše funguje, můžeme adresáře `shib1/` a `shib2/` ze všech virtuálních hostů smazat:

```bash
rm -rf /var/www/devnull*.cesnet.cz/shib[12]
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

Pak ještě restartujeme Apache:

```bash
systemctl restart apache2
```

## DokuWiki

Jelikož DokuWiki již není součástí balíčkovacího systému Debianu (alespoň ne ve verzi 9 s kódovým označením Stretch), nezbývá než stáhnout si archiv ze [stránek projektu](https://download.dokuwiki.org/).

### Stažení

V době psaní tohoto dokumentu (9. listopadu 2018) je k dispozici [poslední stabilní](https://download.dokuwiki.org/src/dokuwiki/dokuwiki-stable.tgz) (a také doporučená) verze *2018-04-22a "Greebo"*.

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
    ServerName              devnull.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    Redirect                permanent / https://devnull.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/error-devnull.log
    CustomLog               ${APACHE_LOG_DIR}/access-devnull.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName              devnull.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    DocumentRoot            /opt/dokuwiki

    ErrorLog                ${APACHE_LOG_DIR}/ssl-error-devnull.log
    CustomLog               ${APACHE_LOG_DIR}/ssl-access-devnull.log combined

    SSLEngine               On
    SSLProtocol             All -SSLv2 -SSLv3
    SSLHonorCipherOrder     On
    SSLCompression          Off
    Header                  always set Strict-Transport-Security "max-age=15768000"
    Header                  always set X-Content-Type-Options    "nosniff"
    Header                  always set X-Xss-Protection          "1; mode=block"
    Header                  always set X-Frame-Options           "DENY"
    Header                  always set Referrer-Policy           "no-referrer-when-downgrade"
    SSLCipherSuite          'EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH:+CAMELLIA256:+AES256:+CAMELLIA128:+AES128:+SSLv3:!aNULL:!eNULL:!LOW:!3DES:!MD5:!EXP:!PSK:!DSS:!RC4:!SEED:!ECDSA:CAMELLIA256-SHA:AES256-SHA:CAMELLIA128-SHA:AES128-SHA'

    SSLCertificateFile      /etc/ssl/certs/devnull.cesnet.cz.crt.pem
    SSLCertificateKeyFile   /etc/ssl/private/devnull.cesnet.cz.key.pem
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
        # lazy sessions
        ShibRequestSetting  requireSession 0
        # required sessions (only authenticated users)
        #ShibRequestSetting requireSession 1

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

Aby fungovaly "hezké adresy", které jsme nakonfigurovali výše, musíme aktivovat modul `rewrite` a restartovat Apache:

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

Zkusíme-li teď přistoupit k `https://devnull.cesnet.cz`, objeví se informace *DokuWiki Setup Error Access to the farmer denied*. To je tím, že jsme zakázali přístup k základní instanci. Chceme totiž, aby všechna naše data byla v `/var/www/farm/` a nic se nenacházelo přímo v DokuWiki, tedy `/opt/dokuwiki/`.

### Skripty pro "farming"

Přestože existují rozšíření jako [Farmer Plugin](https://www.dokuwiki.org/plugin:farmer) (kvalitní dokumentace a poměrně hezké grafické rozhraní) nebo [farmsync Plugin](https://www.dokuwiki.org/plugin:farmsync), které "farming" výrazně zjednodušují, my půjdeme jinou cestou — spoléhat se na to, že tato rozšíření budou fungovat i za pár let, je velice rikantní. V [dokumentaci](https://www.dokuwiki.org/farms) je velice kvalitně popsán ruční způsob, nad kterým byly v balíčku DokuWiki pro Debian 8 (Jessie) napsány dva skripty — `dokuwiki-addsite` pro vytvoření nové instance a `dokuwiki-delsite` pro odstranění stávající instance. Tyto dva skripty z Debianu Jessie jsem lehce upravil pro naše použití a jsou k dispozici zde: [dokuwiki-addsite](https://github.com/JanOppolzer/howto/blob/master/dokuwiki-addsite) a [dokuwiki-delsite](https://github.com/JanOppolzer/howto/blob/master/dokuwiki-delsite). Skripty umístíme do adresáře `/usr/local/sbin/` a nastavíme je jako spustitelné pro vlastníka (uživatele *root*).

```bash
wget -P /usr/local/sbin \
    https://raw.githubusercontent.com/JanOppolzer/howto/master/dokuwiki-addsite \
    https://raw.githubusercontent.com/JanOppolzer/howto/master/dokuwiki-delsite
chmod 700 /usr/local/sbin/dokuwiki-*
```

Ve skriptu `dokuwiki-addsite` je vhodné upravit na řádku 105 hodnotu proměnné `$conf['superuser']`, která je nastavena na hodnotu mého [eduPersonPrincipalName](https://www.eduid.cz/cs/tech/attributes/edupersonprincipalname), tedy `jop@cesnet.cz`. Zde nastavený správce je *superuživatel*:

```php
105 \$conf['superuser'] = 'jop@cesnet.cz';
```

### DokuWiki Shibboleth Auth

Protože chceme využívat k přihlášení federaci identit [eduID.cz](http://eduID.cz), musíme si nainstalovat rozšíření pro DokuWiki, jejímž autorem je bývalý kolega Ivan Novakov. Toto rozšíření se nachází na GitHubu, odkud si ho naklonujeme utilitkou `git` pro snazší potenciální aktualizace.

```bash
apt install git
mkdir -p /opt/src/
git clone https://github.com/JanOppolzer/dokuwiki-shibboleth-auth /opt/src/dokuwiki-shibboleth-auth
cp -r /opt/src/dokuwiki-shibboleth-auth/plugin/authshibboleth/ /opt/dokuwiki/lib/plugins/
```

Nyní již můžeme přistoupit k vytváření jednotlivých instancí DokuWiki a jejich následné konfiguraci.

### Animal \#1

Vytvoříme si první instanci:

```bash
dokuwiki-addsite devnull.cesnet.cz
```

Veškerá data (= stránky, které vytvoříme) jsou uložena v adresáři `/var/www/farm/devnull.cesnet.cz/data/`. Všechna nastavení v adresáři `/var/www/farm/devnull.cesnet.cz/conf/`, kde musíme provést určité změny.

#### Federativní autentizace

V souboru `/var/www/farm/devnull.cesnet.cz/conf/authshibboleth.conf.php` si nastavíme mapování skupin z Peruna na skupiny v DokuWiki. Můžeme si také povolit lokální definice skupin pomocí souboru `custom_groups.php`:

```php
<?php

$conf['plugin']['authshibboleth'] = array(
    'group_source_config' => array(

        'entitlement' => array(
            'type' => 'environment',
            'options' => array(
                'source_attribute' => 'perunUniqueGroupName',
                'map' => array(
                    'urn:geant:cesnet.cz:group:VIRTUAL_ORGANIZATION:GROUP:SUBGROUP#perun.cesnet.cz' => 'SUBGROUP',
                    'urn:geant:cesnet.cz:group:cesnet:devnull:admins#perun.cesnet.cz'               => 'admins',
                ),
                'prefix' => ''
            )
        ),

        'custom' => array(
            'type' => 'file',
            'options' => array(
                'path' => __DIR__ . '/custom_groups.php'
            )
        )
    ),
);
```

Lokální nastavení skupin se provádí v souboru `/var/www/farm/devnull.cesnet.cz/conf/custom_groups.php`, pokud jsme si toto výše povolili. Soubor obsahuje zakomentované příklady, které by měly být dostačující pro pochopení funkčnosti.

```php
<?php

return array(
    /*
    'custom_group_1' => array(
        'tester',
        'foobar'
    ),
     */

    /*
    'custom_group_2' => array(
        'novakoi',
        'tester'
    )
     */
```

##### Nová jména skupin

Pokud jsme v Atributové autoritě měli nějaké skupiny, je nutné je přejmenovat, protože v rámci ProxyIdP mají jiná jména. Např. původní skupina `einfra:www.eduid.cz:internal_admins` se nyní jmenuje `urn:geant:cesnet.cz:group:einfra:www.eduid.cz:internal_admins#perun.cesnet.cz`.

Ke všem původním skupinám tedy musíme přidat prefix `urn:geant:cesnet.cz:group:` a doplnit postfix `#perun.cesnet.cz`. Potřebujete-li radu, pište na login@cesnet.cz.

#### Grafická šablona pro CESNET

Zajisté chceme, aby naše DokuWiki využívala grafickou šablonu pro CESNET. Tu si stáheme z *git* repozitáře na serveru [homeproj.cesnet.cz](https://homeproj.cesnet.cz) a vytvoříme symbolický odkaz na grafické téma do DokuWiki:

```bash
git clone <váš_login>@homeproj.cesnet.cz:design-dokuwiki /opt/src/design-dokuwiki
ln -s /opt/src/design-dokuwiki /opt/dokuwiki/lib/tpl/cesnet
```

Teď již jen zbývá nastavit téma *cesnet* jako používané, což provedeme v konfiguračním souboru `/var/www/farm/devnull.cesnet.cz/conf/local.php`:

```php
$conf['template'] = 'cesnet';
```

### Animal \#2, \#3, ...

Budeme-li chtít na našem serveru provozovat další instanci DokuWiki, jednoduše zavoláme opět skript `dokuwiki-addsite` s argumentem odpovídajícím názvu serveru a následně si doupravíme konfiguraci podle našich potřeb. Je samozřejmě potřeba také vytvořit další virtuální host v Apachi a následně Apache restartovat.

```bash
dokuwiki-addsite devnull1.cesnet.cz
dokuwiki-addsite devnull2.cesnet.cz
```

### Úpravy v DokuWiki

#### Přihlašovací formulář

Nechceme, aby se na stránce zobrazoval přihlašovací formulář, pokud nepřihlášený uživatel přistoupí na chráněnou stránku. Přihlašování se má dít pomocí kliknutí na tlačítko "Přihlásit" v záhlaví (linkeru), výběru domovské organizace z WAYFu a následně vyplněním přihlašovacích údajů na stránce domovské organizace. Toho docílíme triviální úpravou, kdy zakomentujeme na příslušném místě řádek s funkcí `html_login()` a vypíšeme informaci *"Možná jste se zapomněl(a) přihlásit.* Úpravu provedeme v souboru `/opt/dokuwiki/inc/html.php` ve funkci `html_denied()` na řádcích 86 a 87 takto:

```php
82 function html_denied() {
83     print p_locale_xhtml('denied');
84
85     if(empty($_SERVER['REMOTE_USER'])){
86         //html_login();
87         print "<p>Možná jste se zapomněl(a) <a href=\"javascript:document.getElementById('cesnet_login_link').click();\">přihlásit</a>.</p>";
88     }
89 }
```

### Poznámka k rozšířením

**Pokud můžete, VYHNĚTE se rozšířením!** Pokud si rozšíření nenapíšete sami a nejste ochotni se o něj starat, je vysoká pravděpodobnost, že rozšíření v další verzi DokuWiki bude fungovat špatně anebo dokonce vůbec a bude třeba vyvinout netriviální úsilí, aby opět fungovalo. Může se také stát, že se rozšíření bude nadále vyvíjet, ovšem způsobem, který bude naprosto nevyhovující. Takže ještě jednou: **pokud můžete, VYHNĚTE se rozšířením!**

### Migrace stávajících DokuWiki instancí

Chceme-li [přenést](https://www.dokuwiki.org/faq:servermove) stávající DokuWiki do námi právě vytvořené "farmy", stačí přenést **celý datový adresář** (na Debianu 8/Jessie se jedná o adresář `/var/lib/dokuwiki/data/`) ze starého serveru na nový. Musíme však pamatovat na správné umístění na novém serveru s farmou, tedy `/var/www/farm/<jméno_dokuwiki_instance>/data/`, přičemž novou instanci si nejprve vytvoříme příkazem `dokuwiki-addsite <jméno_dokuwiki_instance>`.

Aby nedošlo ke změně posledních úprav u jednotlivých souborů, je vhodné na starém serveru *datový adresář* zabalit do TAR archivu a po přesunu na nový server archiv rozbalit. Je třeba též dodržet správná přístupová práva.

Musíme také samozřejmě vytvořit odpovídající virtuální host v Apachi, zažádat si o nový SSL certifikát s dodatečnou doménou pro novou instanci DokuWiki a zařídit změnu DNS záznamů. Pak už opravdu stačí jen příkazem `dokuwiki-addsite` vytvořit tuto novou instanci a překopírovat datový adresář. Následně ještě musíme provést potřebné konfigurační úpravy DokuWiki nové instance a zkontrolovat všechna potřebná rozšíření. Velice důrazně doporučuji prověřit správnost nastavení přístupových práv ke stránkám.

