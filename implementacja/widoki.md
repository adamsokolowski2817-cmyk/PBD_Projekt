# Widoki

Poniżej przedstawiono zestaw widoków utworzonych w bazie danych.

Widoki służą do generowania raportów oraz zestawień wymaganych w opisie projektu.  

Ułatwiają one analizę danych bez konieczności pisania złożonych zapytań SQL.  

## 1. Widok vw_OrderLinesNet

Wyświetla szczegółowe informacje o pozycjach zamówień wraz z naliczonymi rabatami.
Widok oblicza wartości brutto oraz końcowe wartości netto każdej pozycji zamówienia.

```sql
CREATE OR ALTER VIEW dbo.vw_OrderLinesNet
AS
SELECT
    o.OrderID,
    o.CustomerID,
    o.EmployeeID,
    o.OrderDate,
    o.RequiredDate,
    o.ShippedDate,
    o.StatusID,
    os.StatusCategory,

    od.OrderDetailID,
    od.ProductID,

    p.ProductName,
    p.CategoryID,
    pc.CategoryName,

    od.Quantity,
    od.UnitPrice,

    -- Rabat pozycji (OrderDetails.Discount) i rabat zamówienia (Orders.OrderDiscountPercent)
    ISNULL(od.Discount, 0) AS LineDiscountPercent,
    ISNULL(o.OrderDiscountPercent, 0) AS OrderDiscountPercent,

    -- Brutto: ilość * cena
    CAST(od.Quantity * od.UnitPrice AS decimal(19,4)) AS LineValueGross,

    -- Po rabacie pozycji
    CAST(
        od.Quantity * od.UnitPrice * (1 - ISNULL(od.Discount,0)/100.0)
        AS decimal(19,4)
    ) AS LineValueAfterLineDiscount,

    -- Netto finalne: rabat pozycji + rabat zamówienia
    CAST(
        od.Quantity * od.UnitPrice
        * (1 - ISNULL(od.Discount,0)/100.0)
        * (1 - ISNULL(o.OrderDiscountPercent,0)/100.0)
        AS decimal(19,4)
    ) AS LineValueNet
FROM dbo.Orders o
JOIN dbo.OrderDetails od      ON od.OrderID = o.OrderID
JOIN dbo.Products p           ON p.ProductID = od.ProductID
JOIN dbo.ProductCategories pc ON pc.CategoryID = p.CategoryID
JOIN dbo.OrderStatus os       ON os.StatusID = o.StatusID;
GO
```

## 2. Widok vw_SalesByCategory_Weekly

Wyświetla tygodniowe statystyki sprzedaży z podziałem na kategorie produktów.
Pokazuje liczbę zamówień, ilość sprzedanych produktów oraz przychód netto.

```sql
CREATE OR ALTER VIEW dbo.vw_SalesByCategory_Weekly
AS
SELECT
    DATEPART(ISO_YEAR, oln.OrderDate) AS ISOYear,
    DATEPART(ISO_WEEK, oln.OrderDate) AS ISOWeek,
    oln.CategoryID,
    oln.CategoryName,
    COUNT(DISTINCT oln.OrderID) AS OrdersCount,
    SUM(CAST(oln.Quantity AS decimal(19,4))) AS QuantitySold,
    SUM(oln.LineValueNet) AS RevenueNet
FROM dbo.vw_OrderLinesNet oln
GROUP BY
    DATEPART(ISO_YEAR, oln.OrderDate),
    DATEPART(ISO_WEEK, oln.OrderDate),
    oln.CategoryID,
    oln.CategoryName;
GO
```

## 3. Widok vw_SalesByCategory_Monthly

Wyświetla miesięczne statystyki sprzedaży z podziałem na kategorie produktów.
Umożliwia analizę przychodów i ilości sprzedaży w ujęciu miesięcznym.

```sql
CREATE OR ALTER VIEW dbo.vw_SalesByCategory_Monthly
AS
SELECT
    YEAR(oln.OrderDate) AS [Year],
    MONTH(oln.OrderDate) AS [Month],
    oln.CategoryID,
    oln.CategoryName,
    COUNT(DISTINCT oln.OrderID) AS OrdersCount,
    SUM(CAST(oln.Quantity AS decimal(19,4))) AS QuantitySold,
    SUM(oln.LineValueNet) AS RevenueNet
FROM dbo.vw_OrderLinesNet oln
GROUP BY
    YEAR(oln.OrderDate),
    MONTH(oln.OrderDate),
    oln.CategoryID,
    oln.CategoryName;
GO
```

