# 인그레스 (Ingress)

## 1. prerequisite
먼저, SSL 적용을 위해 키를 생성합니다.

- apache2 설치

- 인증서 생성
```sh
openssl genrsa -out private.key 2048
openssl rsa -in private.key -pubout -out public.key
openssl req -new -key private.key -out private.csr
openssl genrsa -aes256 -out rootCA.key 2048
openssl req -x509 -new -nodes -key rootCA.key -days 3650 -out rootCA.pem
openssl x509 -req -in private.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out private.crt -days 3650
```
> CN에 도메인 정보를 기입합니다.

- 결과

**파일** | **종류**
:---: | :---
private.key | 개인키
public.key | 공개키
private.crt | 인증서 (rootCA 인증)
rootCA.key | rootCA 개인키
rootCA.pem | rootCA 인증요청서 (CRT)

- 이동
```sh
mv * /etc/apache2/ssl/
```

- 설정파일 수정
  - 대상 파일
```sh
vi /etc/apache2/sites-available/default-ssl.conf
```
  - 수정내용
```sh
SSLCertificateFile /etc/apache2/ssl/public.key
SSLCertificateKeyFile /etc/apache2/ssl/private.pem
SSLCACertificateFile /etc/apache2/ssl/rootCA.pem
```
- 포트설정 수정
  - 대상 파일
```sh
vi /etc/apache2/ports.conf
```
  - 내용 : 80, 443으로 수정

- 테스트
```sh
curl --capath /etc/apache2/ssl/ https://test.io:443
```
> 전제 : /etc/hosts 대상 추가



kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.28.0/deploy/static/mandatory.yaml
