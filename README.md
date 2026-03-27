# Домашняя работа OTUS-Angie-2025-10-14 OTUS_Angie_10_Настройка_HTTPS_для_веб-сервисов

#### Пояснение.

Я в рамках этой работы я сделал небольшой сервис, полностью описанный в [http.d/default.conf](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-14/blob/main/http.d/default.conf), который требует от клиента использование взаимной TLS аутентификации, повествует о её результате а так же включает в себя использование фичи, предложенных для реализации в домашней работе.

Кроме [http.d/default.conf](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-14/blob/main/http.d/default.conf), репозитарий включает в себя каталог [./ssl](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-14/tree/main/ssl), содержащий крипто-материалы, необходимые для воспроизведения сценария домашней работы.

### 1. Получите сертификат Let's encrypt или создайте самоподписной. Используйте домен при наличии.

Создание структуры папок
```
!--- Create folders for ssl
!
mkdir /etc/angie/ssl
mkdir /etc/angie/ssl/certs
mkdir /etc/angie/ssl/keys
mkdir /etc/angie/ssl/csrs
mkdir /etc/angie/ssl/crls
mkdir /etc/angie/ssl/for_clients
```

Создание корневого УЦ
```
openssl req -x509 -newkey rsa:4096 -noenc -days 3650 -subj "/C=RU/ST=Moscow/O=SBO Lab/CN=SBO Lab Root Certification authority" -keyout /etc/angie/ssl/keys/sbo_lab_root_ca.key   -out /etc/angie/ssl/certs/sbo_lab_root_ca.crt
```

Генерация CSR запроса для промежуточного ЦЦ
```
openssl req       -newkey rsa:2048 -noenc            -subj "/C=RU/ST=Moscow/O=SBO Lab/CN=SBO Lab Intermediate_01 Certification authority" -keyout /etc/angie/ssl/keys/sbo_lab_intermediate_01_ca.key -out /etc/angie/ssl/csrs/sbo_lab_intermediate_01_ca.csr
```

Выпуск корневым УЦ сертификата для промежуточного УЦ
```
openssl req -x509                         -days  1825 -copy_extensions copy -addext "crlDistributionPoints = URI:http://localhost/ssl/sbo_lab_root_ca.crl" -CA /etc/angie/ssl/certs/sbo_lab_root_ca.crt -CAkey /etc/angie/ssl/keys/sbo_lab_root_ca.key -in /etc/angie/ssl/csrs/sbo_lab_intermediate_01_ca.csr -out /etc/angie/ssl/certs/sbo_lab_intermediate_01_ca.crt
```

Генерация CSR запроса сервера
```
openssl req       -newkey rsa:2048 -noenc            -subj "/C=RU/ST=Moscow/O=SBO Lab/CN=common.local"  -keyout /etc/angie/ssl/keys/common.local.key  -out /etc/angie/ssl/csrs/common.local.csr
```

Выпуск промежуточным УЦ сертификата для сервера
```
openssl req -x509                         -days  365 -copy_extensions copy -addext "subjectAltName = IP:192.168.31.122,IP:192.168.129.180,DNS:common.local" -addext "crlDistributionPoints = URI:http://localhost/ssl/sbo_lab_intermediate_01_ca.crl" -CA /etc/angie/ssl/certs/sbo_lab_intermediate_01_ca.crt -CAkey /etc/angie/ssl/keys/sbo_lab_intermediate_01_ca.key -in /etc/angie/ssl/csrs/common.local.csr -out /etc/angie/ssl/certs/common.local.crt
```

Подклеивание сертификатов промежуточного и корневого УЦ в файл, который будет использоваться для валидации клиентских сертификатов
```
cat /etc/angie/ssl/certs/sbo_lab_intermediate_01_ca.crt /etc/angie/ssl/certs/sbo_lab_root_ca.crt >Ю /etc/angie/ssl/certs/trusted_cas.crt  
```

Подклеивание сертификата промежуточного УЦ в файл сертификата для сервера
```
cat /etc/angie/ssl/certs/sbo_lab_intermediate_01_ca.crt >> /etc/angie/ssl/certs/common.local.crt 
```

