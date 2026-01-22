# Procedury
Poniżej przedstawiono zestaw procedur zaprojektowanych w bazie danych.
Służą one do dodawaia nowych elementów do tabel, aktualizacji wartości pól w tabelach oraz obsługi logiki biznesowej bazy.

## AddComponentToProduct
Procedura dodaje nowy składnik do produktu w tabeli BillOfMaterials po walidacji.
```sql
CREATE OR ALTER PROCEDURE [dbo].[sp_DodajSkladnikDoProduktu]
    @ProductID      INT,            -- Produkt do którego dodajemy komponent?
    @ComponentID    INT,            -- Dodawany komponent
    @Quantity       DECIMAL(12,4)   -- Ile komponentu potrzeba na 1 produkt
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. Walidacja istnienia
        IF NOT EXISTS (SELECT 1 FROM [dbo].[Products] WHERE ProductID = @ProductID)
            THROW 51049, 'Błąd: Produkt o podanym ID nie istnieje.', 1;

        IF NOT EXISTS (SELECT 1 FROM [dbo].[Components] WHERE ComponentID = @ComponentID)
            THROW 51050, 'Błąd: Komponent o podanym ID nie istnieje.', 1;

        -- 2. Walidacja ilości
        IF @Quantity <= 0
            THROW 51051, 'Błąd: Ilość składnika musi być większa od zera.', 1;

        -- 3. Sprawdzenie czy ten składnik już nie jest przypisany
        IF EXISTS (SELECT 1 FROM [dbo].[BillOfMaterials] WHERE ProductID = @ProductID AND ComponentID = @ComponentID)
        BEGIN
             THROW 51052, 'Błąd: Ten komponent jest już przypisany do tego produktu.', 1;
        END

        -- 4. Dodanie wpisu do BillOfMaterials
        INSERT INTO [dbo].[BillOfMaterials] (
            [ProductID],
            [ComponentID],
            [QuantityPerProduct]
        )
        VALUES (
            @ProductID,
            @ComponentID,
            @Quantity
        );

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## AddDepartment
Procedura dodaje nowy dział do tabeli Departments i zwraca jego ID po walidacji.
```sql
CREATE OR ALTER PROCEDURE [dbo].[AddDepartment]
    @DepartmentName VARCHAR(100),       -- Nazwa działu
    @Address        VARCHAR(100) = NULL,-- Adres działu
    @NewDepartmentID INT OUTPUT         -- Zwracamy ID nowego działu
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. Walidacja danych wejściowych

        -- Nazwa działu nie może być pusta
        IF @DepartmentName IS NULL OR LTRIM(RTRIM(@DepartmentName)) = ''
        BEGIN
            THROW 51020, 'Błąd: Nazwa działu (DepartmentName) jest wymagana.', 1;
        END

        -- 2. Sprawdzenie unikalności
        IF EXISTS (SELECT 1 FROM [dbo].[Departments] WHERE DepartmentName = @DepartmentName)
        BEGIN
            THROW 51021, 'Błąd: Dział o podanej nazwie już istnieje w strukturze firmy.', 1;
        END

        -- 3. Wstawienie rekordu
        INSERT INTO [dbo].[Departments] (
            [DepartmentName],
            [Address]
        )
        VALUES (
            @DepartmentName,
            @Address
        );

        -- 4. Pobranie ID nowo dodanego działu
        SET @NewDepartmentID = SCOPE_IDENTITY();

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## AddEmployee
Procedura dodaje nowego pracownika do tabeli Employees po walidacji danych i zwraca jego ID.
``` sql
CREATE OR ALTER PROCEDURE [dbo].[sp_DodajPracownika]
    @LastName       VARCHAR(50),         -- Nazwisko (wymagane)
    @FirstName      VARCHAR(50),         -- Imię (wymagane)
    @Title          VARCHAR(50) = NULL,  -- Stanowisko
    @BirthDate      DATETIME = NULL,     -- Data urodzenia
    @HireDate       DATETIME = NULL,     -- Data zatrudnienia
    @Address        VARCHAR(100) = NULL,
    @City           VARCHAR(50) = NULL,
    @Region         VARCHAR(50) = NULL,
    @PostalCode     VARCHAR(20) = NULL,
    @Country        VARCHAR(50) = NULL,
    @HomePhone      VARCHAR(30) = NULL,
    @DepartmentID   INT,                 -- Dział (wymagane)
    @NewEmployeeID  INT OUTPUT           -- Zwracane ID pracownika
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. WALIDACJA DANYCH

        -- Sprawdzenie imienia i nazwiska
        IF @LastName IS NULL OR @FirstName IS NULL
        BEGIN
            THROW 51022, 'Błąd: Imię i Nazwisko pracownika są wymagane.', 1;
        END

        -- Sprawdzenie czy podany dział istnieje
        IF NOT EXISTS (SELECT 1 FROM [dbo].[Departments] WHERE DepartmentID = @DepartmentID)
        BEGIN
            THROW 51023, 'Błąd: Podany dział (DepartmentID) nie istnieje.', 1;
        END

        -- Logika dat: Data zatrudnienia nie może być wcześniejsza niż urodzenia
        IF @BirthDate IS NOT NULL AND @HireDate IS NOT NULL
        BEGIN
            IF @HireDate < @BirthDate
            BEGIN
                THROW 51024, 'Błąd: Data zatrudnienia nie może być wcześniejsza niż data urodzenia.', 1;
            END
        END

        -- Domyślna data zatrudnienia = GETDATE() (jeśli nie podano)
        IF @HireDate IS NULL
            SET @HireDate = GETDATE();

        -- 2. WSTAWIENIE PRACOWNIKA
        INSERT INTO [dbo].[Employees] (
            [LastName],
            [FirstName],
            [Title],
            [BirthDate],
            [HireDate],
            [Address],
            [City],
            [Region],
            [PostalCode],
            [Country],
            [HomePhone],
            [DepartmentID]
        )
        VALUES (
            @LastName,
            @FirstName,
            @Title,
            @BirthDate,
            @HireDate,
            @Address,
            @City,
            @Region,
            @PostalCode,
            @Country,
            @HomePhone,
            @DepartmentID
        );

        -- 3. POBRANIE ID
        SET @NewEmployeeID = SCOPE_IDENTITY();

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## AddPositionToOrder
Procedura dodaje nową pozycję do zamówienia, rezerwuje dostępny towar i ustala status pozycji, zwracając jej ID.
``` sql
CREATE OR ALTER PROCEDURE [dbo].[AddPositionToOrder]
    @OrderID        INT,                -- Zamowienie, do którego dodajemy pozycję
    @ProductID      INT,                -- Dodawany produkt
    @Quantity       INT,                -- Liczba sztuk
    @Discount       DECIMAL(5,2) = 0,   -- Rabat na konkretną pozycję
    @NewDetailID    INT OUTPUT          -- Zwracane ID nowej pozycji
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. WALIDACJA PODSTAWOWA
        IF NOT EXISTS (SELECT 1 FROM [dbo].[Orders] WHERE OrderID = @OrderID)
            THROW 51012, 'Błąd: Zamówienie o podanym ID nie istnieje.', 1;

        IF NOT EXISTS (SELECT 1 FROM [dbo].[Products] WHERE ProductID = @ProductID)
            THROW 51013, 'Błąd: Produkt o podanym ID nie istnieje.', 1;

        IF @Quantity <= 0
            THROW 51014, 'Błąd: Ilość (Quantity) musi być większa od zera.', 1;

        -- 2. POBRANIE CENY PRODUKTU
        -- Cena w zamówieniu wynika z aktualnej ceny produktu (zabezpieczenie przed zmianami cen w przyszłości)
        DECLARE @CurrentUnitPrice DECIMAL(19,4);

        SELECT @CurrentUnitPrice = UnitPriceNet
        FROM [dbo].[Products]
        WHERE ProductID = @ProductID;

        -- 3. LOGIKA MAGAZYNOWA I USTALENIE STATUSU
        DECLARE @AvailableQty DECIMAL(12,2);
        DECLARE @StatusPozycji VARCHAR(30);

        -- Sprawdzenie stanu magazynoego w tabeli ProductStocks
        SELECT @AvailableQty = QuantityAvailable
        FROM [dbo].[ProductStocks]
        WHERE ProductID = @ProductID;

        -- Jeśli nie ma rekordu w magazynie, to dostępna ilość = 0
        SET @AvailableQty = ISNULL(@AvailableQty, 0);

        IF @AvailableQty >= @Quantity
        BEGIN
            -- SCENARIUSZ A: JEST TOWAR
            -- Dokonanie rezerwacji towaru
            UPDATE [dbo].[ProductStocks]
            SET QuantityAvailable = QuantityAvailable - @Quantity,
                QuantityReserved = ISNULL(QuantityReserved, 0) + @Quantity
            WHERE ProductID = @ProductID;

            SET @StatusPozycji = 'Zarezerwowano';
        END        ELSE
        BEGIN
            -- SCENARIUSZ B: BRAK TOWARU (lub za mało)
            -- Oznaczenie pozycję jako wymagającą produkcji.

            SET @StatusPozycji = 'Do Produkcji';
        END

        -- 4. WSTAWIENIE POZYCJI ZAMÓWIENIA
        INSERT INTO [dbo].[OrderDetails] (
            [OrderID],
            [ProductID],
            [UnitPrice],    -- Cena pobrana z tabeli Products
            [Quantity],
            [Discount],
            [Status]        -- Status wynikający z logiki magazynowej
        )
        VALUES (
            @OrderID,
            @ProductID,
            @CurrentUnitPrice,
            @Quantity,
            @Discount,
            @StatusPozycji
        );

        -- 5. Pobranie ID nowej pozycji
        SET @NewDetailID = SCOPE_IDENTITY();

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## AddProductCategory
Procedura dodaje nową kategorię produktu do tabeli ProductCategories po walidacji i zwraca jej ID.
``` sql
CREATE OR ALTER PROCEDURE [dbo].[sp_DodajKategorieProduktu]
    @CategoryName   VARCHAR(50),         -- Nazwa kategorii (Wymagana)
    @Description    VARCHAR(MAX) = NULL, -- Opis kategorii
    @VATPercentage  DECIMAL(5,2) = 23.00,-- Domyślny VAT dla tej kategorii (np. 23.00)
    @NewCategoryID  INT OUTPUT           -- Zwracane ID nowej kategorii
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. WALIDACJA DANYCH

        -- Nazwa kategorii  (wymagana)
        IF @CategoryName IS NULL OR LTRIM(RTRIM(@CategoryName)) = ''
        BEGIN
            THROW 51029, 'Błąd: Nazwa kategorii (CategoryName) jest wymagana.', 1;
        END

        -- Walidacja stawki VAT (musi być logiczna, (0-100%)
        IF @VATPercentage < 0.00 OR @VATPercentage > 100.00
        BEGIN
            THROW 51030, 'Błąd: Stawka VAT musi mieścić się w przedziale 0-100.', 1;
        END

        -- 2. SPRAWDZENIE DUPLIKATÓW
        IF EXISTS (SELECT 1 FROM [dbo].[ProductCategories] WHERE CategoryName = @CategoryName)
        BEGIN
            THROW 51031, 'Błąd: Kategoria o podanej nazwie już istnieje.', 1;
        END

        -- 3. WSTAWIENIE REKORDU
        INSERT INTO [dbo].[ProductCategories] (
            [CategoryName],
            [Description],
            [VATPercentage]
        )
        VALUES (
            @CategoryName,
            @Description,
            @VATPercentage
        );

        -- 4. POBRANIE ID
        SET @NewCategoryID = SCOPE_IDENTITY();

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## CompleteProductionOrder
Procedura finalizuje zlecenie produkcyjne: zmienia jego status na „Zakończone”, zużywa komponenty z magazynu, usuwa ich rezerwacje, dodaje wyprodukowany produkt do stanu magazynowego i rejestruje wszystkie zmiany w historii magazynowej.
``` sql
/*
 Finalizacja zlecenia produkcyjnego.
 Po zakończeniu produkcji procedura
 - Zmienia status w ProductionOrders na "Zakończone".
 - Zdejmuje rezerwację z komponentów i trwale usunąć je ze stanu (ComponentStocks).
 - Dodać wyprodukowany produkt do magazynu (ProductStocks -> QuantityAvailable).
 - Dodać wpis do historii zmian magazynowych (ProductStockRegister).
 */

