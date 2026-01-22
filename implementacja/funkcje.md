# Procedury
Poniżej przedstawiono zestaw funkcji zaprojektowanych w bazie danych.

## CalculateOrderTotalBrutto
Aktualizuje koszt produkcji produktu lub wszystkich produktów, sumując koszt materiałów i robocizny.
```sql
/*
 PRZYKŁAD: SELECT dbo.fn_CalculateOrderTotalBrutto(102) AS WartoscZamowienia
 */

CREATE FUNCTION fn_CalculateOrderTotalBrutto (@OrderID int)
RETURNS decimal(19,2)
AS
BEGIN
    DECLARE @TotalGross decimal(19,4);
    DECLARE @Freight decimal(19,4);
    DECLARE @GlobalDiscountPercent decimal(5,2);

    -- 1. Obliczenie sumy brutto z pozycji zamówienia
    -- Wzór: Ilość * Cena * (1 - RabatLiniowy) * (1 + StawkaVAT)
    SELECT @TotalGross = SUM(
        (OD.Quantity * OD.UnitPrice * (1 - ISNULL(OD.Discount, 0))) -- Wartość Netto po rabacie pozycji
        * (1 + ISNULL(PC.ProductVatRate, 0)) -- Przeliczenie na Brutto wg stawki VAT kategorii
    )
    FROM OrderDetails OD
    INNER JOIN Products P ON OD.ProductID = P.ProductID
    INNER JOIN ProductCategories PC ON P.CategoryID = PC.CategoryID
    WHERE OD.OrderID = @OrderID;

    -- brak pozycji -> ustaw 0
    SET @TotalGross = ISNULL(@TotalGross, 0);

    -- 2. Pobranie informacji o rabacie globalnym i kosztach wysyłki
    SELECT
        @Freight = Freight,
        @GlobalDiscountPercent = OrderDiscountPercent
    FROM Orders
    WHERE OrderID = @OrderID;

    -- 3. Zastosowanie rabatu globalnego dla zamówienia (jesli istnieje)
    IF @GlobalDiscountPercent > 0
    BEGIN
        SET @TotalGross = @TotalGross * (1 - (@GlobalDiscountPercent / 100.0));
    END

    -- 4. Doliczenie kosztu transportu
    SET @TotalGross = @TotalGross + ISNULL(@Freight, 0);

    -- wynik zaokrąglony do 2 miejsc po przecinku
    RETURN CAST(@TotalGross AS decimal(19,2));
END;
GO
```

## EstimateProductionCompletionDate
Szacuje datę zakończenia zlecenia produkcyjnego na podstawie czasu produkcji i mocy stanowiska.
```sql
/*
 PRZYKŁAD:
 SELECT
    ProductionOrderID,
    PlannedStartDate,
    dbo.fn_EstimateProductionCompletionDate(ProductionOrderID) AS ObliczonaDataKonca,
    PlannedEndDate AS PlanowanaDataKonca
FROM ProductionOrders;
 */

CREATE FUNCTION fn_EstimateProductionCompletionDate (@ProductionOrderID int)
RETURNS datetime
AS
BEGIN
    DECLARE @PlannedStart datetime;
    DECLARE @TotalHoursRequired decimal(12,2);
    DECLARE @DailyCapacityHours float;
    DECLARE @DaysRequired float;
    DECLARE @EstimatedEndDate datetime;

    -- 1. Pobranie danych o zleceniu
    SELECT
        @PlannedStart = PO.PlannedStartDate,
        -- Obliczenie łącznej liczby godzin (Ilość * Czas na sztukę)
        @TotalHoursRequired = PO.QuantityToProduce * P.ProductionTimeInHours,
        -- Pobranie wydajności dziennej (Capacity)
        @DailyCapacityHours = WCC.CapacityHoursPerDay
    FROM ProductionOrders PO
    INNER JOIN Products P ON PO.ProductID = P.ProductID
    LEFT JOIN WorkCenterCapacity WCC ON PO.WorkCenterID = WCC.WorkCenterID
    WHERE PO.ProductionOrderID = @ProductionOrderID;

    -- Zabezpieczenie: Jeśli nie znaleziono zlecenia -> NULL
    IF @PlannedStart IS NULL RETURN NULL;

    -- 2. Obliczenie daty końcowej z uwzględnieniem mocy przerobowych
    IF @DailyCapacityHours IS NOT NULL AND @DailyCapacityHours > 0
    BEGIN
        -- Dzielenie pracy na dni
        -- Przykład: 20 godzin pracy / 8 godzin dziennie = 2.5 dnia
        SET @DaysRequired = @TotalHoursRequired / @DailyCapacityHours;

        -- Dodajemy wyliczone dni do daty startu
        SET @EstimatedEndDate = DATEADD(HOUR, (@DaysRequired * 24), @PlannedStart);
    END
    ELSE
    BEGIN
        -- Jeśli nie zdefiniowano WorkCenterCapacity czas pracy dodawany ciągiem
        SET @EstimatedEndDate = DATEADD(HOUR, @TotalHoursRequired, @PlannedStart);
    END

    RETURN @EstimatedEndDate;
END;
GO
```

