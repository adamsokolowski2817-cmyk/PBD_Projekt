# PBD_Projekt

Projekt realizowany w ramach przedmiotu Podstawy Baz Danych (_PBD_) na kierunku Informatyka na Wydziale Informatyki Akademii Górniczo-Hutniczej w roku akademickim 2025/2026.

## Zespół

Projekt został wykonany przez:

- [Adama Sokołowskiego](https://github.com/adamsokolowski2817-cmyk)
- [Wiktora Stokłosę](https://github.com/wikst00wa)
- [Wojciecha Bernackiego](https://github.com/carbon718)

## Polecenie - wymagania projektu

- Pełna treść polecenia jest dostępna pod odnośnikiem: [Polecenie](https://github.com/adamsokolowski2817-cmyk/PBD_Projekt/blob/main/polecenie-wymagania_projektu.pdf)

Projekt polegał na zaprojektowaniu i implementacji relacyjnej bazy danych dla firmy produkcyjno-sprzedażowej.  
Baza danych wspiera obsługę sprzedaży, planowanie produkcji, zarządzanie magazynem oraz rozliczenia finansowe.  
Projekt został wykonany w środowisku _Microsoft SQL Server_ zgodnie z zasadami normalizacji i integralności danych.  

Wymagane było zaimplementowanie mechanizmów wspierających:
- obsługę zamówień klientów i rabatów,
- planowanie i realizację produkcji,
- zarządzanie magazynem produktów i komponentów,
- rozliczenia finansowe (faktury i płatności).

Projekt musiał zawierać widoki raportowe umożliwiające analizę sprzedaży i kosztów produkcji w różnych przedziałach czasu oraz triggery i procedury realizujące logikę biznesową systemu.

## Nasze założenia

Niektóre elementy systemu nie zostały jednoznacznie określone w treści polecenia,
dlatego na potrzeby projektu przyjęto następujące założenia wynikające z zaprojektowanego
schematu bazy danych:

- Zamówienia klientów mogą zawierać wiele pozycji, a rabaty mogą być przypisane zarówno do pozycji zamówienia, jak i całego zamówienia.
- Produkty mogą być sprzedawane klientom lub planowane do produkcji w ramach zleceń produkcyjnych.
- Produkcja realizowana jest na podstawie zleceń produkcyjnych przypisanych do stanowisk roboczych.
- Produkty mogą składać się z wielu komponentów zgodnie ze strukturą Bill of Materials.
- Stany magazynowe produktów i komponentów są przechowywane jako ilości dostępne oraz zarezerwowane.
- Proces sprzedaży obejmuje faktury, płatności oraz dostawy, które są przechowywane niezależnie w bazie danych.

## Schemat ukończonej bazy
// do wstawienia

## Opis pracy nad projektem

Poniżej przedstawiono etapy realizacji projektu bazy danych, zgodnie z harmonogramem zajęć
oraz przyjętymi założeniami projektowymi.

1. **Projektowanie**
   1. Analiza wymagań systemu produkcyjno-sprzedażowego oraz identyfikacja głównych procesów biznesowych.
   2. Opracowanie struktury logicznej bazy danych oraz zaprojektowanie relacji pomiędzy encjami.
   3. Przygotowanie schematu bazy danych w narzędziu **Redgate Data Modeler**, wraz z kluczami głównymi,
      obcymi oraz warunkami integralności.

2. **Implementacja**
   1. Utworzenie bazy danych w środowisku **Microsoft SQL Server** na platformie Azure oraz implementacja tabel
      i warunków integralności. \
      Ze względów bezpieczeństwa dostęp do bazy ograniczony został do wybranych adresów IP (IP whitelisting)
   2. Przygotowanie danych testowych umożliwiających weryfikację poprawności działania bazy
      (środowisko **Azure Data Studio**). // tu możliwa zmiana
   3. Implementacja **funkcji** oraz **procedur**
   4. Implementacja **triggerów** oraz **widoków** w **Microsoft SQL Server**, wspierających
      integralność danych oraz raportowanie wymagane w projekcie.
   5. Konfiguracja **ról użytkowników**, **uprawnień dostępu** oraz przygotowanie **dokumentacji projektu**.
   6. Finalizacja projektu i przygotowanie wersji końcowej do zaliczenia.
  
   