CREATE OR ALTER PROCEDURE CompleteProductionOrder
    @ProductionOrderID int
AS
BEGIN
    SET NOCOUNT ON;

    -- Deklaracje zmiennych
    DECLARE @ProductID int;
    DECLARE @QtyProduced decimal(12,2);
    DECLARE @CurrentStatusID int;
    DECLARE @CompletedStatusID int;
    DECLARE @WorkCenterID int;

    -- 1. Sprawdzenie czy zlecenie istnieje
    SELECT
        @ProductID = ProductID,
        @QtyProduced = QuantityToProduce,
        @CurrentStatusID = ProductionOrderStatusID,
        @WorkCenterID = WorkCenterID
    FROM ProductionOrders
    WHERE ProductionOrderID = @ProductionOrderID;

    IF @ProductID IS NULL
    BEGIN
        PRINT 'Błąd: Nie znaleziono zlecenia produkcyjnego o ID: ' + CAST(@ProductionOrderID AS VARCHAR);
        RETURN;
    END

    -- Pobranie ID statusu 'Zakończone'
    SELECT TOP 1 @CompletedStatusID = ProductionOrderStatusID
    FROM ProductionOrderStatus
    WHERE StatusName LIKE 'Zakończone%' OR StatusName LIKE 'Wykonane%';

    -- Zabezpieczenie na wypadek braku statusu w słowniku
    IF @CompletedStatusID IS NULL SET @CompletedStatusID = 3; -- 3 -> domyślne zakończone

    IF @CurrentStatusID = @CompletedStatusID
    BEGIN
        PRINT 'Informacja: To zlecenie jest już zakończone.';
        RETURN;
    END

    BEGIN TRANSACTION;

    BEGIN TRY
        -- 2. Aktualizacja statusu zlecenia
        UPDATE ProductionOrders
        SET ProductionOrderStatusID = @CompletedStatusID
        WHERE ProductionOrderID = @ProductionOrderID;

        -- 3. (Zużycie materiałów)
        -- A. Rejestracja historii magazynowej dla komponentów (Wartości ujemne - zużycie)
        INSERT INTO ComponentStockRegister (ComponentID, EntryDate, Amount)
        SELECT
            ComponentID,
            GETDATE(),
            -QuantityRequired
        FROM ProductionOrderComponents
        WHERE ProductionOrderID = @ProductionOrderID;

        -- B. Aktualizacja stanów magazynowych
        UPDATE S
        SET
            S.QuantityAvailable = S.QuantityAvailable - POC.QuantityRequired,
            S.QuantityReserved = S.QuantityReserved - POC.QuantityRequired
        FROM ComponentStocks S
        INNER JOIN ProductionOrderComponents POC ON S.ComponentID = POC.ComponentID
        WHERE POC.ProductionOrderID = @ProductionOrderID;

        -- C. Usunięcie rezerwacji z tabeli ComponentReservations
        DELETE FROM ComponentReservations
        WHERE ProductionOrderID = @ProductionOrderID;

        -- D. Zaktualizowanie ilości wydanej w zleceniu (QuantityIssued)
        UPDATE ProductionOrderComponents
        SET QuantityIssued = QuantityRequired
        WHERE ProductionOrderID = @ProductionOrderID;

        -- 4. Przyjęcie wyrobu gotowego na magazyn

        -- A. Aktualizacja stanu ProductStocks
        IF EXISTS (SELECT 1 FROM ProductStocks WHERE ProductID = @ProductID)
        BEGIN
            UPDATE ProductStocks
            SET QuantityAvailable = QuantityAvailable + @QtyProduced
            WHERE ProductID = @ProductID;
        END
        ELSE
        BEGIN
            INSERT INTO ProductStocks (ProductID, QuantityAvailable, QuantityReserved)
            VALUES (@ProductID, @QtyProduced, 0);
        END

        -- B. Rejestracja w historii ProductStockRegister
        INSERT INTO ProductStockRegister (ProductID, EntryDate, Amount)
        VALUES (@ProductID, GETDATE(), @QtyProduced);

        -- Zatwierdzenie transakcji
        COMMIT TRANSACTION;

        PRINT 'Zlecenie produkcyjne nr ' + CAST(@ProductionOrderID AS VARCHAR) + ' zostało zakończone pomyślnie.';
        PRINT 'Wyprodukowano ' + CAST(@QtyProduced AS VARCHAR) + ' szt. produktu ID ' + CAST(@ProductID AS VARCHAR) + '.';

    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        PRINT 'Błąd podczas finalizacji produkcji: ' + ERROR_MESSAGE();
    END CATCH
