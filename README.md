# ssl-troubleshooting
Tips/Tricks/Tools on troubleshooting SSL Certs

The reason this is here, is to help people troubleshoot, diagnose and fix problems with their certificates. This is not (necsesarily) about self-signed certs, but instead about you either getting a cert from somewhere like LetsEncrypt or purchasing a cert. Common symptoms with certs are that you have a server cert for your email, web, whatever service and when the client connects to it, it says the certificate is either invalid or untrusted or something about a chain and can't validate.

A client might say that a certificate is invalid if it can't read your certificate outright (maybe a persmission problems on your filesystem and the service can't serve the certificate at all) or more likely that you have a certificate that was issued to you but needs *additional* certificates in the chain of trust to validate that your cert is good.

A cautionary tale: Please don't EVER bypass or accept a certificate that can't validate. If you do, you could simply be accepting a cert for a service that you have no idea how the cert was issued. More importantly you could be the victim of an attack because someone has create a cert with your FQDN and is acting as a man in the middle and forwarding on your requests to the real server. If you don't actually validate the cert, you really have no idea if the cert is good or not. 

## Certificating chaining
Cert chaining is where you have a certificate that depends on another upstream certificate that COULD depend on yet another upstream certificate until you get to a trusted (almost certainly pre-installed) root certificate of authority (CA). You may recall the day sof when LetsEncrypt was new and issuing certs but nobody really trusted them until the browser vendors gave out the CA for LetsEncrypt. Once that happend HTTPS became "easy". 
An example of chaining can be found here: https://knowledge.digicert.com/solution/SO16297.html


## Testing certificates in general
OpenSSL has many tricks up it's sleeve and one of them is the ability to act like a client to a secure server and see what's going on.

