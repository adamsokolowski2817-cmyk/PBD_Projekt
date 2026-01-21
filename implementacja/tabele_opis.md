# Opis tabel

## BillOfMaterials
Tabela przechowująca informacje o komponentach potrzebnych do wyprodukowania danego produktu oraz ich ilości.

- **ProductID** (int) – identyfikator produktu (klucz obcy → Products.ProductID)
- **ComponentID** (int) – identyfikator komponentu (klucz obcy → Components.ComponentID)
- **QuantityPerProduct** (decimal) – ilość komponentu potrzebna do wyprodukowania jednej sztuki produktu

---

## Complaints
Tabela przechowująca zgłoszenia reklamacyjne dotyczące zamówień i produktów.

- **ComplaintID** (int) – identyfikator reklamacji  
- **OrderID** (int) – identyfikator zamówienia, którego dotyczy reklamacja  
- **ProductID** (int) – identyfikator reklamowanego produktu  
- **EmployeeID** (int) – pracownik obsługujący reklamację  
- **ComplaintDate** (datetime) – data zgłoszenia reklamacji  

---

## ComponentCategories
Tabela przechowująca kategorie komponentów wykorzystywanych w procesie produkcji.

- **ComponentCategoryID** (int) – identyfikator kategorii komponentu  
- **CategoryName** (varchar) – nazwa kategorii  

---

## ComponentDeliveries
Tabela rejestrująca dostawy komponentów do magazynu.

- **ComponentDeliveryID** (int) – identyfikator dostawy komponentu  
- **ComponentID** (int) – identyfikator dostarczonego komponentu  
- **Quantity** (decimal) – ilość dostarczonych komponentów  
- **DateOfDelivery** (datetime) – data dostawy  

---

## ComponentReservations
Tabela przechowująca informacje o rezerwacjach komponentów na potrzeby zamówień lub produkcji.

- **ComponentReservationID** (int) – identyfikator rezerwacji  
- **ComponentID** (int) – identyfikator komponentu  
- **OrderDetailID** (int) – pozycja zamówienia powiązana z rezerwacją (opcjonalnie)  
- **ProductionOrderID** (int) – zlecenie produkcyjne powiązane z rezerwacją (opcjonalnie)  
- **ReservedQuantity** (decimal) – ilość zarezerwowanych komponentów  

---

## ComponentStockRegister
Tabela przechowująca historię wszystkich zmian stanów magazynowych komponentów.

- **ComponentStockEntryID** (int) – identyfikator wpisu magazynowego  
- **ComponentID** (int) – identyfikator komponentu  
- **EntryDate** (datetime) – data operacji magazynowej  
- **Amount** (decimal) – zmiana ilości komponentów (wartość dodatnia lub ujemna)  

---

## ComponentStocks
Tabela przechowująca aktualne stany magazynowe komponentów.

- **ComponentID** (int) – identyfikator komponentu  
- **QuantityAvailable** (decimal) – ilość dostępna w magazynie  
- **QuantityReserved** (decimal) – ilość zarezerwowana  

---

## Components
Tabela przechowująca informacje o komponentach wykorzystywanych do produkcji produktów.

- **ComponentID** (int) – identyfikator komponentu  
- **ComponentName** (varchar) – nazwa komponentu  
- **ComponentCategoryID** (int) – kategoria komponentu  
- **UnitOfMeasure** (varchar) – jednostka miary  
- **UnitCost** (decimal) – koszt jednostkowy komponentu  

---

## Customers
Tabela przechowująca dane klientów indywidualnych oraz firmowych składających zamówienia.

- **CustomerID** (int) – identyfikator klienta  
- **CompanyName** (varchar) – nazwa firmy (opcjonalnie)  
- **ContactName** (varchar) – osoba kontaktowa  
- **ContactTitle** (varchar) – stanowisko osoby kontaktowej  
- **IsBusiness** (bit) – informacja, czy klient jest firmą  
- **NIP** (varchar) – numer NIP klienta biznesowego  
- **Address** (varchar) – adres  
- **City** (varchar) – miasto  
- **Region** (varchar) – region  
- **PostalCode** (varchar) – kod pocztowy  
- **Country** (varchar) – kraj  
- **Phone** (varchar) – numer telefonu  
- **Email** (varchar) – adres e-mail  

---

## Deliveries
Tabela przechowująca informacje o dostawach realizowanych dla zamówień klientów.

- **DeliveryID** (int) – identyfikator dostawy  
- **ParcelNumber** (varchar) – numer przesyłki  
- **ShippingDate** (datetime) – data wysyłki  
- **EstimatedDeliveryDate** (datetime) – planowana data dostarczenia  
- **ShipperID** (int) – firma realizująca dostawę  
- **OrderID** (int) – identyfikator zamówienia  
- **DeliveryStatus** (varchar) – status dostawy  

## Departments
Tabela przechowująca listę działów firmy (np. produkcja, logistyka, sprzedaż).