END;
GO
```

## GenerateInvoice
Procedura generuje fakturę dla zamówienia: oblicza kwoty netto, VAT i brutto, tworzy nagłówek faktury oraz pozycje faktury i zwraca jej ID.
``` sql
CREATE OR ALTER PROCEDURE [dbo].[GenerateInvoice]
    @OrderID        INT,                -- Numer zamówienia
    @PaymentDays    INT = 14,           -- Termin płatności (domyślnie 14 dni)
    @NewInvoiceID   INT OUTPUT          -- ID nowej faktury
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 0. DEKLARACJA ZMIENNYCH AGREGACYJNYCH
        DECLARE @TotalNet DECIMAL(19,4);
        DECLARE @TotalVat DECIMAL(19,4);
        DECLARE @TotalGross DECIMAL(19,4);

        -- 1. WALIDACJA WSTĘPNA
        IF NOT EXISTS (SELECT 1 FROM [dbo].[Orders] WHERE OrderID = @OrderID)
            THROW 51025, 'Błąd: Zamówienie o podanym ID nie istnieje.', 1;

        IF NOT EXISTS (SELECT 1 FROM [dbo].[OrderDetails] WHERE OrderID = @OrderID)
            THROW 51026, 'Błąd: Zamówienie jest puste.', 1;

        IF EXISTS (SELECT 1 FROM [dbo].[Invoices] WHERE OrderID = @OrderID)
            THROW 51027, 'Błąd: Do tego zamówienia wystawiono już fakturę.', 1;

        -- 2. OBLICZENIA WARTOŚCI (Z Dynamicznym VAT)
        ;WITH OrderCalculations AS (
            SELECT
                OD.ProductID,
                P.ProductName,
                OD.Quantity,
                OD.UnitPrice AS UnitNetPrice,
                ISNULL(OD.Discount, 0) AS DiscountPercent,

                -- POBIERANIE VAT Z TABELI KATEGORII
                ISNULL(PC.VATPercentage, 23.00) AS VatRate
            FROM [dbo].[OrderDetails] OD
            JOIN [dbo].[Products] P ON OD.ProductID = P.ProductID
            -- Dołączamy kategorie, aby pobrać stawkę VAT
            LEFT JOIN [dbo].[ProductCategories] PC ON P.CategoryID = PC.CategoryID
            WHERE OD.OrderID = @OrderID
        ),
        SummaryCalculations AS (
            SELECT
                ProductID,
                ProductName,
                Quantity,
                UnitNetPrice,
                DiscountPercent,
                VatRate,
                -- Wartość Netto (po rabacie)
                (Quantity * UnitNetPrice * (1.0 - DiscountPercent/100.0)) AS LineNetTotal,

                -- Wartość VAT: Netto * (Stawka / 100)
                ((Quantity * UnitNetPrice * (1.0 - DiscountPercent/100.0)) * (VatRate / 100.0)) AS LineVatTotal
            FROM OrderCalculations
        )
        -- 3. PRZYPISANIE DO ZMIENNYCH
        SELECT
            @TotalNet = SUM(LineNetTotal),
            @TotalVat = SUM(LineVatTotal),
            @TotalGross = SUM(LineNetTotal + LineVatTotal)
        FROM SummaryCalculations;

        -- 4. GENEROWANIE NUMERU FAKTURY
        DECLARE @InvoiceNumber VARCHAR(50);
        SET @InvoiceNumber = 'FV/' + FORMAT(GETDATE(), 'yyyy/MM') + '/' + CAST(@OrderID AS VARCHAR(10));

        IF EXISTS (SELECT 1 FROM [dbo].[Invoices] WHERE InvoiceNumber = @InvoiceNumber)
            THROW 51028, 'Błąd: Wygenerowany numer faktury już istnieje.', 1;

        -- 5. WSTAWIENIE NAGŁÓWKA FAKTURY
        INSERT INTO [dbo].[Invoices] (
            [OrderID],
            [InvoiceNumber],
            [InvoiceDate],
            [DueDate],
            [TotalNet],
            [TotalVat],
            [TotalGross]
        )
        VALUES (
            @OrderID,
            @InvoiceNumber,
            GETDATE(),
            DATEADD(DAY, @PaymentDays, GETDATE()),
            @TotalNet,
            @TotalVat,
            @TotalGross
        );

        SET @NewInvoiceID = SCOPE_IDENTITY();

        -- 6. WSTAWIENIE POZYCJI FAKTURY (InvoiceItems)
        INSERT INTO [dbo].[InvoiceItems] (
            [InvoiceID],
            [ProductID],
            [Description],
            [Quantity],
            [UnitNetPrice],
            [VatRate], 
            [DiscountPercent]
        )
        SELECT
            @NewInvoiceID,
            P.ProductID,
            P.ProductName,
            OD.Quantity,
            OD.UnitPrice,
            ISNULL(PC.VAT, 23.00),
            ISNULL(OD.Discount, 0)
        FROM [dbo].[OrderDetails] OD
        JOIN [dbo].[Products] P ON OD.ProductID = P.ProductID
        LEFT JOIN [dbo].[ProductCategories] PC ON P.CategoryID = PC.CategoryID
        WHERE OD.OrderID = @OrderID;

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## NewComplaint
Procedura rejestruje nową reklamację dla produktu w zamówieniu, przypisuje ją pracownikowi i zwraca ID reklamacji.
``` sql
CREATE OR ALTER PROCEDURE [dbo].[sp_ZglosReklamacje]
    @OrderID        INT,            -- Numer reklamowanego zamówienia
    @ProductID      INT,            -- Reklamowany produkt
    @EmployeeID     INT,            -- Przypisany pracownik
    @Status         VARCHAR(30) = 'Zgłoszone', -- Domyślny status
    @NewComplaintID INT OUTPUT      -- Zwracane ID reklamacji
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. WALIDACJA DANYCH

        IF NOT EXISTS (SELECT 1 FROM [dbo].[Orders] WHERE OrderID = @OrderID)
            THROW 51032, 'Błąd: Zamówienie o podanym ID nie istnieje.', 1;

        -- Sprawdzenie Czy produkt istnieje w reklamowanym zamówieniu
        IF NOT EXISTS (SELECT 1 FROM [dbo].[OrderDetails] WHERE OrderID = @OrderID AND ProductID = @ProductID)
            THROW 51033, 'Błąd: Podany produkt nie znajduje się w tym zamówieniu.', 1;

        -- Sprawdzenie czy pracownik istnieje
        IF NOT EXISTS (SELECT 1 FROM [dbo].[Employees] WHERE EmployeeID = @EmployeeID)
            THROW 51034, 'Błąd: Pracownik o podanym ID nie istnieje.', 1;

        -- 2. REJESTRACJA REKLAMACJI
        INSERT INTO [dbo].[Complaints] (
            [OrderID],
            [ProductID],
            [EmployeeID],
            [ComplaintDate],
            [Status]
        )
        VALUES (
            @OrderID,
            @ProductID,
            @EmployeeID,
            GETDATE(),
            @Status
        );

        SET @NewComplaintID = SCOPE_IDENTITY();

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## NewComponent
Procedura dodaje nowy komponent do systemu, inicjalizuje jego stan magazynowy i zwraca jego ID.
``` sql
CREATE OR ALTER PROCEDURE [dbo].[NewComponent]
    @ComponentName      VARCHAR(100),
    @CategoryID         INT,
    @UnitOfMeasure      VARCHAR(20) = 'szt', -- np. szt, kg, m, m2
    @UnitCost           DECIMAL(19,4),       -- Koszt zakupu jednej jednostki
    @NewComponentID     INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. Walidacja
        IF NOT EXISTS (SELECT 1 FROM [dbo].[ComponentCategories] WHERE ComponentCategoryID = @CategoryID)
            THROW 51047, 'Błąd: Podana kategoria komponentu nie istnieje.', 1;

        IF @UnitCost <= 0
            THROW 51048, 'Błąd: Koszt jednostkowy (UnitCost) musi być większy od zera.', 1;

        -- 2. Dodanie Komponentu
        INSERT INTO [dbo].[Components] (
            [ComponentName],
            [ComponentCategoryID],
            [UnitOfMeasure],
            [UnitCost]
        )
        VALUES (
            @ComponentName,
            @CategoryID,
            @UnitOfMeasure,
            @UnitCost
        );

        SET @NewComponentID = SCOPE_IDENTITY();

        -- 3. Inicjalizacja Magazynu (ComponentStocks)
        INSERT INTO [dbo].[ComponentStocks] (
            [ComponentID],
            [QuantityAvailable],
            [QuantityReserved]
        )
        VALUES (
            @NewComponentID,
            0.0000,
            0.0000
        );

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## NewComponentCategories
Procedura tworzy nową kategorię komponentu w tabeli ComponentCategories i zwraca jej ID.
``` sql
CREATE OR ALTER PROCEDURE [dbo].[NewComponentCategory]
    @CategoryName   VARCHAR(50),
    @NewCategoryID  INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;

        IF @CategoryName IS NULL OR LTRIM(RTRIM(@CategoryName)) = ''
            THROW 51045, 'Błąd: Nazwa kategorii komponentu jest wymagana.', 1;

        IF EXISTS (SELECT 1 FROM [dbo].[ComponentCategories] WHERE CategoryName = @CategoryName)
            THROW 51046, 'Błąd: Taka kategoria komponentu już istnieje.', 1;

        INSERT INTO [dbo].[ComponentCategories] (CategoryName)
        VALUES (@CategoryName);

        SET @NewCategoryID = SCOPE_IDENTITY();

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## NewOrder
Procedura tworzy nowe zamówienie w tabeli Orders, ustawia domyślny status „Nowe”, uzupełnia dane dostawy i zwraca ID zamówienia.
``` sql
CREATE OR ALTER PROCEDURE [dbo].[NewOrder]
    @CustomerID      INT,                 -- Customer (Wymagane)
    @EmployeeID      INT,                 -- Employee (Wymagane)
    @RequiredDate    DATETIME = NULL,     -- Przewidywana data realizacji
    @ShipperID       INT = NULL,          -- Przewoźnik (NULL, jeśli wybierany później)
    @Freight         DECIMAL(19,4) = 0,   -- Koszt dostawy
    @ShipName        VARCHAR(100) = NULL, -- Nazwa odbiorcy (opcjonalne)
    @ShipAddress     VARCHAR(100) = NULL, -- Adres dostawy (opcjonalne)
    @ShipCity        VARCHAR(50) = NULL,
    @ShipRegion      VARCHAR(50) = NULL,
    @ShipPostalCode  VARCHAR(20) = NULL,
    @ShipCountry     VARCHAR(50) = NULL,
    @DiscountPercent DECIMAL(5,2) = 0,    -- Rabat na całe zamówienie
    @NewOrderID      INT OUTPUT           -- Zwraca ID utworzonego zamówienia
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. Walidacja istnienia Klienta i Pracownika
        IF NOT EXISTS (SELECT 1 FROM [dbo].[Customers] WHERE CustomerID = @CustomerID)
        BEGIN
            THROW 51009, 'Błąd: Podany klient (CustomerID) nie istnieje.', 1;
        END

        IF NOT EXISTS (SELECT 1 FROM [dbo].[Employees] WHERE EmployeeID = @EmployeeID)
        BEGIN
            THROW 51010, 'Błąd: Podany pracownik (EmployeeID) nie istnieje.', 1;
        END

        -- 2. Automatyczne ustalenie ID dla statusu 'Nowe'
        DECLARE @StatusNoweID INT;
        SELECT @StatusNoweID = StatusID FROM [dbo].[OrderStatus] WHERE StatusCategory = 'Nowe'; -- początkowo status zamówienia to nowe

        IF @StatusNoweID IS NULL
        BEGIN
            THROW 51011, 'Błąd: W tabeli OrderStatus brakuje statusu o nazwie ''Nowe''. Skonfiguruj słownik statusów.', 1;
        END

        -- 3. Logika "Inteligentnego Adresu"
        IF @ShipAddress IS NULL
        BEGIN
            SELECT
                @ShipName = CompanyName,
                @ShipAddress = Address,
                @ShipCity = City,
                @ShipRegion = Region,
                @ShipPostalCode = PostalCode,
                @ShipCountry = Country
            FROM [dbo].[Customers]
            WHERE CustomerID = @CustomerID;

            -- Dla klienta indywidualnego
            IF @ShipName IS NULL
            BEGIN
                 SELECT @ShipName = ContactName FROM [dbo].[Customers] WHERE CustomerID = @CustomerID;
            END
        END

        -- 4. Wstawienie nagłówka zamówienia
        INSERT INTO [dbo].[Orders] (
            [CustomerID],
            [EmployeeID],
            [OrderDate],        -- Automatycznie GETDATE()
            [RequiredDate],
            [ShippedDate],      -- NULL przy utworzeniu
            [ShipperID],
            [Freight],
            [ShipName],
            [ShipAddress],
            [ShipCity],
            [ShipRegion],
            [ShipPostalCode],
            [ShipCountry],
            [OrderDiscountPercent],
            [StatusID]
        )
        VALUES (
            @CustomerID,
            @EmployeeID,
            GETDATE(),
            @RequiredDate,
            NULL,
            @ShipperID,
            @Freight,
            @ShipName,
            @ShipAddress,
            @ShipCity,
            @ShipRegion,
            @ShipPostalCode,
            @ShipCountry,
            @DiscountPercent,
            @StatusNoweID 
        );

        -- 5. Pobranie ID nowego zamówienia
        SET @NewOrderID = SCOPE_IDENTITY();

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## NewOrderStatusCategory
Procedura dodaje nowy status zamówienia do tabeli OrderStatus i zwraca jego ID.
```sql
CREATE OR ALTER PROCEDURE [dbo].[NewOrderStatusCategory]
    @StatusCategory VARCHAR(30),       -- Nazwa statusu
    @NewStatusID    INT OUTPUT         -- Zwracane ID nowego statusu
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. Walidacja danych wejściowych
        IF @StatusCategory IS NULL OR LTRIM(RTRIM(@StatusCategory)) = ''
        BEGIN
            THROW 51007, 'Błąd: Nazwa statusu (StatusCategory) nie może być pusta.', 1;
        END

        -- 2. Sprawdzenie duplikatów
        IF EXISTS (SELECT 1 FROM [dbo].[OrderStatus] WHERE StatusCategory = @StatusCategory)
        BEGIN
            THROW 51008, 'Błąd: Taki status zamówienia już istnieje w bazie danych.', 1;
        END

        -- 3. Wstawienie rekordu
        INSERT INTO [dbo].[OrderStatus] (
            [StatusCategory]
        )
        VALUES (
            @StatusCategory
        );

        -- 4. Pobranie ID nowo dodanego statusu
        SET @NewStatusID = SCOPE_IDENTITY();

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        THROW;
    END CATCH
END;
GO
```



