## GetCurrentProductionCost
Oblicza aktualny koszt produkcji produktu na podstawie kosztów materiałów i robocizny.
```sql
/*
 PRZYKŁAD: UPDATE Products
SET ProductionCost = dbo.fn_GetCurrentProductionCost(ProductID);
 */

CREATE FUNCTION fn_GetCurrentProductionCost (@ProductID int)
RETURNS decimal(19,4)
AS
BEGIN
    DECLARE @MaterialCost decimal(19,4);
    DECLARE @LaborCost decimal(19,4);
    DECLARE @LaborRate money;
    DECLARE @ProductionTime int;

    -- 1. Obliczenie kosztu materiałów
    -- (Suma: ilość z BillOfMaterials * aktualna cena jednostkowa komponentu)
    SELECT @MaterialCost = SUM(B.QuantityPerProduct * C.UnitCost)
    FROM BillOfMaterials B
    INNER JOIN Components C ON B.ComponentID = C.ComponentID
    WHERE B.ProductID = @ProductID;

    -- 2. Pobranie danych do kosztu robocizny
    -- Pobieramy czas produkcji przypisany do produktu
    SELECT @ProductionTime = ProductionTimeInHours
    FROM Products
    WHERE ProductID = @ProductID;

    -- Pobieramy aktualną stawkę godzinową z tabeli parametrów
    SELECT TOP 1 @LaborRate = LabourRatePerHour
    FROM WorkingCostParameters;

    -- 3. Wyliczenie kosztu robocizny (Czas * Stawka)
    SET @LaborCost = ISNULL(@ProductionTime, 0) * ISNULL(@LaborRate, 0);

    -- 4. Zwrócenie sumy całkowitej
    RETURN ISNULL(@MaterialCost, 0) + @LaborCost;
END;
GO
```

## GetCustomerDiscountLevel
Wylicza poziom rabatu klienta na podstawie wartości jego zamówień z ostatniego roku.
```sql
CREATE FUNCTION fn_GetCustomerDiscountLevel (@CustomerID int)
RETURNS decimal(5,2)
AS
BEGIN
    DECLARE @TotalSales decimal(19,4);
    DECLARE @SuggestedDiscount decimal(5,2);

    -- 1. Obliczenie sumy wartości zamówień klienta z ostatnich 365 dni
    -- SUMA (Ilość * Cena jednostkowa) z tabeli OrderDetails
    SELECT @TotalSales = SUM(OD.Quantity * OD.UnitPrice)
    FROM Orders O
    INNER JOIN OrderDetails OD ON O.OrderID = OD.OrderID
    INNER JOIN OrderStatus OS ON O.StatusID = OS.StatusID
    WHERE O.CustomerID = @CustomerID
      AND O.OrderDate >= DATEADD(day, -365, GETDATE()) -- Tylko ostatni rok
      AND OS.StatusCategory NOT LIKE 'Anulowane%'

    --Jeśli klient nie ma historii, suma rabatu 0
    SET @TotalSales = ISNULL(@TotalSales, 0);

    -- 2. Progi rabatowe
    IF @TotalSales > 100000.00
        SET @SuggestedDiscount = 10.00; -- 10% dla klientów >100k
    ELSE IF @TotalSales > 50000.00
        SET @SuggestedDiscount = 5.00;  -- 5% dla klientów >50k
    ELSE IF @TotalSales > 10000.00
        SET @SuggestedDiscount = 2.00;  -- 2% dla klientów >10k
    ELSE
        SET @SuggestedDiscount = 0.00;  -- Brak rabatu dla pozostałych klientów

    RETURN @SuggestedDiscount;
END;
GO
```

