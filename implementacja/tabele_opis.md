# Opis tabel

## BillOfMaterials
Tabela przechowująca informacje o materiałach potrzebnych do stworzenia danego produktu
- **ProductID** (int) - identyfikator produktu (klucz obcy Products.ProductID)
- **ComponentID**
- **QuantityPerProduct**

## Complaints
- **ComplaintID**
- **OrderID**
- **ProductID**
- **EmployeeID**
- **ComplaintDate**

## ComponentCategories
- **ComponentCategoryID**
- **CategoryName**

## ComponentDeliveries
- **ComponentDeliveryID**
- **ComponentID**
- **Quantity**
- **DateOfDelivery**

## ComponentReservations
- **ComponentReservationID**
- **ComponentID**
- **OrderDetailID**
- **ProductionOrderID**
- **ReservedQuantity**

## ComponentStockRegister
- **ComponentStockEntryID**
- **ComponentID**
- **EntryDate**
- **Amount**

## ComponentStocks
- **ComponentID**
- **QuantityAvailable**
- **QuantityReserved**

## Components
- **ComponentID**
- **ComponentName**
- **ComponentCategoryID**
- **UnitOfMeasure**
- **UnitCost**

## Customers
- **CustomerID**
- **CompanyName**
- **ContactName**
- **ContactTitle**
- **IsBusiness**
- **NIP**
- **Address**
- **City**
- **Region**
- **PostalCode**
- **Country**
- **Phone**
- **Email**

## Deliveries
- **DeliveryID**
- **ParcelNumber**
- **ShippingDate**
- **EstimatedDeliveryDate**
- **ShipperID**
- **OrderID**
- **DeliveryStatus**

## Departments
- **DepartmentID**
- **DepartmentName**

## Employees
- **EmployeeID**
- **LastName**
- **FirstName**
- **Title**
- **BirthDate**
- **HireDate**
- **Address**
- **City**
- **Region**
- **PostalCode**
- **Country**
- **HomePhone**
- **DepartmentID**

## InvoiceItems
- **InvoiceItemID**
- **InvoiceID**
- **ProductID**
- **Description**
- **Quantity**
- **UnitNetPrice**
- **VatRate**
- **DiscountPercent**

## Invoices
- **InvoiceID**
- **OrderID**
- **InvoiceNumber**
- **InvoiceDate**
- **DueDate**
- **TotalNet**
- **TotalVat**
- **TotalGross**

## OrderDetails
- **OrderDetailID**
- **OrderID**
- **ProductID**
- **UnitPrice**
- **Quantity**
- **Discount**

## OrderStatus
- **StatusID**
- **StatusCategory**

## Orders
- **OrderID**
- **CustomerID**
- **EmployeeID**
- **OrderDate**
- **RequiredDate**
- **ShippedDate**
- **ShipperID**
- **Freight**
- **ShipName**
- **ShipAddress**
- **ShipCity**
- **ShipRegion**
- **ShipPostalCode**
- **ShipCountry**
- **OrderDiscountPercent**
- **StatusID**

## Payments
- **PaymentID**
- **InvoiceID**
- **PaymentDate**
- **Amount**
- **PaymentMethod**
- **PaymentStatus**

## ProductCategories
- **CategoryID**
- **CategoryName**
- **Description**
- **ProductVatRate**

## ProductStockRegister
- **ProductStockEntryID**
- **ProductID**
- **EntryDate**
- **Amount**

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