## NewPayment
Procedura rejestruje płatność za fakturę, uwzględnia pozostałą kwotę do zapłaty, sprawdza nadpłatę i zwraca ID nowej płatności.
``` sql
CREATE OR ALTER PROCEDURE [dbo].[sp_RejestrujPlatnosc]
    @InvoiceID      INT,                 -- Opłacana faktura
    @Amount         DECIMAL(19,4) = NULL,-- Kwota wpłaty (jeśli NULL -> wpłacana jest całość pozostałej kwoty)
    @PaymentMethod  VARCHAR(30),         -- 'Przelew', 'Karta', 'BLIK'
    @PaymentDate    DATETIME = NULL,     -- Data wpłaty (domyślnie TERAZ)
    @NewPaymentID   INT OUTPUT           -- Zwracane ID płatności
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. WALIDACJA WSTĘPNA

        -- Sprawdzenie czy faktura istnieje
        IF NOT EXISTS (SELECT 1 FROM [dbo].[Invoices] WHERE InvoiceID = @InvoiceID)
            THROW 51035, 'Błąd: Faktura o podanym ID nie istnieje.', 1;

        -- Pobranie danych o fakturze (Łączna kwota płatności)
        DECLARE @TotalGross DECIMAL(19,4);
        SELECT @TotalGross = TotalGross FROM [dbo].[Invoices] WHERE InvoiceID = @InvoiceID;

        -- Obliczenie ile już wpłacono do tej pory
        DECLARE @AlreadyPaid DECIMAL(19,4);
        SELECT @AlreadyPaid = ISNULL(SUM(Amount), 0)
        FROM [dbo].[Payments]
        WHERE InvoiceID = @InvoiceID AND PaymentStatus = 'Zaksięgowana';

        -- Obliczenie ile zostało do zapłaty
        DECLARE @RemainingToPay DECIMAL(19,4);
        SET @RemainingToPay = @TotalGross - @AlreadyPaid;

        -- Jeśli faktura jest już opłacona w całości
        IF @RemainingToPay <= 0
            THROW 51036, 'Błąd: Ta faktura jest już w całości opłacona.', 1;


        -- 2. OBSŁUGA KWOTY WPŁATY

        -- Jeśli użytkownik nie podał kwoty (@Amount IS NULL), zakładamy spłatę całości długu
        IF @Amount IS NULL
        BEGIN
            SET @Amount = @RemainingToPay;
        END

        -- Walidacja kwoty wpłaty
        IF @Amount <= 0
            THROW 51037, 'Błąd: Kwota płatności musi być większa od zera.', 1;

        -- Zabezpieczenie przed nadpłatą (Overpayment check)
        IF @Amount > @RemainingToPay
        BEGIN
            DECLARE @ErrMsg NVARCHAR(200) = 'Błąd: Próba nadpłaty. Pozostało do zapłaty: ' + CAST(@RemainingToPay AS NVARCHAR(20));
            THROW 51038, @ErrMsg, 1;
        END

        -- Domyślna data
        IF @PaymentDate IS NULL SET @PaymentDate = GETDATE();


        -- 3. REJESTRACJA PŁATNOŚCI

        INSERT INTO [dbo].[Payments] (
            [InvoiceID],
            [PaymentDate],
            [Amount],
            [PaymentMethod],
            [PaymentStatus]
        )
        VALUES (
            @InvoiceID,
            @PaymentDate,
            @Amount,
            @PaymentMethod,
            'Zaksięgowana'
        );

        SET @NewPaymentID = SCOPE_IDENTITY();

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## NewProduct
Procedura dodaje nowy produkt do tabeli Products, inicjalizuje jego stan magazynowy i zwraca jego ID.
``` sql
CREATE OR ALTER PROCEDURE [dbo].[sp_DodajProdukt]
    @ProductName            VARCHAR(100),       -- Nazwa produktu
    @CategoryID             INT,                -- Kategoria
    @UnitPriceNet           DECIMAL(19,4),      -- Cena sprzedaży netto
    @ProductionTimeInHours  INT,                -- Czas produkcji (w godzinach)
    @ProductionCost         DECIMAL(19,4),      -- Koszt produkcji
    @NewProductID           INT OUTPUT          -- Zwracane ID nowego produktu
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. WALIDACJA DANYCH WEJŚCIOWYCH

        IF @ProductName IS NULL OR LTRIM(RTRIM(@ProductName)) = ''
        BEGIN
            THROW 51039, 'Błąd: Nazwa produktu jest wymagana.', 1;
        END

        -- Kategoria musi istnieć
        IF NOT EXISTS (SELECT 1 FROM [dbo].[ProductCategories] WHERE CategoryID = @CategoryID)
        BEGIN
            THROW 51040, 'Błąd: Podana kategoria produktu nie istnieje.', 1;
        END

        -- 2. WALIDACJA WARTOŚCI LICZBOWYCH (Constraints CHECK)

        IF @UnitPriceNet <= 0
            THROW 51041, 'Błąd: Cena sprzedaży (UnitPriceNet) musi być większa od zera.', 1;

        IF @ProductionTimeInHours <= 0
            THROW 51042, 'Błąd: Czas produkcji (ProductionTimeInHours) musi być większy od zera.', 1;

        IF @ProductionCost <= 0
            THROW 51043, 'Błąd: Koszt produkcji (ProductionCost) musi być większy od zera.', 1;

        -- 3. SPRAWDZENIE DUPLIKATÓW
        IF EXISTS (SELECT 1 FROM [dbo].[Products] WHERE ProductName = @ProductName)
        BEGIN
            THROW 51044, 'Błąd: Produkt o podanej nazwie już istnieje.', 1;
        END

        -- 4. WSTAWIENIE PRODUKTU (Table Products)
        INSERT INTO [dbo].[Products] (
            [ProductName],
            [CategoryID],
            [UnitPriceNet],
            [ProductionTimeInHours],
            [ProductionCost]
        )
        VALUES (
            @ProductName,
            @CategoryID,
            @UnitPriceNet,
            @ProductionTimeInHours,
            @ProductionCost
        );

        SET @NewProductID = SCOPE_IDENTITY();

        -- 5. INICJALIZACJA STANU MAGAZYNOWEGO (Table ProductStocks)

        INSERT INTO [dbo].[ProductStocks] (
            [ProductID],
            [QuantityAvailable],
            [QuantityReserved]
        )
        VALUES (
            @NewProductID,
            0.00,
            0.00
        );

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## NewWorkCentre
Procedura tworzy nowe stanowisko produkcyjne w tabeli WorkCenters, ustawia jego dzienne moce przerobowe i zwraca jego ID.
``` sql
CREATE OR ALTER PROCEDURE [dbo].[NewWorkCentre]
    @WorkCenterName     VARCHAR(80),    -- Nazwa stanowiska Work Centre
    @CapacityHours      FLOAT,          -- Ile godzin dziennie pracuje
    @LabourRatePerHour  MONEY,          -- Koszt roboczogodziny
    @NewWorkCenterID    INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. WALIDACJA
        IF @WorkCenterName IS NULL OR LTRIM(RTRIM(@WorkCenterName)) = ''
            THROW 51053, 'Błąd: Nazwa stanowiska jest wymagana.', 1;

        IF @CapacityHours <= 0 OR @CapacityHours > 24
            THROW 51054, 'Błąd: Moce przerobowe muszą być z zakresu 0-24 godzin.', 1;

        IF EXISTS (SELECT 1 FROM [dbo].[WorkCenters] WHERE WorkCenterName = @WorkCenterName)
            THROW 51055, 'Błąd: Stanowisko o podanej nazwie już istnieje.', 1;

        -- 2. DODANIE STANOWISKA
        INSERT INTO [dbo].[WorkCenters] (
            [WorkCenterName],
            [IsActive]
        )
        VALUES (
            @WorkCenterName,
            1
        );

        SET @NewWorkCenterID = SCOPE_IDENTITY();

        -- 3. DEFINICJA MOCY PRZEROBOWYCH (WorkCenterCapacity)
        INSERT INTO [dbo].[WorkCenterCapacity] (
            [WorkCenterID],
            [CapacityHoursPerDay]
        )
        VALUES (
            @NewWorkCenterID,
            @CapacityHours
        );

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## PlanProductionForOrder
Procedura analizuje stan magazynowy zamówienia i dla brakujących produktów tworzy zlecenia produkcyjne oraz rezerwuje potrzebne komponenty.
``` sql
/*
 Opis działania procedury
 1. Analiza braków: Procedura sprawdza każdą pozycję zamówienia (OrderDetails).
    Porównuje zamówioną ilość z ilością dostępną w magazynie (ProductStocks).
 2. Tworzenie zlecenia: Jeśli brakuje towaru (ilość dostępna - ilość zarezerwowana < ilość zamówiona),
    system tworzy nowe zlecenie produkcyjne (ProductionOrders) na brakującą liczbę sztuk.
 3. Czas produkcji: Data zakończenia (PlannedEndDate) jest wyliczana na podstawie czasu produkcji jednego produktu
    (ProductionTimeInHours) pomnożonego przez liczbę sztuk
 4. Rezerwacja składników: Na podstawie przepisu (BillOfMaterials) procedura oblicza zapotrzebowanie na surowce,
    dodaje wpisy do ProductionOrderComponents oraz rezerwuje je w magazynie (ComponentStocks i ComponentReservations).
 */