## 4. Widok vw_UnitProductionCost_PerProduct

Wyświetla jednostkowy koszt produkcji oraz czas produkcji dla każdego produktu.
Widok pozwala analizować koszty wytwarzania pojedynczych produktów.

```sql
CREATE OR ALTER VIEW dbo.vw_UnitProductionCost_PerProduct
AS
SELECT
    p.ProductID,
    p.ProductName,
    p.CategoryID,
    pc.CategoryName,
    p.UnitPriceNet,
    p.ProductionTimeInHours,
    p.ProductionCost AS UnitProductionCost
FROM dbo.Products p
JOIN dbo.ProductCategories pc ON pc.CategoryID = p.CategoryID;
GO
```

## 5. Widok vw_ProductionCostByCategory_Yearly

Wyświetla roczne koszty planowanej produkcji z podziałem na kategorie produktów.
Koszt liczony jest na podstawie ilości planowanej produkcji i kosztu jednostkowego.

```sql
CREATE OR ALTER VIEW dbo.vw_ProductionCostByCategory_Yearly
AS
SELECT
    YEAR(po.PlannedStartDate) AS [Year],
    pc.CategoryID,
    pc.CategoryName,
    SUM(CAST(po.QuantityToProduce AS decimal(19,4))) AS QtyPlanned,
    SUM(CAST(po.QuantityToProduce AS decimal(19,4)) * CAST(p.ProductionCost AS decimal(19,4))) AS PlannedProductionCost
FROM dbo.ProductionOrders po
JOIN dbo.Products p           ON p.ProductID = po.ProductID
JOIN dbo.ProductCategories pc ON pc.CategoryID = p.CategoryID
GROUP BY
    YEAR(po.PlannedStartDate),
    pc.CategoryID,
    pc.CategoryName;
GO
```

## 6. Widok vw_ProductionCostByCategory_Quarterly

Wyświetla kwartalne koszty planowanej produkcji dla poszczególnych kategorii produktów.
Umożliwia analizę kosztów produkcji w ujęciu kwartalnym.

```sql
CREATE OR ALTER VIEW dbo.vw_ProductionCostByCategory_Quarterly
AS
SELECT
    YEAR(po.PlannedStartDate) AS [Year],
    DATEPART(QUARTER, po.PlannedStartDate) AS [Quarter],
    pc.CategoryID,
    pc.CategoryName,
    SUM(CAST(po.QuantityToProduce AS decimal(19,4))) AS QtyPlanned,
    SUM(CAST(po.QuantityToProduce AS decimal(19,4)) * CAST(p.ProductionCost AS decimal(19,4))) AS PlannedProductionCost
FROM dbo.ProductionOrders po
JOIN dbo.Products p           ON p.ProductID = po.ProductID
JOIN dbo.ProductCategories pc ON pc.CategoryID = p.CategoryID
GROUP BY
    YEAR(po.PlannedStartDate),
    DATEPART(QUARTER, po.PlannedStartDate),
    pc.CategoryID,
    pc.CategoryName;
GO
```

## 7. Widok vw_ProductionCostByCategory_Monthly

Wyświetla miesięczne koszty planowanej produkcji z podziałem na kategorie produktów.
Pozwala śledzić zmiany kosztów produkcji w czasie.

```sql
CREATE OR ALTER VIEW dbo.vw_ProductionCostByCategory_Monthly
AS
SELECT
    YEAR(po.PlannedStartDate) AS [Year],
    MONTH(po.PlannedStartDate) AS [Month],
    pc.CategoryID,
    pc.CategoryName,
    SUM(CAST(po.QuantityToProduce AS decimal(19,4))) AS QtyPlanned,
    SUM(CAST(po.QuantityToProduce AS decimal(19,4)) * CAST(p.ProductionCost AS decimal(19,4))) AS PlannedProductionCost
FROM dbo.ProductionOrders po
JOIN dbo.Products p           ON p.ProductID = po.ProductID
JOIN dbo.ProductCategories pc ON pc.CategoryID = p.CategoryID
GROUP BY
    YEAR(po.PlannedStartDate),
    MONTH(po.PlannedStartDate),
    pc.CategoryID,
    pc.CategoryName;
GO
```

## 8. Widok vw_ProductionCost_Weekly

