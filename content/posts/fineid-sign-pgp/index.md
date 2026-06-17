+++
date = '2026-06-17T23:11:03+03:00'
draft = false
title = 'My PGP key is national ID signed'
+++

Given the Jia Tan [incident with xz](https://en.wikipedia.org/wiki/XZ_Utils_backdoor) and the increasing awareness of the importance of software supply chain security, what can an individual do? I have a Finnish ID card ([FINEID](https://dvv.fi/en/fineid-specifications) or "kansalaisvarmenne") that can sign things using an RSA-2048 or P-256 key. So lets sign my commit signing key with that. This should give a better guarantee of my developer identity, though not the quality of code itself.

Here is how I set up the card in Linux:

{{< figure
  src="towi.webp"
  alt="colorful smart card reader and old phone card"
  caption="\"Fun\". The aesthetic matches my phone card."
>}}

## Smart card reader
I found a late '90s or early 2000s card reader, Towitoko Chipdrive Micro 100. It uses serial and PS/2 ports for data and power, which my PC still has. It is [not supported](https://ccid.apdu.fr/ccid/section.html) by the widely used CCID package, but has its own driver, [libtowitoko](https://github.com/cprados/towitoko-linux/), last shipped with Debian 9 (Stretch, 2017). It has since been removed for [bugs and lack of users](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1036318). Having open source drivers doesn't mean the hardware is supported forever if nobody uses or maintains it.

The driver builds and works after [some fixes](https://github.com/cprados/towitoko-linux/pull/3).

## Smart card daemon
pcscd is the standard Linux smart card daemon. It interfaces with the card reader through IFDHandler API and provides PC/SC API to applications.

If you've ever seen the weirdly generically named `/etc/reader.conf`, that's pcscd's configuration file for serial port smart card readers:
```
FRIENDLYNAME      "Towitoko Chipdrive Reader"
CHANNELID         0x000001
LIBPATH           /usr/lib/libtowitoko.so.2.0.0
```
Test the driver by reading ATR (Answer To Reset) of the card:
```
$ pcsc_scan -qn
 Reader 0: Towitoko Chipdrive Reader 00 00
  Event number: 5
  Card state: Card inserted, 
  ATR: 3B 7F 96 00 00 80 31 B8 65 B0 85 03 00 EF 12 00 F6 82 90 00
```
We can now send commands and receive responses in APDU (Application Protocol Data Unit) format. What should be sent?

## Card middleware
OpenSC supports [older](https://github.com/OpenSC/OpenSC/wiki/Finnish-FINeID) and [newer](https://github.com/OpenSC/OpenSC/pull/3505) versions of FINEID, but [not mine](https://github.com/OpenSC/OpenSC/pull/2567). Therefore I used the official FINEID middleware [Atostek ID](https://dvv.fi/en/card-reader-software) which does have Linux support (nice).

This makes me wonder why card-specific middleware is needed? Some of the commands are standardized, like `Compute digital signature` in ISO 7816-8 (Commands and mechanisms for security operations). There is also ISO 7816-15 (Cryptographic information application). Maybe the standards leave too many things unspecified or undiscoverable to have a 'universal' middleware that works with every card. Or maybe the standards exist but just aren't implemented.

As an aside, it's quite annoying that ISO standards are not freely available. Buying all 15 parts for ISO 7816 would cost over 2000 CHF. Wikipedia has links to PDFs from the Moroccan Standards Institute's Wordpress uploads folder, but that doesn't seem like an official release.

## Crypto operations API
The middleware ships a shared library `Atostek-ID-PKCS11.so` that provides a PKCS#11 API for listing keys, exporting cetificates, signing and encryption. I can export my signing key certificate with:
```
p11tool --list-tokens
p11tool --list-all-certs $token_from_previous
p11tool --export $cert_from_previous
```
There is a bug where I had to remove `object=...` from the certificate url. [pkcs11-spy](https://github.com/OpenSC/OpenSC/wiki/Using-OpenSC#pkcs-11-spy) shows that the `object=...` (here called CKA_LABEL) matched what the middleware returned elsewhere, so it shouldn't be a p11tool problem:
```
8: C_FindObjectsInit
CKA_LABEL 00005c5b54397e80 / 24
5649494E 494B4B41 2056494C 4C452031 35303133 34323531
V I I N  I K K A  . V I L  L E . 1  5 0 1 3  4 2 5 1
Returned: 19 CKR_ATTRIBUTE_VALUE_INVALID
```
```
19: C_GetAttributeValue
CKA_LABEL 00007ffe4bc76bb0 / 24
5649494E 494B4B41 2056494C 4C452031 35303133 34323531
V I I N  I K K A  . V I L  L E . 1  5 0 1 3  4 2 5 1
Returned: 0 CKR_OK
```
I reported this to DVV and Atostek and it's expected to get fixed.

## Interfacing with gpg
`gpg` doesn't directly support the PKCS#11 API, so I need `gnupg-pkcs11-scd`. The version in Ubuntu 24.04 fails to find the key when signing:
```
gpg-agent DBG: chan_13 -> KEYINFO E365578724C8D0D55F7F6A68847F080FED193A8A
gpg-agent DBG: chan_13 <- ERR 41 Wrong public key algorithm <Unspecified source>
```
The software seems to be confused by the ECC keys on the card. This [was fixed](https://github.com/alonbl/gnupg-pkcs11-scd/commit/9cbc61b06e531ce324f68fc80ec69af59c425ae8) in 2024, but the project hasn't made a new release since then, so I had to use the git version.

It's only able to use RSA keys. Create ~/.gnupg/gnupg-pkcs11-scd.conf with:
```
providers p1
provider-p1-library /usr/lib/Atostek-ID-PKCS11.so
```
And ~/.gnupg/gpg-agent.conf with
```
scdaemon-program path_to_your_compiled_gnupg-pkcs11-scd
```
Make sure running gpg daemons are restarted
```
gpgconf --kill all
```
## Generating a proxy PGP identity

In order to use the card keys with gpg, we have to attach them to a PGP identity first. I could attach one of them to my main key as a signing subkey. That would be enough proof that I own them because of [signing subkey cross-certication](https://www.gnupg.org/faq/subkey-cross-certify.html). However I think the extra subkey is kind of ugly. Lets generate a proxy identity that uses the card keys and use that to sign my existing key.

The card shows up as two tokens. PIN1 has the encryption and authentication key, PIN2 the signing key. Use the PIN2 key as the PGP master (certify) key.
```
$ gpg --full-generate-key
Please select what kind of key you want:
  (14) Existing key from card
Your selection? 14
Available keys:
   (1) A16CC8D40251356FDE09F854E4EA0A160CE0C036 VRK\x2DFINEID\x20\x28Gemalto\x29
       /MultiApp\x20v3\x2E0/541312930/HENKILOKORTTI\x20\x28PIN1\x29/45 rsa2048
   (2) E365578724C8D0D55F7F6A68847F080FED193A8A VRK\x2DFINEID\x20\x28Gemalto\x29
       /MultiApp\x20v3\x2E0/541312930/HENKILOKORTTI\x20\x28PIN2\x29/46 rsa2048
Your selection? 2
pub   rsa2048 2026-06-17 [SCEAR] [expires: 2028-05-31]
      DD16AC2A9E6A5CFF09D74BFD51264FC8D8700F85
uid                      VIINIKKA VILLE 150134251
```
Annoyingly the key usage flags are wrong. PKCS#11 does have key attributes CKA_SIGN and CKA_DECRYPT, which gnupg-pkcs11-scd could return. It seems that the pkcs11-helper library it uses doesn't expose them. They are set correctly on the card (read using pkcs11-tool):
```
PIN1: decrypt, sign, unwrap
PIN2: sign
```
There is no authenticate flag, authentication is performed with the sign operation. Wrapping seems to allow exporting an encrypted version of a key and then importing it back.

There is an `--expert` option to set the flags manually when generating the key but it doesn't work in this case. gnupg-pkcs11-scd doesn't return any key flags, so they get set to 0. The manual key capabilities only allow adding or removing flags that the underlying key reports supported, so it does nothing. Finally, there is some fallback code that enables all possible capabilities if they were set zero. That's how they end up as `[SCEAR]`. The usage can be later changed with `change-usage`.

I also add the encryption key, even if it's not needed. The combination of a low-powered card, an old card reader and all the abstraction layers causing unnecessary operations make using the card quite slow. Some operations like listing the keys can take up to 30 seconds.
```
$ gpg --edit-key 51264FC8D8700F85
gpg> change-usage
Changing usage of the primary key.
Your selection? =CS
gpg> addkey
  (14) Existing key from card
Your selection? 14
Available keys:
   (1) A16CC8D40251356FDE09F854E4EA0A160CE0C036 VRK\x2DFINEID\x20\x28Gemalto\x29
       /MultiApp\x20v3\x2E0/541312930/HENKILOKORTTI\x20\x28PIN1\x29/45 rsa2048
   (2) E365578724C8D0D55F7F6A68847F080FED193A8A VRK\x2DFINEID\x20\x28Gemalto\x29
       /MultiApp\x20v3\x2E0/541312930/HENKILOKORTTI\x20\x28PIN2\x29/46 rsa2048
Your selection? 1
...
gpg> key 1
gpg> change-usage
Changing usage of a subkey.
Your selection? =E

sec  rsa2048/51264FC8D8700F85
     created: 2026-06-17  expires: 2028-05-31  usage: SC  
     card-no: 3131 FB400D6D
     trust: ultimate      validity: ultimate
sub* rsa2048/54403ADA63E46F7A
     created: 2026-06-17  expires: 2028-05-31  usage: E   
[ultimate] (1). VIINIKKA VILLE 150134251
```
## Signing

Now do the actual signing:
```
$ gpg --local-user 'VIINIKKA VILLE 150134251' --sign-key 376E166545590E1D

pub  rsa4096/376E166545590E1D
     created: 2026-05-23  expires: 2028-05-31  usage: C   
     trust: ultimate      validity: ultimate
 Primary key fingerprint: 23A0 2503 1FD0 5687 92F7  C54E 376E 1665 4559 0E1D

This key is due to expire on 2028-05-31.
Are you sure that you want to sign this key with your
key "VIINIKKA VILLE 150134251" (51264FC8D8700F85)

Really sign? (y/N) y
```

## Verifying the signature
Anybody can verify the signature. First, download the proxy key [51264FC8D8700F85.asc](51264FC8D8700F85.asc). Import it with:
```
$ gpg --import ~/Downloads/51264FC8D8700F85.asc
```
Verify the signature:
```
$ gpg --check-sigs 376E166545590E1D
...
sig!         51264FC8D8700F85 2026-06-17  VIINIKKA VILLE 150134251
...
gpg: 5 good signatures
```
But how do you tell the `51264FC8D8700F85` key is trustworthy and not just something I generated myself? The fingerprint is a hash of more than just the public key, so it doesn't match my certificate (it also changes if I run full-generate-key again). We have to compare the raw public keys.

Get my FINEID signing key certificate [3C03D121.crt](3C03D121.crt). Read the Public key modulus:
```
$ openssl x509 -in 3C03D121.crt -noout -modulus | cut -d '=' -f 2
B2D37E993CDA9D199D54A2FA7919BD85FFB3F5B8D5A75BA29749FE70F58257473136B46FF644DBE6FEA8BC1A9E3281B012C2420E43F32C93E3B8BD2CF80148A2BA330539B913BAFA01B6AC9540F12BF96E10D6EA3345DAA2CA8E3533C50C806EFB60CC7BCBDEDE6CA1CFFE6B3A6CF9FDBA792346E94E3577DAF70C78792BC9E3CA4D9FC8455D8AC08DA7ED75BD56B3F737EABC868E51E642421F2CA1F6AD54C37A98855A3F1A809C517B90C4E7FD81CE30D53A369424A827B6E61DE299B31AB2398682181058B6FDF9A529FE0843FA1B8747A8845099E7B60AAF4AC4DC2EEE8F9CB213EE0EDAD61C62666AD9E8E3BFC51643CEDBDAAABBA2151038114CF9AFAF
```
It is the same as the PGP key modulus:
```
$ gpg --with-key-data --list-key 51264FC8D8700F85 | grep '^pkd:' \
| head -1 | cut -d ':' -f 4
B2D37E993CDA9D199D54A2FA7919BD85FFB3F5B8D5A75BA29749FE70F58257473136B46FF644DBE6FEA8BC1A9E3281B012C2420E43F32C93E3B8BD2CF80148A2BA330539B913BAFA01B6AC9540F12BF96E10D6EA3345DAA2CA8E3533C50C806EFB60CC7BCBDEDE6CA1CFFE6B3A6CF9FDBA792346E94E3577DAF70C78792BC9E3CA4D9FC8455D8AC08DA7ED75BD56B3F737EABC868E51E642421F2CA1F6AD54C37A98855A3F1A809C517B90C4E7FD81CE30D53A369424A827B6E61DE299B31AB2398682181058B6FDF9A529FE0843FA1B8747A8845099E7B60AAF4AC4DC2EEE8F9CB213EE0EDAD61C62666AD9E8E3BFC51643CEDBDAAABBA2151038114CF9AFAF
```
The gpg output format is described in [doc/DETAILS](https://github.com/gpg/gnupg/blob/master/doc/DETAILS).

## Verify 3C03D121.crt
How can you tell 3C03D121.crt is a valid certificate from a trusted authority? Download `VRK Gov. CA for Citizen Certificates - G3` and `VRK Gov. Root CA - G2` from [DVV](https://dvv.fi/en/ca-certificates) (Finnish [Digital and Population Data Services Agency](https://www.suomi.fi/organization/digital-and-population-data-services-agency/d06969b2-342b-4063-bfc8-6372e2545d81)).
```
$ openssl verify -no_check_time -verbose -show_chain \
-trusted vrkroot2c.crt -untrusted vrkcqc3.crt 3C03D121.crt
3C03D121.crt: OK
Chain:
depth=0: C = FI, serialNumber = 150134251, GN = VILLE, SN = VIINIKKA,
         CN = VIINIKKA VILLE 150134251 (untrusted)
depth=1: C = FI, O = Vaestorekisterikeskus CA, OU = Valtion kansalaisvarmenteet,
         CN = VRK Gov. CA for Citizen Certificates - G3 (untrusted)
depth=2: C = FI, O = Vaestorekisterikeskus CA, OU = Certification Authority Services,
         OU = Varmennepalvelut, CN = VRK Gov. Root CA - G2
```
First I tried `-CAfile vrkroot2c.crt` but that gives an error on Ubuntu 24.04:
```
Error loading file vrkroot2c.crt
40F7BF7509720000:error:05800088:x509 certificate routines:
X509_load_cert_crl_file_ex:no certificate or crl found:../crypto/x509/by_file.c:251:
```
Turns out old openssl versions only support `-CAfile` in PEM format, DER was [added](https://github.com/openssl/openssl/commit/57c0205b4df7d612a0333415dfc0a845c22e7458) in 3.2.0. Giving the certificate using `-trusted` supports both.

And yes, my card and the certificate are expired, hence `-no_check_time`. Maybe I'll get a new one when they support PQC. Or maybe the [EU Digital Identity Wallet](https://en.wikipedia.org/wiki/EU_Digital_Identity_Wallet) will be more convenient, even if it probably won't allow creative uses like this.

## Alternative with S/MIME
Some of the commands are not obvious, so in theory I could use similar looking flags that don't actually verify the signatures or read the the public keys from a comment field or something like that. Here is an S/MIME (or technically PKCS#7) signed [message](message_signed.p7m) of my PGP fingerprint, created with:
```
gpgsm --import 3C03D121.crt
gpgsm --faked-system-time 20250101T000000 --local-user 0x7F23D29B \
--sign -o message_signed.p7m message.txt
```
This verifies the Certificate Revocation List from `http://proxy.fineid.fi/crl/vrkcqc3c.crl` (45MB). We can also read it manually:
```
$ openssl x509 -in 3C03D121.crt -noout -serial
serial=3C03D121
$ openssl crl -in vrkcqc3c.crl -text -out vrkcqc3c.txt
$ grep 3C03D121 vrkcqc3c.txt | wc -l
0
$ grep 3C03D12. vrkcqc3c.txt | wc -l
10
$ grep -A1 --no-group-separator 'X509v3 CRL Reason Code' vrkcqc3c.txt | sort | uniq -c
 970099                 Superseded
 970099             X509v3 CRL Reason Code:
```
My certificate is not currently on the list. Looking at other serial numbers with the same initial 28 bits, 10 out of 16 have been revoked. The reason code for every revocation is "superseded".

Back to the topic, verify the message:
```
$ gpgsm -o message.txt --verify message_signed.p7m
gpgsm: Signature made 2025-01-01 00:00:19 UTC
gpgsm:                using rsa2048 key A33E952756EAE9FE3AEF184FB8D6FE117F23D29B
gpgsm: certificate has expired
gpgsm:   (expired at 2025-06-08 20:59:59)
gpgsm: Good signature from "/CN=VIINIKKA VILLE 150134251/C=FI
                            /SN=VIINIKKA/GN=VILLE/SerialNumber=150134251"
$ cat message.txt 
Hello to you from www.villeviinikka.com. My PGP key created on 2026-05-23
has fingerprint 23A025031FD0568792F7C54E376E166545590E1D.
```
Print the output separately to avoid the message potentialy overwriting signature verification output like in [gpg.fail](https://gpg.fail/).

## Conclusion
Some of the problems could have been avoided by using a usb card reader, the newer card, and a more up to date linux distro. Also, the whole idea of trusting a PGP key by the key material is tricky because it's non-standard and requires special commands to verify.

Should you trust Jia Tan more if their PGP key had a government ID signature? While it's quite hard to get a valid signature for a specific name (like a Finnish ID for Ville Viinikka), that's not required for something like the XZ attack. You can just find a stolen or lost identity card, or an infected computer with a card reader and piggyback off a legitimate transaction (why did I have to sign that pdf twice?) and use whatever name you get as a basis for the fake identity. My name can be found at [Aalto University Radio Club](https://oh2ti.fi/prkn-aiempien-vuosien-toimihenkiloita/) and [LKML](https://lkml.org/lkml/2016/7/8/515), which is hopefully enough proof that the identity I'm trying to verify is relevant.

[xkcd 1121](https://xkcd.com/1121/) describes another value of this exercise. The construction of the identity proof scheme can act as a proof itself. Although without personal familiarity this is a proof of personality, not identity.

Finally, somebody might worry about government authorized keys supplanting the PGP community key signing. I believe them to be complementary. In any case, this post is more of a demonstration and a way for me to get familiar with the technology than a guide or a call to action. Welcome to my blog.
