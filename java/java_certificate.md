
## วิธีตรวจสอบ Certificate ที่อยู่ใน Keystore (cacerts)

1. ตรวจสอบโดยการเอา O=Organization ไปหาใน cacerts (เป็นการตรวจสอบเบื้องต้น)

```bash
cd $JAVA_HOME/lib/security
```

เข้าไปที่อยู่ของไฟล์ cacerts ตามเส้นทางข้างต้น(สำหรับ linux)

```
keytool -v -list -keystore cacerts -storepass changeit |grep Owner |grep 'DigiCert Inc'
```

หลังจากนั้นทำการแสดง certificates ที่อยู่ในไฟล์ cacerts พร้อมกับทำการกรองด้วยชื่อ Organization ที่ต้องการ

```
Owner: CN=DigiCert Assured ID Root CA, OU=www.digicert.com, O=DigiCert Inc, C=US
Owner: CN=DigiCert Assured ID Root G2, OU=www.digicert.com, O=DigiCert Inc, C=US
Owner: CN=DigiCert Assured ID Root G3, OU=www.digicert.com, O=DigiCert Inc, C=US
Owner: CN=DigiCert Global Root CA, OU=www.digicert.com, O=DigiCert Inc, C=US
Owner: CN=DigiCert Global Root G2, OU=www.digicert.com, O=DigiCert Inc, C=US
Owner: CN=DigiCert Global Root G3, OU=www.digicert.com, O=DigiCert Inc, C=US
Owner: CN=DigiCert High Assurance EV Root CA, OU=www.digicert.com, O=DigiCert Inc, C=US
Owner: CN=DigiCert Trusted Root G4, OU=www.digicert.com, O=DigiCert Inc, C=US
```

2. ตรวจสอบโดยใช้ Hash SHA1 ของ certificate

```
keytool -printcert -v -file DigiCertCA.der |grep SHA1

SHA1: A0:31:C4:67:82:E6:E6:C6:62:C2:C8:7C:76:DA:9A:A6:2C:CA:BD:8E
```

นำ SHA1 ไปหาใน cacerts

```
keytool -v -list -keystore cacerts -storepass changeit |grep ‘A0:31:C4:67:82:E6:E6:C6:62:C2:C8:7C:76:DA:9A:A6:2C:CA:BD:8E’
```

หากเจอจะแสดงข้อมูล

```
SHA1: A0:31:C4:67:82:E6:E6:C6:62:C2:C8:7C:76:DA:9A:A6:2C:CA:BD:8E
```

3. ตรวจสอบโดยการเพิ่ม certificate

```
keytool -import -alias domain.test2 -file DigiCertCA.der -storetype JKS -keystore cacerts -storepass changeit
```

certificate มีอยู่ใน cacerts แล้วตอนเพิ่มมันจะแสดงข้อความแจ้งเตือนอออกมา

```
Certificate already exists in keystore under alias <domain.test>
Do you still want to add it? [no]
```