Wyświetla tygodniowe koszty planowanej produkcji dla całej firmy.
Widok służy do szybkiej analizy kosztów produkcji w krótkich okresach czasu.

```sql
CREATE OR ALTER VIEW dbo.vw_ProductionCost_Weekly
AS
SELECT
    DATEPART(ISO_YEAR, po.PlannedStartDate) AS ISOYear,
    DATEPART(ISO_WEEK, po.PlannedStartDate) AS ISOWeek,
    SUM(CAST(po.QuantityToProduce AS decimal(19,4)) * CAST(p.ProductionCost AS decimal(19,4))) AS PlannedProductionCost
FROM dbo.ProductionOrders po
JOIN dbo.Products p ON p.ProductID = po.ProductID
GROUP BY
    DATEPART(ISO_YEAR, po.PlannedStartDate),
    DATEPART(ISO_WEEK, po.PlannedStartDate);
GO
```

## 9. Widok vw_ProductionCost_Monthly

Wyświetla miesięczne koszty planowanej produkcji.
Umożliwia porównywanie kosztów produkcji pomiędzy kolejnymi miesiącami.

```sql
CREATE OR ALTER VIEW dbo.vw_ProductionCost_Monthly
AS
SELECT
    YEAR(po.PlannedStartDate) AS [Year],
    MONTH(po.PlannedStartDate) AS [Month],
    SUM(CAST(po.QuantityToProduce AS decimal(19,4)) * CAST(p.ProductionCost AS decimal(19,4))) AS PlannedProductionCost
FROM dbo.ProductionOrders po
JOIN dbo.Products p ON p.ProductID = po.ProductID
GROUP BY
    YEAR(po.PlannedStartDate),
    MONTH(po.PlannedStartDate);
GO
```

## 10. Widok vw_CurrentProductStock

Wyświetla aktualne stany magazynowe produktów.
Pokazuje ilość dostępną, zarezerwowaną oraz faktycznie wolną w magazynie.

```sql
CREATE OR ALTER VIEW dbo.vw_CurrentProductStock
AS
SELECT
    ps.ProductID,
    p.ProductName,
    p.CategoryID,
    pc.CategoryName,
    ps.QuantityAvailable,
    ps.QuantityReserved,
    CAST(ps.QuantityAvailable - ps.QuantityReserved AS decimal(12,2)) AS QuantityFree
FROM dbo.ProductStocks ps
JOIN dbo.Products p           ON p.ProductID = ps.ProductID
JOIN dbo.ProductCategories pc ON pc.CategoryID = p.CategoryID;
GO
```

## 11. Widok vw_PlannedProduction_Products

Wyświetla produkty zaplanowane do produkcji wraz z terminami oraz statusem zleceń.
Widok służy do przeglądania aktualnego planu produkcji.

```sql
CREATE OR ALTER VIEW dbo.vw_PlannedProduction_Products
AS
SELECT
    po.ProductionOrderID,
    po.ProductID,
    p.ProductName,
    p.CategoryID,
    pc.CategoryName,
    po.QuantityToProduce,
    po.PlannedStartDate,
    po.PlannedEndDate,
    po.SourceOrderID,
    po.WorkCenterID,
    wc.WorkCenterName,
    po.ProductionOrderStatusID,
    pos.StatusName AS ProductionStatus
FROM dbo.ProductionOrders po
JOIN dbo.Products p               ON p.ProductID = po.ProductID
JOIN dbo.ProductCategories pc     ON pc.CategoryID = p.CategoryID
JOIN dbo.WorkCenters wc           ON wc.WorkCenterID = po.WorkCenterID
JOIN dbo.ProductionOrderStatus pos ON pos.ProductionOrderStatusID = po.ProductionOrderStatusID;
GO
```

## 12. Widok vw_ProductionPlan_WithLoad

Wyświetla plan produkcji wraz z obciążeniem stanowisk roboczych.
Pokazuje wymagany czas produkcji oraz dostępne moce przerobowe.

```sql
CREATE OR ALTER VIEW dbo.vw_ProductionPlan_WithLoad
AS
SELECT
    po.ProductionOrderID,
    po.ProductID,
    p.ProductName,
    po.QuantityToProduce,
    po.PlannedStartDate,
    po.PlannedEndDate,
    po.WorkCenterID,
    wc.WorkCenterName,
    cap.CapacityHoursPerDay,
    CAST(CAST(po.QuantityToProduce AS decimal(19,4)) * CAST(p.ProductionTimeInHours AS decimal(19,4)) AS decimal(19,4)) AS RequiredHours
FROM dbo.ProductionOrders po
JOIN dbo.Products p            ON p.ProductID = po.ProductID
JOIN dbo.WorkCenters wc        ON wc.WorkCenterID = po.WorkCenterID
JOIN dbo.WorkCenterCapacity cap ON cap.WorkCenterID = po.WorkCenterID;
GO
```

