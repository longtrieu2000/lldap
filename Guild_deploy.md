# Huong dan deploy LLDAP server bang Docker

Tai lieu nay huong dan deploy LLDAP bang Docker Compose. Cach nay dung image chinh thuc `lldap/lldap:stable`, luu du lieu vao volume Docker va cau hinh bang bien moi truong.

## 1. Yeu cau

- Da cai Docker va Docker Compose plugin.
- Server co port `17170` cho giao dien web.
- Port `3890` cho LDAP chi nen mo ra ngoai neu that su can. Neu cac ung dung khac cung chay bang Docker, nen de chung network Docker va khong publish port LDAP ra internet.

Kiem tra Docker:

```bash
docker --version
docker compose version
```

## 2. Tao thu muc deploy

Ban co the tao mot thu muc rieng de chay LLDAP:

```bash
mkdir -p ~/deploy/lldap
cd ~/deploy/lldap
```

## 3. Tao secret

LLDAP can it nhat 2 gia tri bi mat:

- `LLDAP_JWT_SECRET`: dung de ky phien dang nhap.
- `LLDAP_KEY_SEED`: dung de sinh private key bao ve mat khau.

Neu dang o trong repo source nay, co the tao secret bang script co san:

```bash
./generate_secrets.sh
```

Hoac tu tao bang OpenSSL:

```bash
openssl rand -base64 32
openssl rand -base64 32
```

Giu lai 2 gia tri nay. Khong doi `LLDAP_KEY_SEED` sau khi da co du lieu that neu ban khong hieu ro anh huong.

## 4. Tao file `.env`

Tao file `.env` trong thu muc deploy:

```env
TZ=Asia/Ho_Chi_Minh
UID=1000
GID=1000

LLDAP_JWT_SECRET=REPLACE_WITH_RANDOM_JWT_SECRET
LLDAP_KEY_SEED=REPLACE_WITH_RANDOM_KEY_SEED
LLDAP_LDAP_BASE_DN=dc=example,dc=com
LLDAP_LDAP_USER_EMAIL=admin@example.com
LLDAP_LDAP_USER_PASS=CHANGE_ME_STRONG_PASSWORD
LLDAP_HTTP_URL=http://localhost:17170
```

Can thay cac gia tri sau:

- `LLDAP_JWT_SECRET`: secret da tao o buoc 3.
- `LLDAP_KEY_SEED`: secret da tao o buoc 3.
- `LLDAP_LDAP_BASE_DN`: base DN cua he thong LDAP. Vi du domain `example.com` thi dung `dc=example,dc=com`.
- `LLDAP_LDAP_USER_EMAIL`: email cua admin mac dinh.
- `LLDAP_LDAP_USER_PASS`: mat khau admin ban dau, toi thieu 8 ky tu.
- `LLDAP_HTTP_URL`: URL public cua web UI, dac biet quan trong neu dung tinh nang reset password qua email.

Luu y: neu mat khau co ky tu `$` trong file compose dang inline, can escape thanh `$$`. Neu dat trong `.env`, van nen tranh cac ky tu de gay loi parse khi moi bat dau.

## 5. Tao `docker-compose.yml`

Tao file `docker-compose.yml`:

```yaml
services:
  lldap:
    image: lldap/lldap:stable
    container_name: lldap
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "17170:17170"
      # Chi mo LDAP ra host neu can truy cap tu may/ung dung ngoai Docker.
      # - "3890:3890"
      # Neu bat LDAPS thi publish them port nay.
      # - "6360:6360"
    volumes:
      - lldap_data:/data

volumes:
  lldap_data:
```

Neu ban muon cac service Docker khac truy cap LDAP, hay cho chung vao mot Docker network va dung dia chi:

```text
ldap://lldap:3890
```

## 6. Khoi dong LLDAP

Chay container:

```bash
docker compose up -d
```

Xem log:

```bash
docker compose logs -f lldap
```

Kiem tra trang web:

```text
http://IP_SERVER:17170
```

Tai khoan mac dinh:

- Username: `admin`
- Password: gia tri `LLDAP_LDAP_USER_PASS` trong file `.env`

## 7. Thong tin LDAP de cau hinh ung dung khac

Voi cau hinh mau `LLDAP_LDAP_BASE_DN=dc=example,dc=com`, cac gia tri hay dung la:

```text
LDAP URL: ldap://lldap:3890
LDAP URL tu may ngoai Docker: ldap://IP_SERVER:3890
Base DN: dc=example,dc=com
Admin bind DN: cn=admin,ou=people,dc=example,dc=com
People OU: ou=people,dc=example,dc=com
Groups OU: ou=groups,dc=example,dc=com
```