CREATE OR ALTER PROCEDURE [dbo].[PlanProductionForOrder]
    @OrderID int
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @OrderDetailID int;
    DECLARE @ProductID int;
    DECLARE @OrderQty decimal(12,2);
    DECLARE @StockAvailable decimal(12,2);
    DECLARE @StockReserved decimal(12,2);
    DECLARE @FreeStock decimal(12,2);
    DECLARE @MissingQty decimal(12,2);

    DECLARE @ProductionOrderID int;
    DECLARE @ProductionTime int;
    DECLARE @WorkCenterID int;
    DECLARE @NewStatusID int;
    DECLARE @PlannedStart datetime;
    DECLARE @PlannedEnd datetime;

    -- Pobranie ID statusu dla 'Planowane' (lub 'Nowe')
    SELECT TOP 1 @NewStatusID = ProductionOrderStatusID
    FROM ProductionOrderStatus
    WHERE StatusName LIKE 'Planowane%' OR StatusName LIKE 'Nowe%'
    ORDER BY ProductionOrderStatusID;

    IF @NewStatusID IS NULL SET @NewStatusID = 1;

    SELECT TOP 1 @WorkCenterID = WorkCenterID
    FROM WorkCenters
    WHERE IsActive = 1;

    IF @WorkCenterID IS NULL
    BEGIN
        PRINT 'Błąd: Brak aktywnych gniazd produkcyjnych (WorkCenters).';
        RETURN;
    END

    DECLARE cur_orders CURSOR FOR
    SELECT OrderDetailID, ProductID, Quantity
    FROM OrderDetails
    WHERE OrderID = @OrderID;

    OPEN cur_orders;

    FETCH NEXT FROM cur_orders INTO @OrderDetailID, @ProductID, @OrderQty;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        -- Sprawdzenie stanu magazynowego produktu
        SELECT
            @StockAvailable = ISNULL(QuantityAvailable, 0),
            @StockReserved = ISNULL(QuantityReserved, 0)
        FROM ProductStocks
        WHERE ProductID = @ProductID;

        -- Jeśli nie ma wpisu w ProductStocks, przyjmujemy 0
        SET @StockAvailable = ISNULL(@StockAvailable, 0);
        SET @StockReserved = ISNULL(@StockReserved, 0);

        -- Obliczenie wolnego zapasu (Dostępne - Zarezerwowane)
        SET @FreeStock = @StockAvailable - @StockReserved;

        -- Jeśli brakuje towaru na pokrycie zamówienia
        IF @FreeStock < @OrderQty
        BEGIN
            SET @MissingQty = @OrderQty - CASE WHEN @FreeStock > 0 THEN @FreeStock ELSE 0 END;

            -- Pobranie czasu produkcji dla produktu
            SELECT @ProductionTime = ProductionTimeInHours
            FROM Products
            WHERE ProductID = @ProductID;

            SET @PlannedStart = GETDATE();
            -- Wyliczenie daty końca: Start + (Czas jednostkowy * Ilość)
            SET @PlannedEnd = DATEADD(HOUR, ISNULL(@ProductionTime, 1) * @MissingQty, @PlannedStart);

            -- 1. Utworzenie zlecenia produkcyjnego (ProductionOrders)
            INSERT INTO ProductionOrders (
                ProductID,
                QuantityToProduce,
                PlannedStartDate,
                PlannedEndDate,
                SourceOrderID,
                WorkCenterID,
                ProductionOrderStatusID
            )
            VALUES (
                @ProductID,
                @MissingQty,
                @PlannedStart,
                @PlannedEnd,
                @OrderID,
                @WorkCenterID,
                @NewStatusID
            );

            -- Pobranie ID nowo utworzonego zlecenia
            SET @ProductionOrderID = SCOPE_IDENTITY();

            PRINT 'Utworzono zlecenie produkcyjne ID: ' + CAST(@ProductionOrderID AS VARCHAR) + ' dla produktu ID: ' + CAST(@ProductID AS VARCHAR);

            -- 2. Przypisanie komponentów do zlecenia (na podstawie BillOfMaterials)
            INSERT INTO ProductionOrderComponents (
                ProductionOrderID,
                ComponentID,
                QuantityRequired,
                QuantityIssued
            )
            SELECT
                @ProductionOrderID,
                ComponentID,
                QuantityPerProduct * @MissingQty,
                0
            FROM BillOfMaterials
            WHERE ProductID = @ProductID;

            -- 3. Rezerwacja komponentów w magazynie (ComponentStocks + ComponentReservations)

            -- Pętla po komponentach, aktualizująca rezerwacje
            DECLARE @CompID int;
            DECLARE @CompQtyNeeded decimal(12,4);

            DECLARE cur_components CURSOR FOR
            SELECT ComponentID, QuantityRequired
            FROM ProductionOrderComponents
            WHERE ProductionOrderID = @ProductionOrderID;

            OPEN cur_components;
            FETCH NEXT FROM cur_components INTO @CompID, @CompQtyNeeded;

            WHILE @@FETCH_STATUS = 0
            BEGIN
                -- A. Dodanie wpisu do tabeli rezerwacji
                INSERT INTO ComponentReservations (
                    ComponentReservationID,
                    ComponentID,
                    OrderDetailID,
                    ProductionOrderID,
                    ReservedQuantity
                )
                VALUES (
                    (SELECT ISNULL(MAX(ComponentReservationID), 0) + 1 FROM ComponentReservations),
                    @CompID,
                    NULL,
                    @ProductionOrderID,
                    @CompQtyNeeded
                );

                -- B. Aktualizacja stanu zarezerwowanego w ComponentStocks
                IF EXISTS (SELECT 1 FROM ComponentStocks WHERE ComponentID = @CompID)
                BEGIN
                    UPDATE ComponentStocks
                    SET QuantityReserved = ISNULL(QuantityReserved, 0) + @CompQtyNeeded
                    WHERE ComponentID = @CompID;
                END
                ELSE
                BEGIN
                    INSERT INTO ComponentStocks (ComponentID, QuantityAvailable, QuantityReserved)
                    VALUES (@CompID, 0, @CompQtyNeeded);
                END

                FETCH NEXT FROM cur_components INTO @CompID, @CompQtyNeeded;
            END

            CLOSE cur_components;
            DEALLOCATE cur_components;

        END
        ELSE
        BEGIN
            PRINT 'Produkt ID ' + CAST(@ProductID AS VARCHAR) + ' dostępny w magazynie. Nie zlecono produkcji.';
        END

        FETCH NEXT FROM cur_orders INTO @OrderDetailID, @ProductID, @OrderQty;
    END

    CLOSE cur_orders;
    DEALLOCATE cur_orders;
