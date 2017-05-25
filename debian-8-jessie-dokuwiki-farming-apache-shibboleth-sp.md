# DokuWiki Farming s Apache a Shibboleth SP na Debianu

V případě, že chceme provozovat několik různých DokuWiki instancí pro různé projekty, není potřeba mít pro každou DokuWiki vlastní [virtuální nebo fyzický] server a starat se tak o několik operačních systémů. Je možné použít tzv. [DokuWiki Farming](https://www.dokuwiki.org/farms), který je v rámci [Debianu](https://www.debian.org) velice dobře podporován pomocí příkazů `dokuwiki-addsite` (přidání nové instance) a `dokuwiki-delsite` (odebrání existující instance).

Všechny instance sdílí jednu instalaci DokuWiki a také rozšíření. Mají však své vlastní stránky a také nastavení (přístupová práva, vzhled, použitá rozšíření, způsob přihlašování, ...).

Tento dokument počítá s tím, že všechny instance budou používat přihlašování pomocí [České akademické federace identit eduID.cz](https://www.eduid.cz). To je implementováno pomocí rozšíření [dokuwiki-shibboleth-auth](https://github.com/ivan-novakov/dokuwiki-shibboleth-auth). Zároveň hostující server bude z pohledu federace jeden poskytovatel služby neboli jedna služba (Service Provider, SP) s jedním *entityID*. Jednotlivé instance DokuWiki běžící na svých vlastních adresách (doménách) budou mít své "koncové body" (endpointy) uvedeny v metadatech jediné entity. V návodu je použit CESNETí "linker" (horní záhlaví/hlavička) a [jednotný vzhled pro DokuWiki](https://wiki.cesnet.cz/doku.php?id=groups:weby:dokuwiki:start).

Níže popsaný přístup sníží nároky jak na správu operačních systémů (bude nutné spravovat jen jeden operační systém), tak instalace DokuWiki i federativní autentizace. Jeden operační systém, jedna instalace DokuWiki, jeden Shibboleth SP s jedním certifikátem atd. Budeme-li mít certifikát pro webserver obsahující v alternativních jménech všechny potřebné domény (doporučený způsob), budeme mít i jeden SSL certifikát. Znamená to ovšem, že když vyprší SSL certifikát, vyprší pro všechny instance najednou. Rozbije-li se DokuWiki např. při neopatrné aktualizaci, rozbije to všechny instance najednou. (*With great power comes great responsibility.*)

V krocích níže se předpokládá použití [Debianu 8](https://www.debian.org/releases/jessie/) s kódovým označením [Jessie](https://www.debian.org/releases/jessie/).

Jako první nainstalujeme a nakonfigurujeme Apache a zprovozníme si virtuální hosty včetně SSL konfigurace. Následně nainstalujeme a nakonfigurujeme Shibboleth SP a zprovozníme federativní přihlašování. Poté bude na řadě instalace a konfigurace DokuWiki a její nakonfigurování v Apachi. Posledním krokem, který bude v řadě případů nepotřebný, bude návod, jak převést stávající DokuWiki instance na novou instalaci -- to pro případ, že byste konsolidovali více existujících instancí DokuWiki na několika serverech na jeden server.

## Konvence

V návodu používám následující:

* DNS název fyzického serveru (hostname): `blackhole.cesnet.cz`
* entityID služby ve federaci: `https://blackhole.cesnet.cz/shibboleth`
* hostované weby/domény: `blackhole.cesnet.cz`, `blackhole1.cesnet.cz`, `blackhole2.cesnet.cz`
* SSL certifikát se všemi hostovanými doménami: `/etc/ssl/certs/blackhole.cesnet.cz.crt.pem`
* privátní klíč k SSL certifikátu: `/etc/ssl/private/blackhole.cesnet.cz.key.pem`
* řetěz certifikátů až ke kořenové CA: `/etc/ssl/certs/chain_TERENA_SSL_CA_3.pem`

## Apache

Instalaci Apache provedeme obvyklým způsobem pomocí příkazu `apt-get`.

```bash
apt-get install apache2
```

Nejprve nakonfigurujeme server `blackhole.cesnet.cz`. Vytvoříme konfigurační soubor `/etc/apache2/sites-available/blackhole.cesnet.cz.conf` s následujícím obsahem:

```apache
<VirtualHost *:80>
    ServerName              blackhole.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    DocumentRoot            /var/www/blackhole.cesnet.cz/

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
    SSLCipherSuite          'EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH:+CAMELLIA256:+AES256:+CAMELLIA128

    SSLCertificateFile      /etc/ssl/certs/blackhole.cesnet.cz.crt.pem
    SSLCertificateKeyFile   /etc/ssl/private/blackhole.cesnet.cz.key.pem
    SSLCertificateChainFile /etc/ssl/certs/chain_TERENA_SSL_CA_3.pem
</VirtualHost>
```

Vytvoříme adresář `/var/www/blackhole.cesnet.cz/` a v něm soubor `index.html` obsahující pouze text `blackhole.cesnet.cz`.

```bash
mkdir -p /var/www/blackhole.cesnet.cz/
echo 'blackhole.cesnet.cz' > /var/www/blackhole.cesnet.cz/index.html
```

SSL certifikát umístíme do souboru `/etc/ssl/certs/blackhole.cesnet.cz.crt.pem`, privátní klíč do `/etc/ssl/private/blackhole.cesnet.cz.key.pem` a řetěz certifikátů až ke kořenové CA do `/etc/ssl/certs/chain_TERENA_SSL_CA_3.pem`.

Nyní v konfiguraci Apache vypneme všechny virtuální hosty -- tím zajistíme, že výchozí nastavení (`/etc/apache2/sites-available/000-default.conf`), které je po instalaci Apache ve výchozím stavu povolené, nebude nadále aktivní. Následně zapneme nově nadefinovaný server `blackhole.cesnet.cz.conf`.

Jelikož v konfiguraci jednotlivých serverů používáme *SSL* certifikáty pro komunikaci přes *HTTPS*, je potřeba aktivovat modul `ssl` v Apachi. Navíc u SSL nastavujeme *HSTS* hlavičku, takže je potřeba aktivovat i modul `headers`. Pak již zbývá jen restartovat Apache.

```bash
a2dissite *
a2ensite blackhole.cesnet.cz.conf
a2enmod ssl headers
service apache2 restart
```

Vyzkoušíme se připojit webovým prohlížečem a měli bychom vidět text `blackhole.cesnet.cz`. Nyní můžeme přejít ke konfiguraci zbývajících virtuálních hostů `blackhole1.cesnet.cz` a `blackhole2.cesnet.cz`. Konfigurační soubory (`blackhole1.cesnet.cz.conf`, `blackhole2.cesnet.cz.conf`) budou velice podobné výše uvedenému souboru `blackhole.cesnet.cz.conf`. Změny budou jen v parametrech `ServerName`, `DocumentRoot`, `Redirect`, `ErrorLog` a `CustomLog`.

Konfigurační soubor `/etc/apache2/sites-available/blackhole1.cesnet.cz.conf`:
```apache
<VirtualHost *:80>
    ServerName              blackhole1.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    DocumentRoot            /var/www/blackhole1.cesnet.cz/

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

    DocumentRoot            /var/www/blackhole2.cesnet.cz/

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
service apache2 reload
```

Nyní ověříme, že se dostaneme na všechny weby a to jak protokolem *HTTP*, tak protokolem *HTTPS*: `blackhole.cesnet.cz`, `blackhole1.cesnet.cz` i `blackhole2.cesnet.cz`.

## Shibboleth SP

Provozovat Shibboleth SP ve spolupráci s web serverem Apache je velice jednoduché, stačí doinstalovat modul do Apache, povolit ho a upravit nastavení virtuálních hostů, kde má být modul použit:

Instalace modulu Shibboleth SP do Apache:

```bash
apt-get install libapache2-mod-shib2
```

Nyní se můžeme pustit do konfigurace Shibboleth SP. Po instalaci výše uvedeného balíčku si vygenerujeme certifikát pro Shibboleth, který bude v metadatech entity. Následující příkaz vygeneruje self-signed certifikát, nastaví mu vhodná práva i jméno, abychom měli co nejsnazší nastavení. Tento certifikát bude mít platnost 10 let (pomocí parametru `-y` je možné ovlivnit jeho platnost v letech), bude vystaven pro server/hostname `blackhole.cesnet.cz` (parametr `-h`) s entityID `https://blackhole.cesnet.cz/shibboleth` (parametr `-e`).

```bash
shib-keygen -h blackhole.cesnet.cz -e https://blackhole.cesnet.cz/shibboleth
```

Zbytek konfigurace se odehrává v konfiguračním souboru `/etc/shibboleth/shibboleth2.xml`.

Jako první nastavíme entityID (`https://blackhole.cesnet.cz/shibboleth`), což je jedinečný identifikátor entity v rámci federace identit:
```xml
<ApplicationDefaults entityID="https://blackhole.cesnet.cz/shibboleth"
                     REMOTE_USER="eppn persistent-id targeted-id">
```

Dále nakonfigurujeme SSL (změníme `handlerSSL` na *true* a `cookieProps` na *https*):
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

Budeme-li využívat dodatečné atributy z *[atributové autority](https://www.eduid.cz/cs/aa)* (popis [konfigurace](https://www.eduid.cz/cs/tech/aa) je na stránkách [eduID.cz](https://www.eduid.cz)) pro řízení přístupu k jednotlivým instancím DokuWiki, musí naše služba znát i metadata atributové autority:
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

Nyní se dostáváme k dodatečným informacím v metadatech služby. Vytvoříme tedy soubor `/etc/shibboleth/metadata-template.xml` s následujícím obsahem, který modifikujeme (zejména UIInfo a alternativní DNS názvy) dle svých potřeb. Nebudeme zadávat "endpointy" (elementy `<AssertionConsumerService>`) pro `blackhole.cesnet.cz`, ale pouze pro `blackhole1.cesnet.cz` a `blackhole2.cesnet.cz`.
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

V konfiguračním souboru `/etc/shibboleth/attribute-map.xml` musíme odkomentovat několik řádků, abychom získali další uživatelské atributy nad rámec těch definovaných standardně. Jelikož některá IdP by stále nemusela podporovat nový protokol SAML2, budeme akceptovat i atributy kódované protokolem SAML1. Budeme očekávat následující atributy `cn` (celé jméno), `sn` (příjmení), `givenName` (křestní jméno), `displayName` (celé jméno), `mail` (e-mailová adresa), `telephoneNumber` (telefonní číslo), `o` (organizace), `ou` (organizační jednotka).

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

Budeme-li využívat služeb *[atributové autority](https://www.eduid.cz/cs/aa)* (samostatný popis [konfigurace](https://www.eduid.cz/cs/tech/aa) atributové autority je na stránkách [eduID.cz](https://www.eduid.cz)), jejíž data spravuje systém [Perun](https://perun.cesnet.cz/web/), je potřeba v Shibbolethu nakonfigurovat dodatečné atributy, jejichž hodnoty budou plněny právě z atributové autority.

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
service shibd restart
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

Nyní, přistoupíme-li na `https://blackhole.cesnet.cz/shib1/` (případně `https://blackhole1.cesnet.cz/shib1/` nebo `https://blackhole2.cesnet.cz/shib1/`), uvidíme text `shib1`. V tomto adresáři, ač je aktivován Shibboleth, není vyžadováno, aby existovalo sezení. Jedná se o tzv. *přihlášení na vyžádání* (v angličtině *lazy session*).

Pokud však přistoupíme na `https://blackhole.cesnet.cz/shib2/` (případně `https://blackhole1.cesnet.cz/shib2/` nebo `https://blackhole2.cesnet.cz/shib2/`), budeme přesměrováni na stránku, kde si vybereme svého poskytovatele identity, u něhož se chceme ověřit svým přihlašovacím jménem a heslem. Teprve po úspěšném přihlášení uvidíme na těchto stránkách text `shib2`. (Budeme-li se ověřovat pokaždé u stejné domovské organizace, vytvoříme si poprvé sezení na IdP, takže nebudeme muset v dalších případech zadávat jméno a heslo, budeme-li používat stále stejný webový prohlížeč a budeme-li mít stále cookies z předchozího přihlášení.)

Existenci sezení na námi hostovanách stránkách je možné ověřit na adrese `https://blackhole.cesnet.cz/Shibboleth.sso/Session` (případně `https://blackhole1.cesnet.cz/Shibboleth.sso/Session` nebo `https://blackhole2.cesnet.cz/Shibboleth.sso/Session`). Pokud sezení neexistuje, uvidíme pouze informaci `A valid session was not found.`, v opačném případě uvidíme např. tento výpis:

```
Miscellaneous
Session Expiration (barring inactivity): 479 minute(s)
Client Address: 2001:718:1:6::134:166
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
service apache2 reload
```

## DokuWiki

Jsou-li Apache a Shibboleth SP nainstalovány a korektně nastaveny, můžeme postoupit k instalaci DokuWiki pomocí příkazu `apt-get`.

```bash
apt-get install dokuwiki
```

### První instance

Nyní si vytvoříme první instanci DokuWiki a to pro `blackhole.cesnet.cz`:

```bash
dokuwiki-addsite blackhole.cesnet.cz
```

Pro autentizaci pomocí Shibbolethu budeme využívat rozšíření [dokuwiki-shibboleth-auth](https://github.com/ivan-novakov/dokuwiki-shibboleth-auth), které si naklonujeme z jeho Git repozitáře do adresáře `/opt`:

```bash
cd /opt
git clone https://github.com/ivan-novakov/dokuwiki-shibboleth-auth
```

Následně rozšíření a konfigurační soubory překopírujeme na svá místa. V podadresáři `conf/` se nachází soubor `local.protected.php`, se kterým však musíme zacházet opatrně, abychom si nepřepsali soubor vytvořený příkazem `dokuwiki-addsite` (udává, kde jsou uloženy data DokuWiki), proto tento soubor kopírovat nebudeme, překopírujeme pouze jeho obsah definující využívání rozšíření `dokuwiki-shibboleth-auth`:

```bash
cp -r /opt/dokuwiki-shibboleth-auth/plugin/authshibboleth/ /var/lib/dokuwiki/lib/plugins/
cp /opt/dokuwiki-shibboleth-auth/conf/authshibboleth.conf.php /etc/dokuwiki/farm/blackhole.cesnet.cz/
cp /opt/dokuwiki-shibboleth-auth/conf/custom_groups.php /etc/dokuwiki/farm/blackhole.cesnet.cz/
grep authshibboleth.conf.php /opt/dokuwiki-shibboleth-auth/conf/local.protected.php >> /etc/dokuwiki/farm/blackhole.cesnet.cz/local.protected.php
```

Opravíme konfiguraci Apache, aby zobrazoval stránky nově vytvořené instance DokuWiki. Budeme používat tzv. *lazy sessions*, tedy přihlášení bude vyžadováno pouze pro neveřejné stránky anebo pro úpravy stránek (práva v DokuWiki jsou nad rámec této dokumentace):
```apache
<VirtualHost *:80>
    ServerName              blackhole.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    DocumentRoot            /usr/share/dokuwiki

    Redirect                permanent / https://blackhole.cesnet.cz/

    ErrorLog                ${APACHE_LOG_DIR}/error-blackhole.log
    CustomLog               ${APACHE_LOG_DIR}/access-blackhole.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName              blackhole.cesnet.cz
    ServerAdmin             jan.oppolzer@cesnet.cz

    DocumentRoot            /usr/share/dokuwiki

    ErrorLog                ${APACHE_LOG_DIR}/ssl-error-blackhole.log
    CustomLog               ${APACHE_LOG_DIR}/ssl-access-blackhole.log combined

    SSLEngine               On    
    SSLProtocol             All -SSLv2 -SSLv3    
    SSLHonorCipherOrder     On    
    SSLCompression          Off    
    Header                  always set Strict-Transport-Security "max-age=15768000"
    SSLCipherSuite          'EDH+CAMELLIA:EDH+aRSA:EECDH+aRSA+AESGCM:EECDH+aRSA+SHA384:EECDH+aRSA+SHA256:EECDH:+CAMELLIA256:+AES256:+CAMELLIA128

    SSLCertificateFile      /etc/ssl/certs/blackhole.cesnet.cz.crt.pem
    SSLCertificateKeyFile   /etc/ssl/private/blackhole.cesnet.cz.key.pem
    SSLCertificateChainFile /etc/ssl/certs/chain_TERENA_SSL_CA_3.pem

    <Directory /usr/share/dokuwiki/>
        DirectoryIndex doku.php

        AuthType shibboleth
        ShibRequireSession Off 
        require shibboleth

        Order Deny,Allow
        Allow from all
        Deny from all

        RewriteEngine on
        RewriteBase /
        RewriteRule ^_media/(.*)              lib/exe/fetch.php?media=$1  [QSA,L]
        RewriteRule ^_detail/(.*)             lib/exe/detail.php?media=$1  [QSA,L]
        RewriteRule ^_export/([^/]+)/(.*)     doku.php?do=export_$1&id=$2  [QSA,L]
        RewriteRule ^$                        doku.php?id=cs:index  [L] 
        RewriteCond %{REQUEST_FILENAME}       !-f 
        RewriteCond %{REQUEST_FILENAME}       !-d 
        RewriteRule (.*)                      doku.php?id=$1  [QSA,L]
    </Directory>

</VirtualHost>
```
Konfigurace DokuWiki využívá modul `rewrite`, který musíme zapnout a následně restartovat Apache.
```bash
a2enmod rewrite
service apache2 restart
```

Většina konfigurace probíhá v souboru `/etc/dokuwiki/farm/blackhole.cesnet.cz/local.php`, jehož nastavení lze měnit i pomocí webového konfiguračního rozhraní DokuWiki. My v něm určitě nastavíme grafické téma DokuWiki, které si o několik kroků níže stáheme.
```php
<?php

$conf['title'] = 'blackhole.cesnet.cz';
$conf['start'] = 'index';
$conf['lang'] = 'cs';
$conf['template'] = 'blackhole';
$conf['useacl'] = 1;
$conf['defaultgroup'] = 'ALL';
$conf['userewrite'] = '1';
$conf['useslash'] = 1;
```

V konfiguračním souboru `/etc/dokuwiki/farm/blackhole.cesnet.cz/local.protected.php` stačí nahrát soubor zajišťující přihlašování pomocí Shibbolethu, zapnout tento modul, určit cestu k datům této instance (to dělá automaticky skript `dokuwiki-addsite`) a je vhodné nastavit superuživatele (jako hodnotu použijte hodnotu atributu [eduPersonPrincipalName](https://www.eduid.cz/cs/tech/attributes/edupersonprincipalname), kterou můžete pro svou identitu získat např. na portálu [attributes.eduid.cz](https://attributes.eduid.cz)). Tento soubor (narozdíl od souboru `local.php`) nelze přepsat ve webovém konfiguračním rozhraní DokuWiki, je tedy vhodné zde uvádět informace, které se opravdu nemají měnit.
```php
<?php

include __DIR__ . '/authshibboleth.conf.php';

$conf['authtype'] = 'authshibboleth';
$conf['savedir'] = '/var/lib/dokuwiki/farm/blackhole.cesnet.cz/data';
$conf['superuser'] = 'jop@cesnet.cz';
```

Výchozí konfigurační soubor pro rozšíření zajišťující přihlašování pomocí Shibbolethu v `/etc/dokuwiki/farm/blackhole.cesnet.cz/authshibboleth.conf.php` obsahuje dokumentaci ke všem volbám. Konfigurace bude ve většině případů velice jednoduchá. Stačí definovat virtuální organizaci, skupinu a podskupinu v Perunovi a na jakou skupinu v DokuWiki budou uživatelé namapováni. Pak stačí práva pro danou skupinu definovat v `/etc/dokuwiki/farm/blackhole.cesnet.cz/acl.auth.php`.
```php
<?php

$conf['plugin']['authshibboleth'] = array(
    'group_source_config' => array(

        'entitlement' => array(
            'type' => 'environment',
            'options' => array(
                'source_attribute' => 'perunUniqueGroupName',
                'map' => array(
                    'VIRTUAL_ORGANIZATION:GROUP:SUBGROUP' => 'SUBGROUP',
                    'cesnet:blackhole:admins'             => 'admins',
                    'cesnet:blackhole'                    => 'blackhole'
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

V souboru `custom_groups.php` je možné staticky definovat vlastní skupiny. Soubor obsahuje příklady, které je vhodné smazat nebo alespoň zakomentovat, pokud se tento soubor bude díky předchozí konfiguraci načítat.

```php
<?php

return array(
    'admins' => array(
        //'tester',
        //'foobar'
    ),
    
    'internal_users' => array(
        //'novakoi',
        //'tester'
    ),

    'idp_doku' => array(
        //'foo'
    ),
);
```

Ještě musíme stáhnout grafickou šablonu DokuWiki pro CESNET:
```bash
cd /opt
git clone jop@homeproj.cesnet.cz:design-dokuwiki
```

V případě, že budeme provozovat více instancí DokuWiki, je vhodné grafickou šablonu nelinkovat, ale kopírovat. Barvy pro grafické téma budou totiž s největší pravděpodobností odlišné pro každou instanci. (Konfigurace barev pro šablonu je mimo rozsah tohoto návodu.)
```bash
cp -r /opt/design-dokuwiki /var/lib/dokuwiki/lib/tpl/blackhole
```

### Další instance

Podobně můžeme nyní příkazem `dokuwiki-addsite` vytvořit zbývající instance DokuWiki (`blackhole1.cesnet.cz` a `blackhole2.cesnet.cz`):

```bash
dokuwiki-addsite blackhole1.cesnet.cz
dokuwiki-addsite blackhole1.cesnet.cz
```

Pro konfiguraci těchto instancí je možné projít výše uvedený návod. Je třeba překopírovat na odpovídající místo soubory pro rozšíření `dokuwiki-shibboleth-auth` + nakonfigurovat rozšíření, nastavit DokuWiki ve virtuálním hostu v Apachi + provést jeho restart a v neposlední řadě také nakonfigurovat samotnou DokuWiki a upravit vzhled tématu do požadovaných barev.

### Úpravy v DokuWiki

V následující části jsou popsány úpravy, které je potřeba udělat ručně, jsou-li vyžadovány.

#### Přihlašovací formulář

Nechceme, aby se na stránce zobrazoval přihlašovací formulář, pokud nepřihlášený uživatel přistoupí na chráněnou stránku. Přihlašování se má dít pomocí kliknutí na tlačítko "Přihlásit" v záhlaví (linkeru), výběru domovské organizace z WAYFu a následně vyplněním přihlašovacích údajů na stránce domovské organizace.


`/var/lib/dokuwiki/inc/html.php`
```php
  77     if(!$_SERVER['REMOTE_USER']){
  78         //html_login();
  79         print "<p>Možná jste se zapomněl(a) přihlásit.</p>";
  80     }
```

#### Rozšíření `deflist`

Rozšíření `deflist` (ke stažení na [www.dokuwiki.org](https://www.dokuwiki.org/plugin:deflist)) umožňuje snadnou syntaxí do stránek vkládat tzv. *definiční seznamy* (anglicky *description lists*). Ty jsou v běžném HTML generovány pomocí tagů `<dl>` (*description list*), `<dt>` (*description term*) a `<dd>` (*descrition description*).

Původní rozšíření z [DokuWiki](https://www.dokuwiki.org/plugin:deflist) však definice sází *patkovým písmem*. To lze snadno napravit v souboru s kaskádovými styly a zobrazovat tak definiční seznamy písmem bezpatkovým jako všechny ostatní texty v aktuální CESNETí šabloně pro DokuWiki.

```css
$ colordiff -u style.css.orig style.css
--- style.css.orig      2017-02-16 13:57:46.022157515 +0100
+++ style.css   2017-02-16 14:21:46.422207044 +0100
@@ -2,7 +2,7 @@
        Copyright (C) 2005, 2008  M.Watermann, D-10247 Berlin, FRG  -  <support@mwat.de>
 */
 dl{font-size:1em;margin:0.5ex 0 0.5ex 1ex;padding:0.5ex 0 0.5ex 1ex;}
-dl dt{font:105% "Book Antiqua","Bitstream Vera Serif",Times,serif;font-weight:bolder;margin:0.5ex 0 0 0;}
+dl dt{font:105% sans-serif;font-weight:bolder;margin:0.5ex 0 0 0;}
 dl dt a,dl dt a:hover{text-decoration:none;}
 dl dd{margin:0 0 0 2ex;text-align:justify;}
 dl dd dl{margin:0;padding:0;}
```

## Migrace stávajících DokuWiki instancí

Chceme-li [přenést](https://www.dokuwiki.org/faq:servermove) stávající DokuWiki do námi právě vytvořené "farmy", stačí přenést **celý datový adresář** (na Debianu se jedná o adresář `/var/lib/dokuwiki/data/`) ze starého serveru na nový. Musíme však pamatovat na správné umístění na novém serveru s farmou, tedy `/var/lib/dokuwiki/farm/<jméno_dokuwiki_instance>/data/`, přičemž novou instanci si nejprve vytvoříme příkazem `dokuwiki-addsite <jméno_dokuwiki_instance>`.

Aby nedošlo ke změně posledních úprav u jednotlivých souborů, je vhodné na starém serveru *datový adresář* zabalit do TAR archivu a po přesunu na nový server archiv rozbalit. Je třeba též dodržet správná přístupová práva.

Musíme také samozřejmě vytvořit odpovídající virtuální host v Apachi, zažádat si o nový SSL certifikát s dodatečnou doménou pro novou instanci DokuWiki a zařídit změnu DNS záznamů. Pak už opravdu stačí jen příkazem `dokuwiki-addsite` vytvořit tuto novou instanci a překopírovat datový adresář. Následně ještě musíme provést potřebné konfigurační úpravy DokuWiki nové instance a zkontrolovat všechna potřebná rozšíření.
