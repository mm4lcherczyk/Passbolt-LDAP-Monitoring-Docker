# Passbolt + LDAP + Monitoring – Środowisko Docker

Konteneryzowane środowisko testowo-developerskie zawierające Passbolt, LDAP, system monitorowania (Grafana, Prometheus, Loki, Cadvisor, Promtail), bazę danych MariaDB, reverse proxy i narzędzia developerskie (MailHog, phpLDAPadmin, Adminer).

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


## Dostęp do usług

| Usługa           | URL                              |
|------------------|-----------------------------------|
| Passbolt         | https://passbolt.local           |
| Adminer          | https://adminer.local             |
| Mailhog (Web UI) | https://mailhog.local             |
| Grafana          | https://grafana.local             |
| Prometheus       | https://prometheus.local          |
| phpLDAPadmin     | https://phpldapadmin.local        |
| cAdvisor         | https://cadvisor.local            |
| Loki API         | https://loki.local               |


> **Hasła domyślne:**  
> – Grafana: `admin` / `admin`  
> – MariaDB root: zdefiniowane w `env/mysql.env`  
> – LDAP admin: `admin` (pod `cn=admin,dc=mailhog,dc=local`)

### 🚀 Uruchomienie
1. **Pobierz i przygotuj kod źródłowy Passbolt

  ```bash
  git clone https://github.com/passbolt/passbolt_api.git
  cd passbolt_api
  cp config/app.default.php config/app.php
  docker run --rm --interactive --tty --volume $PWD:/app composer install --ignore-platform-reqs
  ```

1. **Uzupełnij `/etc/hosts`** (Linux/macOS lub Windows):

    ```text
    127.0.0.1 passbolt.local grafana.local adminer.local prometheus.local \
               mailhog.local phpldapadmin.local cadvisor.local loki.local
    ```


3. **Zbuduj oraz uruchom kontenery**:

    ```bash
    docker compose up -d
    ```

4. **Sprawdź status**:

    ```bash
    docker compose ps
    ```

5. **Utwórz pierwszego użytkownika (administratora) do Passbolt**:
    
    ```bash
    docker compose exec passbolt /bin/bash -c \
    'su -m -c "/var/www/passbolt/bin/cake passbolt register_user -u admin@passbolt.local \
    -f admin  -l admin  -r admin" -s /bin/sh www-data'
    ```
---

## Struktura katalogów

```
.
├── docker-compose-dev.yaml
├── Dockerfile
├── passbolt_api/
├── .env
├── env/
│   ├── mysql.env
│   ├── openldap.env
│   └── passbolt.env
├── README.md
├── mysql/
│   ├── init.sql
├── nginx/
│   ├── nginx.conf
│   └── certs/
├── monitoring/
│   ├── prometheus/prometheus.yml
│   ├── grafana/provisioning/dashboards.yaml
│   ├── grafana/dashboards/docker-dashboard.json
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

# LDAP – OpenLDAP
Plik: `openldap/openldap.conf`
- Serwer LDAP z TLS
- Użytkownik admin: `cn=admin,dc=mailhog,dc=local
- Hasło admin: `admin`
- Port: `389` (TLS na `636`)
- Baza danych: `dc=mailhog,dc=local`
- Schematy: `core`, `cosine`, `nis`, `inetorgperson`
- Wstępnie zdefiniowane grupy i użytkownicy:
  - Grupa `admins` z użytkownikiem `admin`
  - Grupa `users` z użytkownikami `user1`, `user2`, `user3`
- Użytkownicy mają hasła: `user1`, `user2`, `user3` (wygenerowane losowo)
- Użytkownicy mogą się logować do Passbolt i phpLDAPadmin


## Nginx – reverse proxy

Plik: `nginx/nginx.conf`

- Reverse proxy dla:
  - `grafana.local` → `grafana:3000`
  - `adminer.local` → `adminer:9501`
  - `prometheus.local` → `prometheus:9090`
  - `mailhog.local` → `mailhog:8025`
  - `phpldapadmin.local` → `phpldapadmin:8080`
  - `cadvisor.local` → `cadvisor:8081`
  - `loki.local` → `loki:3100`

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
- Dashboardy:
  - `container-monitoring.json` – monitorowanie kontenerów Docker

### Loki + Promtail
- Zbieranie logów z kontenerów Docker
- Konfiguracje w `monitoring/loki` i `promtail/`

### cAdvisor
- Monitorowanie kontenerów Docker
- Dostępne metryki: CPU, pamięć, sieć, dysk
---
