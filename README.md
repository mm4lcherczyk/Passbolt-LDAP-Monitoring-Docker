# Passbolt + LDAP + Monitoring – Środowisko Docker

Konteneryzowane środowisko testowo-developerskie zawierające Passbolt CE, LDAP, system monitorowania (Grafana, Prometheus, Loki), bazę danych MariaDB, reverse proxy i narzędzia developerskie.

---

## Zawartość środowiska

| Usługa         | Opis                                      |
|----------------|-------------------------------------------|
| **passbolt**   | Aplikacja do zarządzania hasłami (CE)     |
| **db**         | MariaDB 10.11                             |
| **nginx**      | Reverse proxy SSL                         |
| **adminer**    | GUI do bazy danych                        |
| **mailhog**    | Przechwytywanie wiadomości e-mail         |
| **openldap**   | Serwer katalogowy LDAP z TLS              |
| **phpldapadmin** | GUI do LDAP                             |
| **grafana**    | Wizualizacja danych                       |
| **prometheus** | Zbieranie metryk                          |
| **cadvisor**   | Monitorowanie kontenerów Docker           |
| **loki**       | Zbieranie logów                           |
| **promtail**   | Forwardowanie logów do Loki               |

---

##  Wymagania

- Docker 20.10+
- Docker Compose 1.29+ lub `docker compose` (plugin)
- Min. 4 GB RAM
- Min. 10 GB wolnego miejsca

---

##  Uruchomienie środowiska

```bash
# Zbuduj obrazy
docker compose -f build

# Uruchom kontenery
docker compose -f up -d
```

> Plik `docker-compose-dev.yaml` zawiera definicje wszystkich kontenerów.

---

## Dostęp do usług

| Usługa         | URL                                     |
|----------------|------------------------------------------|
| Passbolt       | https://passbolt.local                  |
| Adminer        | https://adminer.local                   |
| Mailhog        | https://mailhog.local                 |
| Grafana        | https://grafana.local                   |
| Prometheus     | https://prometheus.local                |
| PhpLDAPAdmin   | https://phpldapadmin.local                   |

> **Uwaga:** aby domeny `*.local` działały, dodaj do pliku `/etc/hosts`:
```
127.0.0.1 passbolt.local grafana.local adminer.local prometheus.local mailhog.local phpldapadmin.local
```

---

## Struktura katalogów

```
.
├── docker-compose-dev.yaml
├── Dockerfile
├── env/
│   ├── mysql.env
│   └── passbolt.env
├── nginx/
│   ├── nginx.conf
│   └── certs/
├── monitoring/
│   ├── prometheus/prometheus.yml
│   ├── grafana/datasources/
│   ├── grafana/dashboards/
│   ├── loki/loki-config.yml
│   └── promtail/config.yml
├── conf/
│   ├── passbolt.conf
│   └── supervisor/
│       ├── *.conf
├── bin/
│   └── docker-entrypoint.sh
├── scripts/
│   └── wait-for.sh
```

---

## Dockerfile – własny obraz Passbolt CE

Budowany lokalnie z:

- `php:8.2-fpm`
- `composer:2.4` (jako stage build)
- Pobierana wersja: **v3.8.3**
- Rozszerzenia: `pdo_mysql`, `intl`, `gd`, `ldap`, `xdebug`, `gnupg`, `redis`
- Konfiguracja:
  - `nginx` jako serwer HTTP
  - `supervisor` do zarządzania procesami
  - `cron` do przypomnień e-mail

---

## Nginx – reverse proxy

Plik: `nginx/nginx.conf`

- Reverse proxy dla:
  - `grafana.local` → `grafana:3000`
  - `adminer.local` → `adminer:8080`
  - `prometheus.local` → `prometheus:9090`
- Obsługa certyfikatów SSL:
  - `/etc/nginx/certs/local.crt`
  - `/etc/nginx/certs/local.key`
- DNS Docker:
  ```nginx
  resolver 127.0.0.11 valid=30s;
  ```

---

## Monitoring

### Prometheus
- Zbiera metryki od `cadvisor`
- Konfiguracja: `monitoring/prometheus/prometheus.yml`

### Grafana
- Domyślne hasło: `admin / admin`
- Źródła danych: Prometheus, Loki

### Loki + Promtail
- Zbieranie logów z kontenerów Docker
- Konfiguracje w `monitoring/loki` i `promtail/`

---

## Debugowanie

- Logi:
```bash
docker compose logs -f <service_name>
```

- Status kontenerów:
```bash
docker compose ps -a
```
