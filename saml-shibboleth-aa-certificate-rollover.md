# Standalone SAML Shibboleth Attribute Authority Certificate Rollover

This document describes correct certificate rollover technique for standalone SAML Attribute Authority (AA) based on Shibboleth IdP with Apache HTTPD and Apache Tomcat.

1. Obtain a new trusted HTTPS certificate.
2. Add the certificate to AA metadata after the current one:

```xml
<?xml version="1.0" encoding="utf-8"?>
<EntityDescriptor xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
  <AttributeAuthorityDescriptor>

    <KeyDescriptor>
      <ds:KeyInfo>
        <ds:X509Data>
          <ds:X509Certificate>

            <!-- current certificate -->

          </ds:X509Certificate>
        </ds:X509Data>
      </ds:KeyInfo>
    </KeyDescriptor>

    <KeyDescriptor>
      <ds:KeyInfo>
        <ds:X509Data>
          <ds:X509Certificate>

            <!-- new certificate -->

          </ds:X509Certificate>
        </ds:X509Data>
      </ds:KeyInfo>
    </KeyDescriptor>

  </AttributeAuthorityDescriptor>
</EntityDescriptor>
```

3. Update federation metadata and wait for distribution to all Service Providers.
4. Change certificate within Apache HTTPD and Shibboleth IdP.
5. Restart Apache and Shibboleth IdP.
6. Drop the old certificate from metadata.
7. Update federation metadata.