## GetMaterialShortages
Zwraca listę brakujących komponentów dla zlecenia produkcyjnego na podstawie zapotrzebowania i stanów magazynowych.
```sql
/*
 PRZYKŁAD
 SELECT * FROM fn_GetMaterialShortages(105);
 */
CREATE FUNCTION fn_GetMaterialShortages (@ProductionOrderID int)
RETURNS TABLE
AS
RETURN
(
    SELECT
        C.ComponentID,
        C.ComponentName,
        C.UnitOfMeasure,

        -- Łączne zapotrzebowanie dla poszczególnego zlecenia
        POC.QuantityRequired,

        -- Ile jest aktualnie na produckji
        POC.QuantityIssued,

        -- Ile jeszcze trzeba dać na produkcję (Zapotrzebowanie - Issued)
        (POC.QuantityRequired - POC.QuantityIssued) AS QuantityStillNeeded,

        -- Ile jest wolnych zasobów w magazynie (Dostępne - Zarezerwowane)
        (ISNULL(S.QuantityAvailable, 0) - ISNULL(S.QuantityReserved, 0)) AS WarehouseFreeStock,

        -- Wyliczenie braku:
        CASE
            WHEN (POC.QuantityRequired - POC.QuantityIssued) > (ISNULL(S.QuantityAvailable, 0) - ISNULL(S.QuantityReserved, 0))
            THEN (POC.QuantityRequired - POC.QuantityIssued) - (ISNULL(S.QuantityAvailable, 0) - ISNULL(S.QuantityReserved, 0))
            ELSE 0
        END AS MissingQuantity

    FROM ProductionOrderComponents POC
    INNER JOIN Components C ON POC.ComponentID = C.ComponentID
    LEFT JOIN ComponentStocks S ON C.ComponentID = S.ComponentID
    WHERE POC.ProductionOrderID = @ProductionOrderID
);
GO
```

## GetMaxPossibleProduction
Oblicza maksymalną liczbę sztuk produktu możliwych do wyprodukowania na podstawie dostępnych komponentów.
```sql
/*
 PRZYKŁAD
 SELECT
    ProductName,
    QuantityAvailable AS W_Magazynie_Gotowe,
    dbo.fn_GetMaxPossibleProduction(ProductID) AS Mozliwe_Do_Produkcji
FROM Products;
 */

CREATE FUNCTION fn_GetMaxPossibleProduction (@ProductID int)
RETURNS int
AS
BEGIN
    DECLARE @MaxPossible int;

    -- Obliczamy limit produkcji wyznaczony przez każdy składnik z osobna i wybieramy najmniejszą wartość
    SELECT @MaxPossible = MIN(
        FLOOR(
            (ISNULL(S.QuantityAvailable, 0) - ISNULL(S.QuantityReserved, 0)) -- Ilość wolna
            /
            NULLIF(BOM.QuantityPerProduct, 0)
        )
    )
    FROM BillOfMaterials BOM
    -- LEFT JOIN sytuacja, gdy składnik jest w BillOfMaterials, ale nie ma go w tabeli ComponentStocks (wtedy NULL -> 0)
    LEFT JOIN ComponentStocks S ON BOM.ComponentID = S.ComponentID
    WHERE BOM.ProductID = @ProductID;

    IF @MaxPossible IS NULL OR @MaxPossible < 0
        SET @MaxPossible = 0;

    RETURN @MaxPossible;
END;
GO
```