- **DepartmentID** (int) – identyfikator działu 
- **DepartmentName** (varchar(100)) – nazwa działu
- **Address** (varchar(100)) – adres/lokalizacja działu 


## Employees
Tabela przechowująca dane pracowników oraz ich przypisanie do działu.

- **EmployeeID** (int) – identyfikator pracownika
- **LastName** (varchar(50)) – nazwisko pracownika
- **FirstName** (varchar(50)) – imię pracownika
- **Title** (varchar(50)) – stanowisko pracownika
- **BirthDate** (datetime) – data urodzenia pracownika
- **HireDate** (datetime) – data zatrudnienia pracownika
- **Address** (varchar(100)) – adres zamieszkania pracownika
- **City** (varchar(50)) – miasto zamieszkania
- **Region** (varchar(50)) – region/województwo
- **PostalCode** (varchar(20)) – kod pocztowy
- **Country** (varchar(50)) – kraj
- **HomePhone** (varchar(30)) – telefon kontaktowy pracownika
- **DepartmentID** identyfikator działu, do którego przypisany jest pracownik (powiązanie z `Departments`)

## InvoiceItems
Tabela przechowująca szczegółowe pozycje znajdujące się na fakturach sprzedaży.

- **InvoiceItemID (int)** – unikalny identyfikator pozycji faktury  
- **InvoiceID (int)** – identyfikator faktury, do której należy dana pozycja (powiązanie z tabelą `Invoices`)  
- **ProductID (int)** – identyfikator produktu objętego pozycją faktury (powiązanie z tabelą `Products`)  
- **Description (varchar)** – opis pozycji na fakturze (np. nazwa handlowa produktu lub dodatkowe informacje)  
- **Quantity (decimal)** – liczba sprzedanych sztuk produktu  
- **UnitNetPrice (decimal)** – cena jednostkowa netto produktu  
- **VatRate (float)** – stawka podatku VAT obowiązująca dla danej pozycji  
- **DiscountPercent (float)** – procentowy rabat zastosowany do pozycji

## Invoices
Tabela przechowująca podstawowe dane faktur sprzedaży wystawianych do zamówień klientów.

- **InvoiceID (int)** – identyfikator faktury (klucz główny)  
- **OrderID (int)** – identyfikator zamówienia, którego dotyczy faktura (powiązanie z tabelą `Orders`)  
- **InvoiceNumber (varchar)** – numer faktury  
- **InvoiceDate (datetime)** – data wystawienia faktury  
- **DueDate (datetime)** – termin płatności faktury  
- **TotalNet (decimal)** – łączna wartość netto faktury  
- **TotalVat (decimal)** – łączna kwota podatku VAT  
- **TotalGross (decimal)** – łączna wartość brutto faktury

## OrderDetails
Tabela przechowująca szczegółowe pozycje zamówień klientów.

- **OrderDetailID (int)** – identyfikator pozycji zamówienia  
- **OrderID (int)** – identyfikator zamówienia (powiązanie z tabelą `Orders`)  
- **ProductID (int)** – identyfikator zamówionego produktu (powiązanie z tabelą `Products`)  
- **UnitPrice (decimal)** – cena jednostkowa produktu w momencie składania zamówienia  
- **Quantity (decimal)** – liczba zamówionych sztuk produktu  
- **Discount (float)** – rabat udzielony dla danej pozycji zamówienia 

## OrderStatus
Tabela przechowująca możliwe stany realizacji zamówień klientów.

- **StatusID (int)** – identyfikator statusu zamówienia  
- **StatusCategory (varchar)** – nazwa lub kategoria statusu (np. „Nowe”, „W realizacji”, „Wysłane”, „Zakończone”, „Anulowane”)

## Orders
Tabela przechowująca zamówienia składane przez klientów.

- **OrderID (int)** – identyfikator zamówienia  
- **CustomerID (int)** – identyfikator klienta składającego zamówienie  
- **EmployeeID (int)** – identyfikator pracownika obsługującego zamówienie  
- **OrderDate (datetime)** – data złożenia zamówienia  
- **RequiredDate (datetime)** – oczekiwana data realizacji zamówienia  
- **ShippedDate (datetime)** – data wysyłki zamówienia  
- **ShipperID (int)** – identyfikator firmy przewozowej realizującej dostawę  
- **Freight (decimal)** – koszt transportu zamówienia  
- **ShipName (varchar)** – nazwa odbiorcy przesyłki  
- **ShipAddress (varchar)** – adres dostawy  
- **ShipCity (varchar)** – miasto dostawy  
- **ShipRegion (varchar)** – region/województwo dostawy  
- **ShipPostalCode (varchar)** – kod pocztowy dostawy  
- **ShipCountry (varchar)** – kraj dostawy  
- **OrderDiscountPercent (float)** – rabat procentowy zastosowany do całego zamówienia  
- **StatusID (int)** – identyfikator aktualnego statusu zamówienia (powiązanie z tabelą `OrderStatus`)

## Payments
Tabela przechowująca informacje o płatnościach dokonanych za faktury.