Генерация CSR запроса клиентов 01, 02
```
openssl req       -newkey rsa:2048 -noenc            -subj "/C=RU/ST=Moscow/O=SBO Lab/CN=client_01"     -keyout /etc/angie/ssl/for_clients/client_01.key     -out /etc/angie/ssl/for_clients/client_01.csr
openssl req       -newkey rsa:2048 -noenc            -subj "/C=RU/ST=Moscow/O=SBO Lab/CN=client_02"     -keyout /etc/angie/ssl/for_clients/client_02.key     -out /etc/angie/ssl/for_clients/client_02.csr
```

Выпуск промежуточным УЦ сертификата для клиентов 01, 02
```
openssl req -x509                         -days  365 -copy_extensions copy -addext "crlDistributionPoints = URI:http://localhost/ssl/sbo_lab_intermediate_01_ca.crl" -CA /etc/angie/ssl/certs/sbo_lab_intermediate_01_ca.crt -CAkey /etc/angie/ssl/keys/sbo_lab_intermediate_01_ca.key -in /etc/angie/ssl/for_clients/client_01.csr -out /etc/angie/ssl/for_clients/client_01.crt
openssl req -x509                         -days  365 -copy_extensions copy -addext "crlDistributionPoints = URI:http://localhost/ssl/sbo_lab_intermediate_01_ca.crl" -CA /etc/angie/ssl/certs/sbo_lab_intermediate_01_ca.crt -CAkey /etc/angie/ssl/keys/sbo_lab_intermediate_01_ca.key -in /etc/angie/ssl/for_clients/client_02.csr -out /etc/angie/ssl/for_clients/client_02.crt
```

Создание самоподписного сертификата для клиента 03
```
openssl req -x509 -newkey rsa:2048 -noenc -days  365 -subj "/C=RU/ST=Moscow/O=SBO Lab/CN=client_03"     -keyout /etc/angie/ssl/for_clients/client_03.key     -out /etc/angie/ssl/for_clients/client_03.crt
```

Другие полезные команды
```
!--- View certificate
!
openssl x509 -text         -in /etc/angie/ssl/certs/sbo_lab_root_ca.crt
openssl x509 -text         -in /etc/angie/ssl/certs/sbo_lab_intermediate_01_ca.crt

!--- View CSR request
!
openssl req  -text -verify -in /etc/angie/ssl/csrs/trustee.local.csr

! Export CA cert for being trusted
!
sudo cp /etc/angie/ssl/certs/sbo_lab_intermediate_01_ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```
**Замечание.** В папке [./ssl/for_clients](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-14/tree/main/ssl/for_clients) так же имеется крипто-пара client_01_bad.*, выпущенная иным нелегитимным УЦ для воспроизведения соответствующего сценария.

### 2. Настройте основные параметры HTTPS в Angie.
В файле [http.d/default.conf](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-14/blob/main/http.d/default.conf) в контексте **server**
```
    listen       443 ssl;
    listen       443 quic reuseport;
    listen       80;

    ssl_certificate        /etc/angie/ssl/certs/common.local.crt;
    ssl_certificate_key    /etc/angie/ssl/keys/common.local.key;
    ssl_protocols          TLSv1.2 TLSv1.3;
```
### 3. Оптимизируйте восстановление сессий.
В файле [http.d/default.conf](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-14/blob/main/http.d/default.conf) в контексте **server**
```
    ssl_session_cache      shared:SSL:10m;
    ssl_session_timeout    10m;
    ssl_session_tickets    on;
```
### 4. Включите автоматическую переадресацию с HTTP на HTTPS.
В файле [http.d/default.conf](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-14/blob/main/http.d/default.conf) в контексте **location /**
```
        if ( $scheme = http ) {
            return 301 "https://$host$uri$is_args$args";
            break;
        }
```
### 5. Настройте заголовок HSTS.
В файле [http.d/default.conf](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-14/blob/main/http.d/default.conf) в контексте **location /**
```
        add_header Strict-Transport-Security max-age=31536000;
```
### 6. Включите протоколы HTTP/2 и HTTP/3.
В файле [http.d/default.conf](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-14/blob/main/http.d/default.conf) в контексте **server**
```
    http2 on;
    http3 on;
```
### 8*. Настройте аутентификацию клиента по сертификату в Angie. Импортируйте сертификат в браузер и проверьте работоспособность конфигурации.

В файле [http.d/default.conf](https://github.com/SergeyBoonin/OTUS-Angie-2025-10-14/blob/main/http.d/default.conf) в контексте **server**
```
    ssl_client_certificate /etc/angie/ssl/certs/trusted_cas.crt;
    ssl_verify_client      optional;
```

Легитимный клиент
```
C:\curl\bin>curl --cert ssl/for_clients/client_01.crt --key ssl/for_clients/client_01.key --cacert ssl/certs/sbo_lab_root_ca.crt "https://192.168.31.122/some_path/?arga=A&argb=B"


Hello, dear 192.168.31.52:65072 :)

We agreed on TLSv1.3 using 'TLS_AES_256_GCM_SHA384'

Your certificate is VALID!

         DN | CN=client_01,O=SBO Lab,ST=Moscow,C=RU
         SN | 5FE0D46DA7711E93CAA189A9F5E9596729D4CDA8
 Valid  for | 364 days
 Valid from | Mar 26 18:43:23 2026 GMT
 Valid till | Mar 26 18:43:23 2027 GMT
Issuer's DN | CN=SBO Lab Intermediate_01 Certification authority,O=SBO Lab,ST=Moscow,C=RU

/*
 * --- THINK LIKE ANGIE DOES! BE AN ANGIE! ---
 */