## 13. Widok vw_CustomerOrderHistory_WithDiscounts

Wyświetla historię zamówień klientów wraz z zastosowanymi rabatami.
Widok umożliwia analizę zakupów klientów w różnych przedziałach czasowych.

```sql
CREATE OR ALTER VIEW dbo.vw_CustomerOrderHistory_WithDiscounts
AS
SELECT
    c.CustomerID,
    c.CompanyName,
    c.ContactName,
    c.IsB2B,
    c.NIP,

    o.OrderID,
    o.OrderDate,
    os.StatusCategory AS OrderStatusCategory,
    ISNULL(o.OrderDiscountPercent,0) AS OrderDiscountPercent,

    od.OrderDetailID,
    od.ProductID,
    p.ProductName,
    pc.CategoryName,

    od.Quantity,
    od.UnitPrice,
    ISNULL(od.Discount,0) AS LineDiscountPercent,

    CAST(od.Quantity * od.UnitPrice AS decimal(19,4)) AS LineValueGross,
    CAST(od.Quantity * od.UnitPrice * (1 - ISNULL(od.Discount,0)/100.0) AS decimal(19,4)) AS LineValueAfterLineDiscount,
    CAST(
        od.Quantity * od.UnitPrice
        * (1 - ISNULL(od.Discount,0)/100.0)
        * (1 - ISNULL(o.OrderDiscountPercent,0)/100.0)
        AS decimal(19,4)
    ) AS LineValueNetFinal
FROM dbo.Customers c
JOIN dbo.Orders o             ON o.CustomerID = c.CustomerID
JOIN dbo.OrderStatus os       ON os.StatusID  = o.StatusID
JOIN dbo.OrderDetails od      ON od.OrderID   = o.OrderID
JOIN dbo.Products p           ON p.ProductID  = od.ProductID
JOIN dbo.ProductCategories pc ON pc.CategoryID = p.CategoryID;
GO
```

## 14. Widok vw_CurrentComponentStock

Wyświetla aktualne stany magazynowe komponentów wykorzystywanych w produkcji.
Pokazuje ilość dostępną, zarezerwowaną oraz wolną dla każdego komponentu.

```sql
CREATE OR ALTER VIEW dbo.vw_CurrentComponentStock
AS
SELECT
    cs.ComponentID,
    c.ComponentName,
    c.ComponentCategoryID,
    cc.CategoryName AS ComponentCategoryName,
    c.UnitOfMeasure,
    c.UnitCost,
    cs.QuantityAvailable,
    cs.QuantityReserved,
    CAST(cs.QuantityAvailable - cs.QuantityReserved AS decimal(12,4)) AS QuantityFree
FROM dbo.ComponentStocks cs
JOIN dbo.Components c           ON c.ComponentID = cs.ComponentID
JOIN dbo.ComponentCategories cc ON cc.ComponentCategoryID = c.ComponentCategoryID;
GO
```

## 15. Widok vw_ComponentDemand_ForPlannedProduction

Wyświetla zapotrzebowanie na komponenty wynikające z planu produkcji.
Widok wspiera planowanie materiałowe (MRP) i kontrolę dostępności komponentów.

```sql
CREATE OR ALTER VIEW dbo.vw_ComponentDemand_ForPlannedProduction
AS
SELECT
    po.ProductionOrderID,
    po.PlannedStartDate,
    po.PlannedEndDate,
    po.ProductID,
    p.ProductName,

    bom.ComponentID,
    c.ComponentName,
    c.UnitOfMeasure,

    CAST(CAST(po.QuantityToProduce AS decimal(19,4)) * CAST(bom.QuantityPerProduct AS decimal(19,4)) AS decimal(19,4)) AS ComponentQtyRequired
FROM dbo.ProductionOrders po
JOIN dbo.Products p          ON p.ProductID = po.ProductID
JOIN dbo.BillOfMaterials bom ON bom.ProductID = po.ProductID
JOIN dbo.Components c        ON c.ComponentID = bom.ComponentID;
GO
```