- **PaymentID (int)** – identyfikator płatności  
- **InvoiceID (int)** – identyfikator faktury, której dotyczy płatność (powiązanie z tabelą `Invoices`)  
- **PaymentDate (datetime)** – data dokonania płatności  
- **Amount (decimal)** – kwota zapłaty  
- **PaymentMethod (varchar)** – metoda płatności (np. przelew, karta, gotówka)  
- **PaymentStatus (varchar)** – status płatności (np. „oczekująca”, „zaksięgowana”, „odrzucona”)

## ProductCategories
Tabela kategorii produktów oferowanych przez firmę.

- **CategoryID (int)** – identyfikator kategorii produktu  
- **CategoryName (varchar)** – nazwa kategorii (np. „Stoły”, „Krzesła”, „Szafy”)  
- **Description (text)** – opis kategorii produktów  
- **ProductVatRate (float)** – domyślna stawka VAT przypisana do produktów z danej kategorii

## ProductStockRegister
Tabela rejestrująca historię zmian stanów magazynowych produktów.

- **ProductStockEntryID (int)** – identyfikator wpisu w rejestrze magazynowym  
- **ProductID (int)** – identyfikator produktu, którego dotyczy wpis (powiązanie z tabelą `Products`)  
- **EntryDate (datetime)** – data i czas wykonania operacji magazynowej  
- **Amount (decimal)** – zmiana ilości produktu (wartość dodatnia – przyjęcie, ujemna – wydanie)

## ProductStocks
Tabela pokazująca stan magazynu produktów.
- **ProductID** (int) - identyfikator produktu
- **QuantityAvailable** (decimal(12,2)) - liczba dostępnych sztuk produktu
- **QuantityReserved** (decimal(12,2)) - liczba zarezerwowanych sztuk produktu

## ProductionOrderComponents
Tabela przedstawiająca dokładną zawartość zamówienia na produkcję.
- **ProductionOrderID** (int) - identyfikator zamówienia produkcyjnego
- **ComponentID** (int) - identyfikator komponentu
- **QuantityRequired** (decimal(12,4)) - wymagana liczba sztuk materiału
- **QuantityIssued** (decimal(12,4)) - liczba wydanych sztuk materiału

## ProductionOrderStatus
Tabela określająca stany zamówień produkcyjnych.
- **ProductionOrderStatusID** (int) - identyfikator stanu zamówienia produkcyjnego
- **StatusName** (varchar(30)) - nazwa stanu

## ProductionOrders
Tabela zestawiająca obecne zamówienia produkcyjne.
- **ProductionOrderID** (int) - identyfikator zamówienia produkcyjnego
- **ProductID** (int) - identyfikator produktu
- **QuantityToProduce** (decimal(12,2)) - liczba produktów do wyprodukowania
- **PlannedStartDate** (datetime) - planowany czas rozpoczęcia realizacji zamównienia
- **PlannedEndDate** (datetime) - planowany czas zakończenia realizacji zamównienia
- **SourceOrderID** (int) - identyfikator zamówienia klienta, którego pokrycie ma zapewnić to zamówienie produkcyjne
- **WorkCenterID** (int) - identyfikator zakładu, w którym zostanie zrealizowane zamówienie
- **ProductionOrderStatusID** (int) - identyfikator stanu zamówienia produkcyjnego

## Products
Tabela przedstawiająca oferowane produkty.
- **ProductID** (int) - identyfikator produktu
- **ProductName** (varchar(100)) - nazwa produktu
- **CategoryID** (int) - identyfikator kategorii produktu
- **UnitPriceNet** (decimal(19,4)) - cena netto za dany produkt
- **ProductionTimeInHours** (int) - czas produkcji produktu w godzinach
- **ProductionCost** (decimal(19,4)) - koszt produkcji produktu

## Shippers
Tabela zestawiająca przewoźników.
- **ShipperID** (int) - identyfikator przewoźnika
- **CompanyName** (varchar(100)) - nazwa firmy spedycyjnej
- **Address** (varchar(100)) - adres firmy
- **Phone** (varchar(30)) - telefon do firmy
- **ContactEmail** (varchar(100)) - email kontaktowy firmy

## WorkCenterCapacity
Tabela opisująca zdolność produkcji danego zakładu produkcyjnego.
- **WorkCenterID** (int) - identyfikator zakładu
- **CapacityHoursPerDay** (float) - liczba godzin w ciągu dnia, kiedy zakład jest czynny (i wytwarza materiały)

## WorkCenters
Tabela zestawiająca zakłady produkcyjne.
- **WorkCenterID** (int) - identyfikator zakładu
- **WorkCenterName** (varchar(80)) - nazwa zakładu
- **IsActive** (bit) - flaga logiczna określająca, czy  zakład jest czynny

## WorkingCostParameters
Tabela zestawiająca koszty pracy w zakładach.
- **CostParametersID** (int) - identyfikator parametrów kosztów pracy w zakładach
- **LabourRatePerHour** (money) - stawka godzinowa za pracę w zakładzie
