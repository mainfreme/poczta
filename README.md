# Poczta

Kontener z serwerem poczty (Postfix + Dovecot w docker-mailserver) oraz klientem webmail (Roundcube), z możliwością wielu domen. Działa za nginx-proxy.

---

## Co i jak zrobić, żeby kontener działał poprawnie i serwer odpowiadał

### 1. Wymagania

- **Docker** i **Docker Compose** (v2)
- Sieć **proxy-network** (używana przez nginx-proxy) – musi istnieć przed startem

### 2. Utworzenie sieci (jeśli nginx-proxy jeszcze jej nie utworzył)

```bash
docker network create proxy-network
```

Jeśli nginx-proxy już działa, sieć zwykle już istnieje.

### 3. Konfiguracja przed pierwszym startem

1. **Skopiuj plik środowiska i uzupełnij (opcjonalnie):**
   ```bash
   cp .env.example .env
   ```
   W `.env` ustaw np. `MAIL_HOSTNAME=mail.twojadomena.pl` – hostname, pod którym serwer poczty ma być widoczny w DNS.

2. **Katalogi na dane** – Compose tworzy je przy pierwszym `up`; jeśli chcesz mieć je w repozytorium, możesz wcześniej:
   ```bash
   mkdir -p data/mail data/state data/logs data/roundcube
   mkdir -p config/roundcube
   ```
   Dla Roundcube: jeśli `config/roundcube` jest pusty, obraz używa domyślnej konfiguracji (wystarczy do startu).

### 4. Uruchomienie kontenerów

W katalogu projektu (`poczta`):

```bash
docker compose up -d
```

Sprawdź, że kontenery działają:

```bash
docker compose ps
```

Powinny być w stanie **Up**: `poczta-mailserver` (porty 25, 465, 587, 143, 993) oraz `poczta-webmail`.

### 5. Konfiguracja nginx-proxy (dostęp do webmaila przez WWW)

Aby Roundcube był dostępny pod Twoją domeną (np. `webmail.twojadomena.pl`), dodaj w **nginx-proxy** plik vhosta, np.:

**`../nginx-proxy/nginx-proxy/conf.d/webmail.conf`:**

```nginx
server {
    listen 80;
    server_name webmail.twojadomena.pl;   # zmień na swoją domenę

    location / {
        proxy_pass http://poczta-webmail:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Następnie przeładuj konfigurację nginx-proxy:

```bash
docker exec <nazwa-kontenera-nginx-proxy> nginx -s reload
```

(lub z katalogu nginx-proxy: `docker compose exec nginx-proxy nginx -s reload`).

Dzięki temu ruch na `http://webmail.twojadomena.pl` będzie kierowany na kontener `poczta-webmail`.

### 6. DNS dla serwera poczty (żeby serwer „odpowiadał” i maile działały)

Na serwerze VPN ustaw w DNS dla **każdej domeny**, z której chcesz wysyłać/odbierać pocztę:

- **Rekord A** (np. `mail.twojadomena.pl`) → IP serwera VPN (to samo, na którym działają kontenery).
- **Rekord MX** dla domeny (np. `twojadomena.pl`) → `mail.twojadomena.pl` (lub inna nazwa z rekordu A).

Bez poprawnego MX i A serwer nie będzie poprawnie odbierał ani rozpoznawany przez inne serwery pocztowe.

### 7. Dodawanie kont pocztowych (wiele domen)

Po uruchomieniu kontenera dodaj konta przez skrypt setup. **Adres e-mail i hasło podawaj w cudzysłowach** – inaczej znaki specjalne (spacja, `$`, `!`, `#`, `&`, `*` itd.) zostaną zinterpretowane przez powłokę i polecenie może się nie udać lub ustawić złe hasło.

```bash
docker exec -it poczta-mailserver setup email add 'user@domena.pl' 'twoje_haslo'
```

Dla kolejnych domen powtarzaj z innymi adresami, np.:

```bash
docker exec -it poczta-mailserver setup email add 'info@inna-domena.pl' 'hasło_z_znakami!@#'
```

**Uwaga:** Używaj **pojedynczych** cudzysłowów `'...'` – wtedy cały ciąg jest przekazywany literalnie (np. `$` nie jest rozwijane). W cudzysłowach podwójnych `"..."` zmienne jak `$VAR` są nadal interpretowane.

Lista kont:

```bash
docker exec poczta-mailserver setup email list
```

Więcej opcji (aliasy, pliki konfiguracyjne) – patrz **`config/mailserver/README.txt`**.

### 8. Sprawdzenie, czy serwer działa i odpowiada

- **Webmail:** wejdź w przeglądarce na adres ustawiony w nginx-proxy (np. `http://webmail.twojadomena.pl`). Zaloguj się adresem e-mail i hasłem ustawionym w `setup email add`. Jeśli logowanie i lista skrzynek działają – Roundcube i połączenie z serwerem poczty są OK.

- **Porty:** na hoście sprawdź, czy porty są nasłuchiwane:
  ```bash
  ss -tlnp | grep -E '25|587|143|993'
  ```
  (albo `netstat -tlnp` w zależności od systemu).

- **Logi:** w razie problemów:
  ```bash
  docker compose logs -f mailserver
  docker compose logs -f webmail
  ```

- **Test SMTP (np. z zewnątrz):** wyślij maila na dodane konto i sprawdź, czy pojawia się w Roundcube (skrzynka Odebrane). To potwierdza, że serwer „odpowiada” na ruch z internetu.

### 9. Restart i aktualizacje

- Restart usług:
  ```bash
  docker compose restart
  ```

- Po zmianie plików w `config/mailserver` (np. `postfix-virtual.cf`):
  ```bash
  docker compose restart mailserver
  ```

- Odświeżenie obrazów i ponowne uruchomienie:
  ```bash
  docker compose pull && docker compose up -d
  ```

---

Podsumowanie: utworzysz sieć `proxy-network`, skonfigurujesz `.env` i katalogi, uruchomisz `docker compose up -d`, dodasz vhost w nginx-proxy dla webmaila, ustawisz DNS (A + MX), dodasz konta przez `setup email add` – wtedy kontener będzie działał poprawnie, a serwer będzie odpowiadał na pocztę i logowanie w Roundcube.
