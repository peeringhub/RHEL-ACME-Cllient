# DNL command-line ACME client #

## Installation ##
------------------

Install app dependencies:

```
# dnf install curl openssl libuuid
```

Deploy the program:

```
$ tar -xvf dnl_acme_client.tar.gz
```


## Configuration ##
-------------------

ACME client uses configuration file `./acme_client.conf`:

- `server_url` - ACME server URL (https://stica.peeringhub.io/acme)
- `kid` - Any human-readable string, which can identify the client (e.g. Company Name)
- `pa_user_id` - Iconectiv login
- `pa_password` - Iconectiv password


## Certificate creation ##
--------------------------

1. Create EC private key:

```
openssl ecparam -genkey -name prime256v1 -out ./private_key.pem
```

2. Get SPC from Iconectiv (if do not have one):

If server doesn't have a white-listed at Iconectiv IP address, ACME client cannot generate SPC tokens, required to prove ownership of SP account.
In that case, you must use SPC token file, acquired from a different server, and skip this step.

```
gen_spc [--ca] {EC key path} {OCN}

- --ca - whether to create SPC for SCA or a regular SP certificate
- {EC key path} - path to EC private key, generated in the first step
- {OCN} - Iconectiv Service Provider account ID (4 alnum characters, e.g. 123H)
```

For regular SP certificate:

```
./dnl_acme_client -c ./acme_client.conf gen_spc ./private_key.pem 123H > ./123H.spc
```

For SCA certificate:

```
./dnl_acme_client -c ./acme_client.conf gen_spc --ca ./private_key.pem 123H > ./123H_CA.spc
```

3. Create a new certificate order:

```
new_order [--ca] [--spc {SPC file path}] {EC key path} {OCN} {not_before} {not_after} {subject parameters}

- --ca - (optional) if set, order SCA certificate, otherwise order a regular SP certificate
- --spc - (optional) path to SPC file, generated in step 2. If not set, a new SPC will be requested from Iconectiv
- {EC key path} - path to EC private key, generated in the first step
- {OCN} - Iconectiv Service Provider account ID (4 alnum characters, e.g. 123H)
- {not_before} - (optional) certificate start date
- {not_after} - (optional) certificate expiration date

Subject parameters:
  C={country} - mandatory (must be US)
  S={state} - optional
  L={locality} (e.g. city) - optional
  O={organization} - mandatory
  OU={organization unit} - optional
  CN={common name} - mandatory
```

For regular SP certificate:

```
./dnl_acme_client -c ./acme_client.conf --spc ./123H.spc new_order ./private_key.pem 123H 0 0 C=US S=AZ L=Phoenix O="Company Name" CN="Company Name SHAKEN 123H" > 123H.pem
```

For SCA certificate:

```
./dnl_acme_client -c ./acme_client.conf --spc ./123H_CA.spc new_order --ca ./private_key.pem 123H 0 0 C=US S=AZ L=Phoenix O="Company Name" CN="Company Name Subordinate CA 123H" > 123H_CA.pem
```

4. Review downloaded certificate:

```
openssl x509 -text -noout -in ./123H.pem

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            7e:52:58:50:e5:59:cb:19:c9:3b:dd:46:a4:85:16:9c
        Signature Algorithm: ecdsa-with-SHA256
        Issuer: C = US, O = Peeringhub Inc, OU = Certification Authorities, CN = Peeringhub Inc SHAKEN Intermediate CA 2
        Validity
            Not Before: May 25 18:18:54 2022 GMT
            Not After : May 25 18:18:54 2023 GMT
        Subject: C = US, ST = AZ, L = Phoenix, O = Company Name, CN = Company Name SHAKEN 123H
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:5f:8e:7b:f1:6b:cf:46:d1:5d:43:33:c7:66:98:
                    b6:d6:24:8a:e6:95:9f:38:27:94:a8:21:35:55:27:
                    17:29:11:5b:5b:26:ee:3e:8a:ab:6d:d4:4a:62:a2:
                    fc:e2:09:e5:44:df:31:3c:83:1a:21:b5:ff:3e:3a:
                    94:c2:33:8e:d8
                ASN1 OID: prime256v1
                NIST CURVE: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Key Identifier:
                B9:14:9E:E1:C5:2B:00:29:E3:00:D8:F3:60:FF:3F:EB:34:50:A6:E0
            X509v3 Authority Key Identifier:
                keyid:22:DE:75:3E:D4:5E:08:6A:FF:01:1C:EA:7D:E3:C7:39:53:42:97:05

            X509v3 Certificate Policies:
                Policy: 2.16.840.1.114569.1.1.1

            1.3.6.1.5.5.7.1.26:
                0.....123H
            X509v3 CRL Distribution Points:

                Full Name:
                  URI:https://authenticate-api.iconectiv.com/download/v1/crl
                CRL Issuer:
                  DirName:L = Bridgewater, ST = NJ, CN = STI-PA CRL, C = US, O = STI-PA

    Signature Algorithm: ecdsa-with-SHA256
         30:46:02:21:00:b8:8d:fa:9f:cd:c0:20:65:f4:ce:84:6e:2f:
         8d:7e:26:22:33:26:f4:3c:d4:1d:40:11:cf:ec:5f:35:ae:71:
         40:02:21:00:88:41:11:8f:cb:2a:28:a9:48:b3:1d:8a:c5:9e:
         30:52:09:e7:7c:f5:12:de:ff:37:95:d9:b9:21:7f:d6:0d:be
```

## Account management ##
------------------------

### Login and list active orders ###

```
login {EC key path}

./dnl_acme_client -c ./acme_client.conf login ./private_key.pem
```

### Change account's access key ###

1. Create new EC private key:

```
openssl ecparam -genkey -name prime256v1 -out ./new_private_key.pem
```

2. Update key on server:

```
key_change {old key path} {new key path}

./dnl_acme_client -c ./acme_client.conf key_change ./private_key.pem ./new_private_key.pem
```

### Deactivate account ###

```
deactivate {EC key path}

./dnl_acme_client -c ./acme_client.conf deactivate ./private_key.pem
```