Nen tao mot user rieng de bind cho cac ung dung khac, roi them user do vao group `lldap_strict_readonly` thay vi dung admin.

Vi du filter theo group:

```text
(memberOf=cn=app_users,ou=groups,dc=example,dc=com)
```

## 8. Bat LDAPS tuy chon

Neu can LDAP qua TLS, mount certificate vao container va them bien moi truong:

```yaml
services:
  lldap:
    image: lldap/lldap:stable
    env_file:
      - .env
    ports:
      - "17170:17170"
      - "6360:6360"
    volumes:
      - lldap_data:/data
      - ./certs:/certs:ro
    environment:
      - LLDAP_LDAPS_OPTIONS__ENABLED=true
      - LLDAP_LDAPS_OPTIONS__CERT_FILE=/certs/fullchain.pem
      - LLDAP_LDAPS_OPTIONS__KEY_FILE=/certs/privkey.pem

volumes:
  lldap_data:
```

Sau do cac ung dung co the dung:

```text
ldaps://lldap:6360
```

## 9. Dat sau reverse proxy

Neu dat web UI sau Nginx, Caddy, Traefik hoac proxy khac:

- Chi can proxy HTTP port `17170`.
- Dat `LLDAP_HTTP_URL` thanh URL public, vi du `https://ldap.example.com`.
- Nen bat HTTPS o reverse proxy.
- Khong proxy LDAP bang reverse proxy HTTP thong thuong. LDAP port `3890`/`6360` can TCP proxy neu muon public.

Vi du `.env`:

```env
LLDAP_HTTP_URL=https://ldap.example.com
```

## 10. Backup va restore

Neu dung SQLite mac dinh, du lieu nam trong volume `/data`, bao gom database va file cau hinh.

Backup volume:

```bash
docker run --rm \
  -v lldap_lldap_data:/data:ro \
  -v "$PWD:/backup" \
  alpine \
  tar czf /backup/lldap-backup.tar.gz -C /data .
```

Restore vao volume moi:

```bash
docker compose down
docker run --rm \
  -v lldap_lldap_data:/data \
  -v "$PWD:/backup" \
  alpine \
  sh -c "cd /data && tar xzf /backup/lldap-backup.tar.gz"
docker compose up -d
```

Ten volume thuc te co the khac tuy ten thu muc deploy. Kiem tra bang:

```bash
docker volume ls | grep lldap
```

## 11. Update version

Keo image moi va restart:

```bash
docker compose pull
docker compose up -d
docker compose logs -f lldap
```

Nen backup truoc khi update production.

## 12. Build image tu source repo nay tuy chon

Neu muon build chinh source trong repo hien tai thay vi dung image chinh thuc:

```yaml
services:
  lldap:
    build:
      context: /home/longth1/workspace/lldap
    container_name: lldap
    restart: unless-stopped
    env_file:
      - .env
    ports:
      - "17170:17170"
    volumes:
      - lldap_data:/data

volumes:
  lldap_data:
```

Sau do chay:

```bash
docker compose up -d --build
```

Build tu source se lau hon va can tai dependency trong qua trinh build. Neu chi can deploy on dinh, nen dung `image: lldap/lldap:stable`.

## 13. Xu ly loi thuong gap

Neu khong dang nhap duoc admin:

```bash
docker compose logs lldap
```

Kiem tra `LLDAP_LDAP_USER_PASS` co toi thieu 8 ky tu khong. Bien nay chi dung khi tao admin lan dau. Neu da tao database roi, doi bien moi truong khong tu dong doi mat khau admin.

Neu quen mat khau admin, co the tam thoi them bien sau, restart container, dang nhap bang mat khau trong `LLDAP_LDAP_USER_PASS`, sau do bo bien nay ra:

```env
LLDAP_FORCE_LDAP_USER_PASS_RESET=true
```

Neu ung dung khac khong ket noi LDAP duoc:

- Neu ung dung cung Docker network: dung `ldap://lldap:3890`.
- Neu ung dung nam ngoai Docker: can publish port `"3890:3890"` va mo firewall tren server.
- Kiem tra Base DN, Bind DN va password.
- Tao user bind rieng va them vao group `lldap_strict_readonly`.

Neu web UI khong truy cap duoc:

- Kiem tra port `17170` da publish chua.
- Kiem tra firewall/security group cua server.
- Xem log container bang `docker compose logs -f lldap`.