END;
GO
```

## RegisterComponentDelivery
Rejestruje dostawę komponentu, aktualizując stan magazynu i historię zmian.
``` sql
/*
 Rejestracja dostawy: Tworzy nowy wpis w tabeli ComponentDeliveries ze statusem "Dostarczono" (lub "Zrealizowano").

Aktualizacja stanu magazynowego: Zwiększa ilość dostępną (QuantityAvailable) w tabeli ComponentStocks.
 Jeśli dany komponent nie widnieje jeszcze w rejestrze stanów, tworzy nowy rekord.

Historia zmian: Dodaje wpis do rejestru ComponentStockRegister, dokumentując przyrost ilości surowca.
 */

CREATE OR ALTER PROCEDURE [dbo].[RegisterComponentDelivery]
    @ComponentID int,
    @Quantity decimal(12,4),
    @DeliveryDate date = NULL -- Opcjonalnie, domyślnie dzisiejsza data
AS
BEGIN
    SET NOCOUNT ON;

    -- Ustawienie domyślnej daty, jeśli nie podano
    IF @DeliveryDate IS NULL SET @DeliveryDate = CAST(GETDATE() AS date);

    -- Walidacja danych wejściowych
    IF @Quantity <= 0
    BEGIN
        PRINT 'Błąd: Ilość dostarczonych komponentów musi być większa od zera.';
        RETURN;
    END

    -- Sprawdzenie czy komponent istnieje
    IF NOT EXISTS (SELECT 1 FROM Components WHERE ComponentID = @ComponentID)
    BEGIN
        PRINT 'Błąd: Nie znaleziono komponentu o ID: ' + CAST(@ComponentID AS VARCHAR);
        RETURN;
    END

    BEGIN TRANSACTION;

    BEGIN TRY
        -- 1. Dodanie wpisu do tabeli dostaw (ComponentDeliveries)
        INSERT INTO ComponentDeliveries (
            ComponentID,
            Quantity,
            DateOfDelivery,
            Status
        )
        VALUES (
            @ComponentID,
            @Quantity,
            @DeliveryDate,
            'Dostarczono'
        );

        -- Pobranie ID nowej dostawy
        DECLARE @NewDeliveryID int = SCOPE_IDENTITY();

        -- 2. Aktualizacja stanu magazynowego (ComponentStocks)
        IF EXISTS (SELECT 1 FROM ComponentStocks WHERE ComponentID = @ComponentID)
        BEGIN
            UPDATE ComponentStocks
            SET QuantityAvailable = QuantityAvailable + @Quantity
            WHERE ComponentID = @ComponentID;
        END
        ELSE
        BEGIN
            INSERT INTO ComponentStocks (ComponentID, QuantityAvailable, QuantityReserved)
            VALUES (@ComponentID, @Quantity, 0);
        END

        -- 3. Rejestracja w historii zmian magazynowych (ComponentStockRegister)
        INSERT INTO ComponentStockRegister (
            ComponentID,
            EntryDate,
            Amount
        )
        VALUES (
            @ComponentID,
            GETDATE(),
            @Quantity
        );

        COMMIT TRANSACTION;

        PRINT 'Zarejestrowano dostawę nr ' + CAST(@NewDeliveryID AS VARCHAR) +
              '. Przyjęto ' + CAST(@Quantity AS VARCHAR) +
              ' szt. komponentu ID ' + CAST(@ComponentID AS VARCHAR) + '.';

    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        PRINT 'Błąd podczas rejestracji dostawy: ' + ERROR_MESSAGE();
    END CATCH
