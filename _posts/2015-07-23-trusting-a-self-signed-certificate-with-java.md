---
layout: post
title:  "Trusting a Self-Signed Certificate with Java"
comments: true
date:   2015-07-27 15:14:07
categories: code
---

I know next to nothing about SSL certificates so all this was mostly winged.

**Scenario**: Authenticate a user using a remote LDAP server.  

Having enabled the intercept URL

```xml
<security:intercept-url pattern="/**" access="IS_AUTHENTICATED_FULLY" />
```

and configured the login page to be unsecured

```xml
<security:http security="none" pattern="/login" />
```

I was able to post credentials to the LDAP server, however I would get an error when trying to complete the handshake.  [Atlassian] have provided a nice little utility to let you test this on the command line.  You can download this and give it a go for example:

```
$ java SSLPoke my.domain.to.test 10636
```

Here you can see that I have provided the server address and a port to test.  I am using a standard secure LDAP port here and not  the default 636 port as everything below port 1024 is considered a trusted port i.e. requires sudo.  You should get an error message similar to this

```
sun.security.validator.ValidatorException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
```

Essentially we need to manually tell Java that this is a trusted server, we can do this by grabbing the certificate from the server using openSSL

```
$ openssl s_client -connect my.domain.to.test:10636
```

This will return some data and you want to grab the lines that start with
---BEGIN CERTIFICATE---
blahblahblah
---END CERTIFICATE---  
Copy these into a file and save it somewhere with a .crt extension e.g. ldapca.crt  Now that we have the certificate from the the server we can add it to the trusted sources in our JRE

```
$ cd $JRE_HOME/lib/security
$ keytool -import -alias cacert -keystore cacerts -storepass changeit -file ldapca.crt
```

So we now have a keystore that contains the LDAP server certificate. You can now try out SSLPoke again and pass this keystore in as a param

```
$ java -Djavax.net.ssl.trustStore=my/dir/cacert SSLPoke my.domain.to.test 10636
Successfully connected
```

Connection now succeeds.  I chose to implement this by adding the Java option to my Tomcat JAVA_OPTS but I'm sure there's loads of ways you can achieve this.

[Atlassian]:https://confluence.atlassian.com/display/JIRAKB/Unable+to+Connect+to+SSL+Services+due+to+PKIX+Path+Building+Failed+sun.security.provider.certpath.SunCertPathBuilderException
