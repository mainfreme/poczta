# Konfiguracja docker-mailserver (wiele domen)

## Dodawanie domen i kont

### Metoda 1: Skrypt setup (zalecane)
Adres i hasło podawaj w cudzysłowach (obowiązkowo przy znakach specjalnych: spacja, $, !, #, itd.).

Po uruchomieniu kontenera:
  docker exec -it poczta-mailserver setup email add 'user@domena.pl' 'haslo'

Dla wielu domen powtarzaj dla każdej domeny:
  docker exec -it poczta-mailserver setup email add 'info@domena2.pl' 'hasło_ze_znakami!'

Lista kont:
  docker exec poczta-mailserver setup email list

### Metoda 2: Plik postfix-accounts.cf
Utwórz plik postfix-accounts.cf w tym katalogu. Format (jedna linia na konto):
  user@domena.pl|{SHA512-CRYPT}$6$...

Hash hasła wygeneruj np.:
  docker run --rm -it ghcr.io/docker-mailserver/docker-mailserver:latest doveadm pw -s SHA512-CRYPT

### Aliasy (opcjonalnie)
Plik postfix-virtual.cf – przekierowania (jedna linia na alias):
  alias@domena.pl user@domena.pl
  info@domena.pl user@domena.pl

Po zmianie plików zrestartuj: docker compose restart mailserver