```

Клиент без сертификата
```
C:\curl\bin>curl --cacert ssl/certs/sbo_lab_root_ca.crt "https://192.168.31.122/some_path/?arga=A&argb=B"


Hello, dear 192.168.31.52:55535 :)

We agreed on TLSv1.3 using 'TLS_AES_256_GCM_SHA384'

You have not used any client's certificate!

/*
 * WE WILL NOT SHARE WITH YOU OUR TOP-SECTETS BECAUSE OF HAVING NO IDEA WHO YOU ARE.
 */
```

Клиент с самоподписным сертификатом
```
C:\curl\bin>curl --cert ssl/for_clients/client_03.crt --key ssl/for_clients/client_03.key --cacert ssl/certs/sbo_lab_root_ca.crt "https://192.168.31.122/some_path/?arga=A&argb=B"


Hello, dear 192.168.31.52:54267 :)

We agreed on TLSv1.3 using 'TLS_AES_256_GCM_SHA384'

Your certificate is INVALID with the comment: 'self-signed certificate'!

         DN | CN=client_03,O=SBO Lab,ST=Moscow,C=RU
         SN | 7E458D843D7AD5F4E63719D7DC3154E085260415
 Valid  for | 364 days
 Valid from | Mar 26 16:02:48 2026 GMT
 Valid till | Mar 26 16:02:48 2027 GMT
Issuer's DN | CN=client_03,O=SBO Lab,ST=Moscow,C=RU

/*
 * WE WILL NOT SHARE WITH YOU OUR TOP SECTETS BECAUSE WE CANNOT TRUST YOUR CERTIFICATE.
 */
```

Клиент с сертификатом выпущенным недоверенным УЦ
```
C:\curl\bin>curl --cert ssl/for_clients/client_01_bad.crt --key ssl/for_clients/client_01_bad.key --cacert ssl/certs/sbo_lab_root_ca.crt "https://192.168.31.122/some_path/?arga=A&argb=B"


Hello, dear 192.168.31.52:58106 :)

We agreed on TLSv1.3 using 'TLS_AES_256_GCM_SHA384'

Your certificate is INVALID with the comment: 'unable to verify the first certificate'!

         DN | CN=client_01,O=SBO Lab,ST=Moscow,C=RU
         SN | 3BD167F0FE8D64F7ECC17F7F24C3ED324583A371
 Valid  for | 343 days
 Valid from | Mar  5 11:57:01 2026 GMT
 Valid till | Mar  5 11:57:01 2027 GMT
Issuer's DN | CN=SBO Lab Root Certification authority,O=SBO Lab,ST=Moscow,C=RU

/*
 * WE WILL NOT SHARE WITH YOU OUR TOP SECTETS BECAUSE WE CANNOT TRUST YOUR CERTIFICATE.
 */