END;
GO
```

## RegisterCustomer
Rejestruje nowego klienta w bazie, walidując dane i przypisując ID.
``` sql
CREATE OR ALTER PROCEDURE [dbo].[RegisterCustomer]
    @CompanyName    VARCHAR(100) = NULL,
    @ContactName    VARCHAR(60) = NULL,
    @ContactTitle   VARCHAR(60) = NULL,
    @IsBusiness     BIT,
    @NIP            VARCHAR(15) = NULL,
    @Address        VARCHAR(100) = NULL,
    @City           VARCHAR(50) = NULL,
    @Region         VARCHAR(50) = NULL,
    @PostalCode     VARCHAR(20) = NULL,
    @Country        VARCHAR(50) = NULL,
    @Phone          VARCHAR(30) = NULL,
    @Email          VARCHAR(100) = NULL,
    @NewCustomerID  INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. Walidacja danych wejściowych

        IF @IsBusiness = 1
        BEGIN
            IF @CompanyName IS NULL OR LTRIM(RTRIM(@CompanyName)) = ''
            BEGIN
                THROW 51000, 'Błąd: Dla klienta biznesowego wymagane jest podanie nazwy firmy (CompanyName).', 1;
            END

            IF @NIP IS NULL OR LTRIM(RTRIM(@NIP)) = ''
            BEGIN
                THROW 51001, 'Błąd: Dla klienta biznesowego wymagane jest podanie numeru NIP.', 1;
            END
        END

        IF @IsBusiness = 0
        BEGIN
            -- Wymagane imię i nazwisko ORAZ NIP NULL
            IF @ContactName IS NULL OR LTRIM(RTRIM(@ContactName)) = ''
            BEGIN
                THROW 51002, 'Błąd: Dla klienta indywidualnego wymagane jest podanie imienia i nazwiska (ContactName).', 1;
            END

            IF @NIP IS NOT NULL AND LTRIM(RTRIM(@NIP)) <> ''
            BEGIN
                THROW 51004, 'Błąd: Klient indywidualny nie może posiadać numeru NIP. Pole NIP musi być puste.', 1;
            END
        END

        -- 2. Sprawdzenie unikalności NIP
        IF @NIP IS NOT NULL
        BEGIN
            IF EXISTS (SELECT 1 FROM [dbo].[Customers] WHERE NIP = @NIP)
            BEGIN
                THROW 51003, 'Błąd: Klient o podanym numerze NIP już istnieje w bazie danych.', 1;
            END
        END

        -- 3. Wstawienie rekordu do tabeli Customers
        INSERT INTO [dbo].[Customers] (
            [CompanyName],
            [ContactName],
            [ContactTitle],
            [IsBusiness],
            [NIP],
            [Address],
            [City],
            [Region],
            [PostalCode],
            [Country],
            [Phone],
            [Email]
        )
        VALUES (
            -- Jeśli klient indywidualny, CompanyName jako NULL
            CASE WHEN @IsBusiness = 1 THEN @CompanyName ELSE NULL END,
            @ContactName,
            @ContactTitle,
            @IsBusiness,
            -- Jeśli klient indywidualny, NIP jako NULL
            CASE WHEN @IsBusiness = 1 THEN @NIP ELSE NULL END,
            @Address,
            @City,
            @Region,
            @PostalCode,
            @Country,
            @Phone,
            @Email
        );

        -- 4. Pobranie ID nowo dodanego klienta
        SET @NewCustomerID = SCOPE_IDENTITY();

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrorSeverity INT = ERROR_SEVERITY();
        DECLARE @ErrorState INT = ERROR_STATE();

        RAISERROR (@ErrorMessage, @ErrorSeverity, @ErrorState);
    END CATCH