### OpenSSL Connect
For this example, lets connect to slashdot.com and you will see that this chain has two certs. The first one designated with the *0 s:/CN=slashdot.org*  is the host cert (like what you'd request from a certificate vendor) and the next one up (*1 s:/C=US/O=Let's Encrypt/CN=Let's Encrypt Authority X3*) is what has validated the certificate and is the Root CA installed in your browser (or OS or whatever) that lets the client verify the issuer did indeed give out slashdot.com. For this example there is NO immediate certificate just the server cert and the Root CA from Let's Encrypt. 

```
openssl s_client -connect slashdot.com:443
CONNECTED(00000003)
depth=2 O = Digital Signature Trust Co., CN = DST Root CA X3
verify return:1
depth=1 C = US, O = Let's Encrypt, CN = Let's Encrypt Authority X3
verify return:1
depth=0 CN = slashdot.org
verify return:1
---
Certificate chain
 0 s:/CN=slashdot.org
   i:/C=US/O=Let's Encrypt/CN=Let's Encrypt Authority X3
 1 s:/C=US/O=Let's Encrypt/CN=Let's Encrypt Authority X3
   i:/O=Digital Signature Trust Co./CN=DST Root CA X3
---
```
for the next example, I'll show a cert chain that has an intermediate cert 

```
openssl s_client -connect wellsfargo.com:443
CONNECTED(00000003)
depth=2 C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root G2
verify return:1
depth=1 C = US, O = Wells Fargo & Company, OU = Organization Validated TLS, CN = Wells Fargo Public Trust Certification Authority 01 G2
verify return:1
depth=0 C = US, ST = California, L = San Francisco, O = Wells Fargo & Company, OU = DCG-PSG, CN = wellsfargo.com
verify return:1
---
Certificate chain
 0 s:/C=US/ST=California/L=San Francisco/O=Wells Fargo & Company/OU=DCG-PSG/CN=wellsfargo.com
   i:/C=US/O=Wells Fargo & Company/OU=Organization Validated TLS/CN=Wells Fargo Public Trust Certification Authority 01 G2
 1 s:/C=US/O=Wells Fargo & Company/OU=Organization Validated TLS/CN=Wells Fargo Public Trust Certification Authority 01 G2
   i:/C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert Global Root G2
 2 s:/C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert Global Root G2
   i:/C=US/O=DigiCert Inc/OU=www.digicert.com/CN=DigiCert Global Root G2
---
```
So for this connection to Wells Fargo, you can see that wellsfargo.com was issued by *Wells Fargo Public Trust Certification Authority 01 G2* and THAT cert was issued by *DigiCert Global Root G2* and then ROOT cert is DigiCert Global Root G2

A great example of a cert NOT verifying ie openssl is saying: *Verify return code: 21 (unable to verify the first certificate)*
```
 openssl s_client   -connect logging.fakeserver.com:6514
CONNECTED(00000003)
depth=0 OU = Domain Control Validated, OU = GoGetSSL Wildcard SSL, CN = *.fakeserver.com
verify error:num=20:unable to get local issuer certificate
verify return:1
depth=0 OU = Domain Control Validated, OU = GoGetSSL Wildcard SSL, CN = *.fakeserver.com
verify error:num=27:certificate not trusted
verify return:1
depth=0 OU = Domain Control Validated, OU = GoGetSSL Wildcard SSL, CN = *.fakeserver.com
verify error:num=21:unable to verify the first certificate
verify return:1
---
Certificate chain
 0 s:/OU=Domain Control Validated/OU=GoGetSSL Wildcard SSL/CN=*.fakeserver.com
   i:/C=LV/L=Riga/O=GoGetSSL/CN=GoGetSSL RSA DV CA
 1 s:/C=US/ST=New Jersey/L=Jersey City/O=The USERTRUST Network/CN=USERTrust RSA Certification Authority
   i:/C=SE/O=AddTrust AB/OU=AddTrust External TTP Network/CN=AddTrust External CA Root
---
```
You probably noticed that OpenSSL is very unhappy with the above. The problem is that *.fakeserver.com (a star cert) jumps right to the CA (USERTrust RSA Certification Authority) when we really need the GoGetSSL RSA DV CA cert in the chain. If we add that cert in, the you will see something like this:
```
openssl s_client   -connect logging.fakeserver.com:6514
CONNECTED(00000003)
depth=3 C = SE, O = AddTrust AB, OU = AddTrust External TTP Network, CN = AddTrust External CA Root
verify return:1
depth=2 C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
verify return:1
depth=1 C = LV, L = Riga, O = GoGetSSL, CN = GoGetSSL RSA DV CA
verify return:1
depth=0 OU = Domain Control Validated, OU = GoGetSSL Wildcard SSL, CN = *.fakeserver.com
verify return:1
---
Certificate chain
 0 s:/OU=Domain Control Validated/OU=GoGetSSL Wildcard SSL/CN=*.fakeserver.com
   i:/C=LV/L=Riga/O=GoGetSSL/CN=GoGetSSL RSA DV CA
 1 s:/C=LV/L=Riga/O=GoGetSSL/CN=GoGetSSL RSA DV CA
   i:/C=US/ST=New Jersey/L=Jersey City/O=The USERTRUST Network/CN=USERTrust RSA Certification Authority
 2 s:/C=US/ST=New Jersey/L=Jersey City/O=The USERTRUST Network/CN=USERTrust RSA Certification Authority
   i:/C=SE/O=AddTrust AB/OU=AddTrust External TTP Network/CN=AddTrust External CA Root
---
```
Note that we are now CORRECTLY jumping from GoGetSSL RSA DV CA to The USERTRUST Network/CN=USERTrust RSA Certification Authority and THEN to the AddTrust External CA Root and OpenSSL is happy because it can follow the chain from logging.fakeserver.com to *.fakeserver.com to GoGetSSL RSA DV CA to The USERTRUST Network/CN=USERTrust RSA Certification Authority and lastly to the root CA of AddTrust External CA Root

### Ordering of your certificate
How do I know the order to put my certificate, my intermediate cert(s) and my root cert? The good news is that yes, there is an RFC! https://tools.ietf.org/html/rfc5280 To summarize:
Client Cert FIRST (ie YOUR CERT )
Intermediate Cert Next
Possibly another Intermediate Cert Next
Root Cert Last

The best way to do this is to cat your certificates together and MAKE SURE YOU keep your 
-----BEGIN CERTIFICATE-----\
and \
-----END CERTIFICATE-----\
and they are on their OWN new lines. Also make sure keep the line breaks in the original cert. Like the below chained cert (minus the server cert)

```
-----BEGIN CERTIFICATE-----
MIIF1zCCA7+gAwIBAgIRAJOLsI5imHtPdfmMtqUEXJYwDQYJKoZIhvcNAQEMBQAw
gYgxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpOZXcgSmVyc2V5MRQwEgYDVQQHEwtK
ZXJzZXkgQ2l0eTEeMBwGA1UEChMVVGhlIFVTRVJUUlVTVCBOZXR3b3JrMS4wLAYD
VQQDEyVVU0VSVHJ1c3QgUlNBIENlcnRpZmljYXRpb24gQXV0aG9yaXR5MB4XDTE4
MDkwNjAwMDAwMFoXDTI4MDkwNTIzNTk1OVowTDELMAkGA1UEBhMCTFYxDTALBgNV
BAcTBFJpZ2ExETAPBgNVBAoTCEdvR2V0U1NMMRswGQYDVQQDExJHb0dldFNTTCBS
U0EgRFYgQ0EwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCfwF4hD6E1
kLglXs1n2fH5vMQukCGyyD4LqLsc3pSzeh8we7njU4TB85BH5YXqcfwiH1Sf78aB
hk1FgXoAZ3EQrF49We8mnTtTPFRnMwEHLJRpY9I/+peKeAZNL0MJG5zM+9gmcSpI
OTI6p7MPela72g0pBQjwcExYLqFFVsnroEPTRRlmfTBTRi9r7rYcXwIct2VUCRmj
jR1GX13op370YjYwgGv/TeYqUWkNiEjWNskFDEfxSc0YfoBwwKdPNfp6t/5+RsFn
lgQKstmFLQbbENsdUEpzWEvZUpDC4qPvRrxEKcF0uLoZhEnxhskwXSTC64BNtc+l
VEk7/g/be8svAgMBAAGjggF1MIIBcTAfBgNVHSMEGDAWgBRTeb9aqitKz1SA4dib
wJ3ysgNmyzAdBgNVHQ4EFgQU+ftQxItnu2dk/oMhpqnOP1WEk5kwDgYDVR0PAQH/
BAQDAgGGMBIGA1UdEwEB/wQIMAYBAf8CAQAwHQYDVR0lBBYwFAYIKwYBBQUHAwEG
CCsGAQUFBwMCMCIGA1UdIAQbMBkwDQYLKwYBBAGyMQECAkAwCAYGZ4EMAQIBMFAG
A1UdHwRJMEcwRaBDoEGGP2h0dHA6Ly9jcmwudXNlcnRydXN0LmNvbS9VU0VSVHJ1
c3RSU0FDZXJ0aWZpY2F0aW9uQXV0aG9yaXR5LmNybDB2BggrBgEFBQcBAQRqMGgw
PwYIKwYBBQUHMAKGM2h0dHA6Ly9jcnQudXNlcnRydXN0LmNvbS9VU0VSVHJ1c3RS
U0FBZGRUcnVzdENBLmNydDAlBggrBgEFBQcwAYYZaHR0cDovL29jc3AudXNlcnRy
dXN0LmNvbTANBgkqhkiG9w0BAQwFAAOCAgEAXXRDKHiA5DOhNKsztwayc8qtlK4q
Vt2XNdlzXn4RyZIsC9+SBi0Xd4vGDhFx6XX4N/fnxlUjdzNN/BYY1gS1xK66Uy3p
rw9qI8X12J4er9lNNhrsvOcjB8CT8FyvFu94j3Bs427uxcSukhYbERBAIN7MpWKl
VWxT3q8GIqiEYVKa/tfWAvnOMDDSKgRwMUtggr/IE77hekQm20p7e1BuJODf1Q7c
FPt7T74m3chg+qu0xheLI6HsUFuOxc7R5SQlkFvaVY5tmswfWpY+rwhyJW+FWNbT
uNXkxR4v5KOQPWrY100/QN68/j17paKuSXNcsr56snuB/Dx+MACLBdsF35HxPadx
78vkfQ37WcVmKZtHrHJQ/QUyjxdG8fezMsh0f+puUln/O+NlsFtipve8qYa9h/K5
yD0oZN93ChWve78XrV4vCpjO75Nk5B8O9CWQqGTHbhkgvjyb9v/B+sYJqB22/NLl
R4RPvbmqDJGeEI+4u6NJ5YiLIVVsX+dyfFP8zUbSsj6J34RyCYKBbQ4L+r7k8Srs
LY51WUFP292wkFDPSDmV7XsUNTDOZoQcBh2Fycf7xFfxeA+6ERx2d8MpPPND7yS2
1dkf+SY5SdpSbAKtYmbqb9q8cZUDEImNWJFUVHBLDOrnYhGwJudE3OBXRTxNhMDm
IXnjEeWrFvAZQhk=
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIFdzCCBF+gAwIBAgIQE+oocFv07O0MNmMJgGFDNjANBgkqhkiG9w0BAQwFADBv
MQswCQYDVQQGEwJTRTEUMBIGA1UEChMLQWRkVHJ1c3QgQUIxJjAkBgNVBAsTHUFk
ZFRydXN0IEV4dGVybmFsIFRUUCBOZXR3b3JrMSIwIAYDVQQDExlBZGRUcnVzdCBF
eHRlcm5hbCBDQSBSb290MB4XDTAwMDUzMDEwNDgzOFoXDTIwMDUzMDEwNDgzOFow
gYgxCzAJBgNVBAYTAlVTMRMwEQYDVQQIEwpOZXcgSmVyc2V5MRQwEgYDVQQHEwtK
ZXJzZXkgQ2l0eTEeMBwGA1UEChMVVGhlIFVTRVJUUlVTVCBOZXR3b3JrMS4wLAYD
VQQDEyVVU0VSVHJ1c3QgUlNBIENlcnRpZmljYXRpb24gQXV0aG9yaXR5MIICIjAN
BgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAgBJlFzYOw9sIs9CsVw127c0n00yt
UINh4qogTQktZAnczomfzD2p7PbPwdzx07HWezcoEStH2jnGvDoZtF+mvX2do2NC
tnbyqTsrkfjib9DsFiCQCT7i6HTJGLSR1GJk23+jBvGIGGqQIjy8/hPwhxR79uQf
jtTkUcYRZ0YIUcuGFFQ/vDP+fmyc/xadGL1RjjWmp2bIcmfbIWax1Jt4A8BQOujM
8Ny8nkz+rwWWNR9XWrf/zvk9tyy29lTdyOcSOk2uTIq3XJq0tyA9yn8iNK5+O2hm
AUTnAU5GU5szYPeUvlM3kHND8zLDU+/bqv50TmnHa4xgk97Exwzf4TKuzJM7UXiV
Z4vuPVb+DNBpDxsP8yUmazNt925H+nND5X4OpWaxKXwyhGNVicQNwZNUMBkTrNN9
N6frXTpsNVzbQdcS2qlJC9/YgIoJk2KOtWbPJYjNhLixP6Q5D9kCnusSTJV882sF
qV4Wg8y4Z+LoE53MW4LTTLPtW//e5XOsIzstAL81VXQJSdhJWBp/kjbmUZIO8yZ9
HE0XvMnsQybQv0FfQKlERPSZ51eHnlAfV1SoPv10Yy+xUGUJ5lhCLkMaTLTwJUdZ
+gQek9QmRkpQgbLevni3/GcV4clXhB4PY9bpYrrWX1Uu6lzGKAgEJTm4Diup8kyX
HAc/DVL17e8vgg8CAwEAAaOB9DCB8TAfBgNVHSMEGDAWgBStvZh6NLQm9/rEJlTv
A73gJMtUGjAdBgNVHQ4EFgQUU3m/WqorSs9UgOHYm8Cd8rIDZsswDgYDVR0PAQH/
BAQDAgGGMA8GA1UdEwEB/wQFMAMBAf8wEQYDVR0gBAowCDAGBgRVHSAAMEQGA1Ud
HwQ9MDswOaA3oDWGM2h0dHA6Ly9jcmwudXNlcnRydXN0LmNvbS9BZGRUcnVzdEV4
dGVybmFsQ0FSb290LmNybDA1BggrBgEFBQcBAQQpMCcwJQYIKwYBBQUHMAGGGWh0
dHA6Ly9vY3NwLnVzZXJ0cnVzdC5jb20wDQYJKoZIhvcNAQEMBQADggEBAJNl9jeD
lQ9ew4IcH9Z35zyKwKoJ8OkLJvHgwmp1ocd5yblSYMgpEg7wrQPWCcR23+WmgZWn
RtqCV6mVksW2jwMibDN3wXsyF24HzloUQToFJBv2FAY7qCUkDrvMKnXduXBBP3zQ
YzYhBx9G/2CkkeFnvN4ffhkUyWNnkepnB2u0j4vAbkN9w6GAbLIevFOFfdyQoaS8
Le9Gclc1Bb+7RrtubTeZtv8jkpHGbkD4jylW6l/VXxRTrPBPYer3IsynVgviuDQf
Jtl7GQVoP7o81DgGotPmjw7jtHFtQELFhLRAlSv0ZaBIefYdgWOWnU914Ph85I6p
0fKtirOMxyHNwu8=
-----END CERTIFICATE-----
```

### Pick your cypher

### How do I know it's working locally?
To actually create a good combined cert and be able to verify it locally try using the following command:
```
openssl crl2pkcs7 -nocrl -certfile star.kiatek.com_combined.pem| openssl pkcs7 -print_certs  -noout
subject=/OU=Domain Control Validated/OU=GoGetSSL Wildcard SSL/CN=*.kiatek.com
issuer=/C=LV/L=Riga/O=GoGetSSL/CN=GoGetSSL RSA DV CA

subject=/C=LV/L=Riga/O=GoGetSSL/CN=GoGetSSL RSA DV CA
issuer=/C=US/ST=New Jersey/L=Jersey City/O=The USERTRUST Network/CN=USERTrust RSA Certification Authority

subject=/C=US/ST=New Jersey/L=Jersey City/O=The USERTRUST Network/CN=USERTrust RSA Certification Authority
issuer=/C=SE/O=AddTrust AB/OU=AddTrust External TTP Network/CN=AddTrust External CA Root
```
This of course assumes that you've created a file star.kiatek.com_combined.pem that has the server cert, intermediate cert and CA combined together in that order. You can see from the output that this is indeed true and use the command openssl s_client   -connect <your server>:<port of the service on the server> to validate that the certificate chain looks correct and validates per openssl
```
  openssl s_client   -connect logging.kiatek.com:6514
CONNECTED(00000003)
depth=3 C = SE, O = AddTrust AB, OU = AddTrust External TTP Network, CN = AddTrust External CA Root
verify return:1
depth=2 C = US, ST = New Jersey, L = Jersey City, O = The USERTRUST Network, CN = USERTrust RSA Certification Authority
verify return:1
depth=1 C = LV, L = Riga, O = GoGetSSL, CN = GoGetSSL RSA DV CA
verify return:1
depth=0 OU = Domain Control Validated, OU = GoGetSSL Wildcard SSL, CN = *.kiatek.com
verify return:1
---
Certificate chain
 0 s:/OU=Domain Control Validated/OU=GoGetSSL Wildcard SSL/CN=*.kiatek.com
   i:/C=LV/L=Riga/O=GoGetSSL/CN=GoGetSSL RSA DV CA
 1 s:/C=LV/L=Riga/O=GoGetSSL/CN=GoGetSSL RSA DV CA
   i:/C=US/ST=New Jersey/L=Jersey City/O=The USERTRUST Network/CN=USERTrust RSA Certification Authority
 2 s:/C=US/ST=New Jersey/L=Jersey City/O=The USERTRUST Network/CN=USERTrust RSA Certification Authority
   i:/C=SE/O=AddTrust AB/OU=AddTrust External TTP Network/CN=AddTrust External CA Root
---
```

## Email

## Web

## Syslog