```

### 7. Проведите тестирование корректности конфигурации с помощью внешнего сервиса.

Подтверждение наличия заголовков 'Strict-Transport-Security' и 'Alt-Svc', а так же переиспользования SSL сессии без повторной передачи сертификата после установления нового TCP соединения 
```
C:\curl\bin>curl --cert ssl/for_clients/client_01.crt --key ssl/for_clients/client_01.key -v --tls-max 1.2 --http1.1 -H "Connection: close" --cacert ssl/certs/sbo_lab_root_ca.crt "https://192.168.31.122/some_path/?arga=A1&argb=B1" "https://192.168.31.122/some_path/?arga=A2&argb=B2"
Note: Using embedded CA bundle, for proxies (227919 bytes)
*   Trying 192.168.31.122:443...
* ALPN: curl offers http/1.1
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
*  CAfile: ssl/certs/sbo_lab_root_ca.crt
*  CApath: none
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Request CERT (13):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Certificate (11):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS handshake, CERT verify (15):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384 / [blank] / UNDEF
* ALPN: server accepted http/1.1
* Server certificate:
*  subject: C=RU; ST=Moscow; O=SBO Lab; CN=common.local
*  start date: Mar 26 18:43:09 2026 GMT
*  expire date: Mar 26 18:43:09 2027 GMT
*  subjectAltName: host "192.168.31.122" matched cert's IP address!
*  issuer: C=RU; ST=Moscow; O=SBO Lab; CN=SBO Lab Intermediate_01 Certification authority
*  SSL certificate verify ok.
*   Certificate level 0: Public key type ? (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
*   Certificate level 1: Public key type ? (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
*   Certificate level 2: Public key type ? (4096/128 Bits/secBits), signed using sha256WithRSAEncryption
* Established connection to 192.168.31.122 (192.168.31.122 port 443) from 192.168.31.52 port 56944
* using HTTP/1.x
> GET /some_path/?arga=A1&argb=B1 HTTP/1.1
> Host: 192.168.31.122
> User-Agent: curl/8.16.0
> Accept: */*
> Connection: close
>
< HTTP/1.1 200 OK
< Server: Angie/1.11.4
< Date: Fri, 27 Mar 2026 10:39:52 GMT
< Content-Type: text/plain
< Content-Length: 479
< Connection: close
< Strict-Transport-Security: max-age=31536000
< Alt-Svc: h3=":443"; ma=86400
<


Hello, dear 192.168.31.52:56944 :)

We agreed on TLSv1.2 using 'ECDHE-RSA-AES256-GCM-SHA384'

Your certificate is VALID!

         DN | CN=client_01,O=SBO Lab,ST=Moscow,C=RU
         SN | 5FE0D46DA7711E93CAA189A9F5E9596729D4CDA8
 Valid  for | 364 days
 Valid from | Mar 26 18:43:23 2026 GMT
 Valid till | Mar 26 18:43:23 2027 GMT
Issuer's DN | CN=SBO Lab Intermediate_01 Certification authority,O=SBO Lab,ST=Moscow,C=RU

/*
 * --- THINK LIKE ANGIE DOES! BE AN ANGIE! ---
 */

* we are done reading and this is set to close, stop send
* abort upload
* shutting down connection #0
Note: Using embedded CA bundle, for proxies (227919 bytes)
* Hostname 192.168.31.122 was found in DNS cache
*   Trying 192.168.31.122:443...
* SSL reusing session with ALPN '-'
* ALPN: curl offers http/1.1
* TLSv1.2 (OUT), TLS handshake, Client hello (1):
*  CAfile: ssl/certs/sbo_lab_root_ca.crt
*  CApath: none
* TLSv1.2 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS change cipher, Change cipher spec (1):
												
* TLSv1.2 (IN), TLS handshake, Finished (20):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
												 
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384 / [blank] / UNDEF
* ALPN: server accepted http/1.1
* Server certificate:
*  subject: C=RU; ST=Moscow; O=SBO Lab; CN=common.local
*  start date: Mar 26 18:43:09 2026 GMT
*  expire date: Mar 26 18:43:09 2027 GMT
*  subjectAltName: host "192.168.31.122" matched cert's IP address!
*  issuer: C=RU; ST=Moscow; O=SBO Lab; CN=SBO Lab Intermediate_01 Certification authority
*  SSL certificate verify ok.
																										
																										
* Established connection to 192.168.31.122 (192.168.31.122 port 443) from 192.168.31.52 port 59994
* using HTTP/1.x
> GET /some_path/?arga=A2&argb=B2 HTTP/1.1
> Host: 192.168.31.122
> User-Agent: curl/8.16.0
> Accept: */*
> Connection: close
>
< HTTP/1.1 200 OK
< Server: Angie/1.11.4
< Date: Fri, 27 Mar 2026 10:39:54 GMT
< Content-Type: text/plain
< Content-Length: 479
< Connection: close
< Strict-Transport-Security: max-age=31536000
< Alt-Svc: h3=":443"; ma=86400
<


Hello, dear 192.168.31.52:59994 :)

We agreed on TLSv1.2 using 'ECDHE-RSA-AES256-GCM-SHA384'

Your certificate is VALID!

         DN | CN=client_01,O=SBO Lab,ST=Moscow,C=RU
         SN | 5FE0D46DA7711E93CAA189A9F5E9596729D4CDA8
 Valid  for | 364 days
 Valid from | Mar 26 18:43:23 2026 GMT
 Valid till | Mar 26 18:43:23 2027 GMT
Issuer's DN | CN=SBO Lab Intermediate_01 Certification authority,O=SBO Lab,ST=Moscow,C=RU

/*
 * --- THINK LIKE ANGIE DOES! BE AN ANGIE! ---
 */

* we are done reading and this is set to close, stop send
* abort upload
* shutting down connection #1
```

Подтверждение автоматической переадресации с HTTP на HTTPS
```
C:\curl\bin>curl -v "http://192.168.31.122/some_path/?arga=A1&argb=B1"
Note: Using embedded CA bundle, for proxies (227919 bytes)
*   Trying 192.168.31.122:80...
* Established connection to 192.168.31.122 (192.168.31.122 port 80) from 192.168.31.52 port 62226
* using HTTP/1.x
> GET /some_path/?arga=A1&argb=B1 HTTP/1.1
> Host: 192.168.31.122
> User-Agent: curl/8.16.0
> Accept: */*
>
* Request completely sent off
< HTTP/1.1 301 Moved Permanently
< Server: Angie/1.11.4
< Date: Fri, 27 Mar 2026 10:53:01 GMT
< Content-Type: text/html
< Content-Length: 169
< Connection: keep-alive
< Location: https://192.168.31.122/some_path/?arga=A1&argb=B1
< Strict-Transport-Security: max-age=31536000
<
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>Angie/1.11.4</center>
</body>
</html>
* Connection #0 to host 192.168.31.122:80 left intact
```

Подтверждение работы HTTP2
```
C:\curl\bin>curl --cert ssl/for_clients/client_01.crt --key ssl/for_clients/client_01.key -v --http2 --cacert ssl/certs/sbo_lab_root_ca.crt "https://192.168.31.122/some_path/?arga=A1&argb=B1"
Note: Using embedded CA bundle, for proxies (227919 bytes)
*   Trying 192.168.31.122:443...
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
*  CAfile: ssl/certs/sbo_lab_root_ca.crt
*  CApath: none
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Unknown (8):
* TLSv1.3 (IN), TLS handshake, Request CERT (13):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS handshake, Certificate (11):
* TLSv1.3 (OUT), TLS handshake, CERT verify (15):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / [blank] / UNDEF
* ALPN: server accepted h2
* Server certificate:
*  subject: C=RU; ST=Moscow; O=SBO Lab; CN=common.local
*  start date: Mar 26 18:43:09 2026 GMT
*  expire date: Mar 26 18:43:09 2027 GMT
*  subjectAltName: host "192.168.31.122" matched cert's IP address!
*  issuer: C=RU; ST=Moscow; O=SBO Lab; CN=SBO Lab Intermediate_01 Certification authority
*  SSL certificate verify ok.
*   Certificate level 0: Public key type ? (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
*   Certificate level 1: Public key type ? (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
*   Certificate level 2: Public key type ? (4096/128 Bits/secBits), signed using sha256WithRSAEncryption
* Established connection to 192.168.31.122 (192.168.31.122 port 443) from 192.168.31.52 port 59015
* using HTTP/2
* [HTTP/2] [1] OPENED stream for https://192.168.31.122/some_path/?arga=A1&argb=B1
* [HTTP/2] [1] [:method: GET]
* [HTTP/2] [1] [:scheme: https]
* [HTTP/2] [1] [:authority: 192.168.31.122]
* [HTTP/2] [1] [:path: /some_path/?arga=A1&argb=B1]
* [HTTP/2] [1] [user-agent: curl/8.16.0]
* [HTTP/2] [1] [accept: */*]
> GET /some_path/?arga=A1&argb=B1 HTTP/2
> Host: 192.168.31.122
> User-Agent: curl/8.16.0
> Accept: */*
>
< HTTP/2 200
< server: Angie/1.11.4
< date: Fri, 27 Mar 2026 10:56:10 GMT
< content-type: text/plain
< content-length: 474
< strict-transport-security: max-age=31536000
< alt-svc: h3=":443"; ma=86400
<


Hello, dear 192.168.31.52:59015 :)

We agreed on TLSv1.3 using 'TLS_AES_256_GCM_SHA384'

Your certificate is VALID!

         DN | CN=client_01,O=SBO Lab,ST=Moscow,C=RU
         SN | 5FE0D46DA7711E93CAA189A9F5E9596729D4CDA8
 Valid  for | 364 days
 Valid from | Mar 26 18:43:23 2026 GMT
 Valid till | Mar 26 18:43:23 2027 GMT
Issuer's DN | CN=SBO Lab Intermediate_01 Certification authority,O=SBO Lab,ST=Moscow,C=RU

/*
 * --- THINK LIKE ANGIE DOES! BE AN ANGIE! ---
 */

* abort upload
* Connection #0 to host 192.168.31.122:443 left intact
```

Подтверждение работы HTTP3
```
C:\curl\bin>curl --cert ssl/for_clients/client_01.crt --key ssl/for_clients/client_01.key -v --http3 --cacert ssl/certs/sbo_lab_root_ca.crt "https://192.168.31.122/some_path/?arga=A1&argb=B1"
Note: Using embedded CA bundle, for proxies (227919 bytes)
*   Trying 192.168.31.122:443...
*  CAfile: ssl/certs/sbo_lab_root_ca.crt
*  CApath: none
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384 / [blank] / UNDEF
* Server certificate:
*  subject: C=RU; ST=Moscow; O=SBO Lab; CN=common.local
*  start date: Mar 26 18:43:09 2026 GMT
*  expire date: Mar 26 18:43:09 2027 GMT
*  subjectAltName: host "192.168.31.122" matched cert's IP address!
*  issuer: C=RU; ST=Moscow; O=SBO Lab; CN=SBO Lab Intermediate_01 Certification authority
*  SSL certificate verify ok.
*   Certificate level 0: Public key type ? (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
*   Certificate level 1: Public key type ? (2048/112 Bits/secBits), signed using sha256WithRSAEncryption
*   Certificate level 2: Public key type ? (4096/128 Bits/secBits), signed using sha256WithRSAEncryption
* Established connection to 192.168.31.122 (192.168.31.122 port 443) from 192.168.31.52 port 55883
* using HTTP/3
* [HTTP/3] [0] OPENED stream for https://192.168.31.122/some_path/?arga=A1&argb=B1
* [HTTP/3] [0] [:method: GET]
* [HTTP/3] [0] [:scheme: https]
* [HTTP/3] [0] [:authority: 192.168.31.122]
* [HTTP/3] [0] [:path: /some_path/?arga=A1&argb=B1]
* [HTTP/3] [0] [user-agent: curl/8.16.0]
* [HTTP/3] [0] [accept: */*]
> GET /some_path/?arga=A1&argb=B1 HTTP/3
> Host: 192.168.31.122
> User-Agent: curl/8.16.0
> Accept: */*
>
* Request completely sent off
< HTTP/3 200
< server: Angie/1.11.4
< date: Fri, 27 Mar 2026 10:57:00 GMT
< content-type: text/plain
< content-length: 474
< strict-transport-security: max-age=31536000
< alt-svc: h3=":443"; ma=86400
<


Hello, dear 192.168.31.52:55883 :)

We agreed on TLSv1.3 using 'TLS_AES_256_GCM_SHA384'

Your certificate is VALID!

         DN | CN=client_01,O=SBO Lab,ST=Moscow,C=RU
         SN | 5FE0D46DA7711E93CAA189A9F5E9596729D4CDA8
 Valid  for | 364 days
 Valid from | Mar 26 18:43:23 2026 GMT
 Valid till | Mar 26 18:43:23 2027 GMT
Issuer's DN | CN=SBO Lab Intermediate_01 Certification authority,O=SBO Lab,ST=Moscow,C=RU

/*
 * --- THINK LIKE ANGIE DOES! BE AN ANGIE! ---
 */

* Connection #0 to host 192.168.31.122:443 left intact
```
```