END;
GO
```

## RegisterDelivery
Rejestruje wysyłkę zamówienia, aktualizuje status zamówienia i magazyn oraz zapisuje historię wydania towaru.
``` sql
CREATE OR ALTER PROCEDURE [dbo].[RegisterDelivery]
    @OrderID            INT,                -- ID wysyłanego zamówienia
    @ShipperID          INT,                -- Shipper ID
    @ParcelNumber       VARCHAR(30),        -- Numer listu przewozowego
    @EstimatedDays      INT = 2,            -- Szacowana liczba dni dostawy
    @NewDeliveryID      INT OUTPUT          -- Zwracane ID dostawy
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. WALIDACJA DANYCH

        -- Sprawdzenie czy zamówienie istnieje
        IF NOT EXISTS (SELECT 1 FROM [dbo].[Orders] WHERE OrderID = @OrderID)
            THROW 51015, 'Błąd: Zamówienie o podanym ID nie istnieje.', 1;

        -- Sprawdzenie czy Shipper istnieje
        IF NOT EXISTS (SELECT 1 FROM [dbo].[Shippers] WHERE ShipperID = @ShipperID)
            THROW 51016, 'Błąd: Wybrany kurier (ShipperID) nie istnieje.', 1;

        -- Sprawdzenie unikalności numeru paczki
        IF EXISTS (SELECT 1 FROM [dbo].[Deliveries] WHERE ParcelNumber = @ParcelNumber)
            THROW 51017, 'Błąd: Ten numer przesyłki (ParcelNumber) jest już zarejestrowany w systemie.', 1;

        -- Sprawdzenie czy zamówienie nie zostało już wysłane (ShippedDate musi być NULL)
        IF EXISTS (SELECT 1 FROM [dbo].[Orders] WHERE OrderID = @OrderID AND ShippedDate IS NOT NULL)
            THROW 51018, 'Błąd: To zamówienie zostało już wysłane.', 1;


        -- 2. TWORZENIE DOSTAWY

        DECLARE @ShippingDate DATETIME = GETDATE();
        DECLARE @EstimatedDate DATETIME = DATEADD(DAY, @EstimatedDays, @ShippingDate);

        INSERT INTO [dbo].[Deliveries] (
            [ParcelNumber],
            [ShippingDate],
            [EstimatedDeliveryDate],
            [ShipperID],
            [OrderID],
            [DeliveryStatus]
        )
        VALUES (
            @ParcelNumber,
            @ShippingDate,
            @EstimatedDate,
            @ShipperID,
            @OrderID,
            'W drodze'
        );

        SET @NewDeliveryID = SCOPE_IDENTITY();


        -- 3. AKTUALIZACJA ZAMÓWIENIA (Tabela Orders)

        -- Pobranie ID statusu 'Wysłane'
        DECLARE @StatusWyslaneID INT;
        SELECT @StatusWyslaneID = StatusID FROM [dbo].[OrderStatus] WHERE StatusCategory = 'Wysłane';

        -- Brak statusu -> błąd
        IF @StatusWyslaneID IS NULL
            THROW 51019, 'Błąd: Brak statusu ''Wysłane'' w tabeli OrderStatus.', 1;

        UPDATE [dbo].[Orders]
        SET
            ShippedDate = @ShippingDate,
            ShipperID = @ShipperID,
            StatusID = @StatusWyslaneID
        WHERE OrderID = @OrderID;


        -- 4. CYKLE MAGAZYNOWE, KONIEC REZERWACJI

        -- KROK A: Usunięcie towaru z rezerwacji (ProductStocks)
        UPDATE S
        SET QuantityReserved = S.QuantityReserved - OD.Quantity
        FROM [dbo].[ProductStocks] S
        INNER JOIN [dbo].[OrderDetails] OD ON S.ProductID = OD.ProductID
        WHERE OD.OrderID = @OrderID;

        -- KROK B: Update ProductStockRegister
        INSERT INTO [dbo].[ProductStockRegister] (
            [ProductID],
            [EntryDate],
            [Amount]
        )
        SELECT
            ProductID,
            GETDATE(),
            -Quantity
        FROM [dbo].[OrderDetails]
        WHERE OrderID = @OrderID;

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## RegisterShipper
Rejestruje nowego przewoźnika w systemie i zwraca jego ID.
``` sql
CREATE OR ALTER PROCEDURE [dbo].[RegisterShipper]
    @CompanyName    VARCHAR(100),        -- Nazwa firmy przewozowej (Wymagana)
    @Phone          VARCHAR(30) = NULL,  -- Telefon kontaktowy
    @Address        VARCHAR(100) = NULL, -- Adres siedziby
    @ContactEmail   VARCHAR(100) = NULL, -- Email kontaktowy
    @NewShipperID   INT OUTPUT           -- Zwracane ID nowego przewoźnika
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. Walidacja danych wejściowych

        -- Nazwa firmy nie może być pusta
        IF @CompanyName IS NULL OR LTRIM(RTRIM(@CompanyName)) = ''
        BEGIN
            THROW 51005, 'Błąd: Nazwa przewoźnika (CompanyName) jest wymagana.', 1;
        END

        -- 2. Sprawdzenie duplikatów
        IF EXISTS (SELECT 1 FROM [dbo].[Shippers] WHERE CompanyName = @CompanyName)
        BEGIN
            THROW 51006, 'Błąd: Przewoźnik o podanej nazwie już istnieje w systemie.', 1;
        END

        -- 3. Wstawienie rekordu
        INSERT INTO [dbo].[Shippers] (
            [CompanyName],
            [Phone],
            [Address],
            [ContactEmail]
        )
        VALUES (
            @CompanyName,
            @Phone,
            @Address,
            @ContactEmail
        );

        -- 4. Pobranie wygenerowanego ID
        SET @NewShipperID = SCOPE_IDENTITY();

        COMMIT TRANSACTION;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
        THROW;
    END CATCH
END;
GO
```

## UpdateProductProductionCost
Aktualizuje koszt produkcji produktu lub wszystkich produktów, sumując koszt materiałów i robocizny.
``` sql
CREATE PROCEDURE [dbo].[UpdateProductProductionCost]
    @SingleProductID int = NULL -- Opcjonalnie: ID produktu. NULL -> aktualizacja wszystkich produktów.
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @LaborRate money;

    -- 1. Pobranie aktualnej stawki roboczogodziny
    SELECT TOP 1 @LaborRate = LabourRatePerHour
    FROM WorkingCostParameters;

    IF @LaborRate IS NULL SET @LaborRate = 0;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- Wyliczenia kosztu materiałów dla produktów
        WITH MaterialCosts AS (
            SELECT
                B.ProductID,
                SUM(B.QuantityPerProduct * C.UnitCost) AS TotalMaterialCost
            FROM BillOfMaterials B
            JOIN Components C ON B.ComponentID = C.ComponentID
            GROUP BY B.ProductID
        )
        -- Aktualizacja tabeli Products
        UPDATE P
        SET ProductionCost =
            (
                -- A. Koszt materiałów
                ISNULL(MC.TotalMaterialCost, 0)
                +
                -- B. Koszt robocizny (Czas * Stawka)
                (ISNULL(P.ProductionTimeInHours, 0) * @LaborRate)
            )
        FROM Products P
        LEFT JOIN MaterialCosts MC ON P.ProductID = MC.ProductID
        WHERE
            (@SingleProductID IS NULL OR P.ProductID = @SingleProductID);

        COMMIT TRANSACTION;

        IF @SingleProductID IS NOT NULL
            PRINT 'Zaktualizowano koszt produkcji dla produktu ID: ' + CAST(@SingleProductID AS VARCHAR);
        ELSE
            PRINT 'Zaktualizowano koszty produkcji dla wszystkich produktów.';

    END TRY
    BEGIN CATCH
        ROLLBACK TRANSACTION;
        PRINT 'Błąd podczas aktualizacji kosztów: ' + ERROR_MESSAGE();
    END CATCH
END;
GO
```
