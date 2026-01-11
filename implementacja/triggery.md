# Triggery
Poniżej przedstawiono zestaw triggerów utworzonych w bazie danych.
Służą one zachowaniu integralności danych w bazie, a także umożliwiają zgrabne wykonywanie operacji niemożliwych/bardzo skomplikowanych do osiągnięcia samymi zapytaniami SQL.

## 1. Trigger trg_OrderDetails_ReserveStock
Automatyzuje on rezerwację produktu po dodaniu pozycji zamówienia.

```sql
CREATE TRIGGER trg_OrderDetails_ReserveStock
ON dbo.OrderDetails
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;

    IF EXISTS (
        SELECT 1
        FROM inserted i
        JOIN dbo.ProductStocks ps ON ps.ProductID = i.ProductID
        GROUP BY ps.ProductID, ps.QuantityAvailable, ps.QuantityReserved
        HAVING SUM(i.Quantity) > (ps.QuantityAvailable - ps.QuantityReserved)
    )
    BEGIN
        RAISERROR ('Brak wystarczającego stanu magazynowego produktu', 16, 1);
        ROLLBACK TRANSACTION;
        RETURN;
    END;

    UPDATE ps
    SET QuantityReserved = QuantityReserved + r.SumQty
    FROM dbo.ProductStocks ps
    JOIN (
        SELECT ProductID, SUM(Quantity) AS SumQty
        FROM inserted
        GROUP BY ProductID
    ) r ON r.ProductID = ps.ProductID;
END;
```

## 2. Trigger trg_OrderDetails_ReleaseStock
Umożliwia on zwolnienie rezerwacji po usunięciu pozycji zamówienia.
```sql
CREATE TRIGGER trg_OrderDetails_ReleaseStock
ON dbo.OrderDetails
AFTER DELETE
AS
BEGIN
    SET NOCOUNT ON;

    UPDATE ps
    SET QuantityReserved = QuantityReserved - r.SumQty
    FROM dbo.ProductStocks ps
    JOIN (
        SELECT ProductID, SUM(Quantity) AS SumQty
        FROM deleted
        GROUP BY ProductID
    ) r ON r.ProductID = ps.ProductID;
END;
```
## 3. Trigger trg_OrderDetails_UpdateQuantity
Dzięki niemu możiwa jest korekta rezerwacji przy zmianie ilości zamawianego produktu.
```sql
CREATE TRIGGER trg_OrderDetails_UpdateQuantity
ON dbo.OrderDetails
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    IF NOT UPDATE(Quantity)
        RETURN;

    IF EXISTS (
        SELECT 1
        FROM inserted i
        JOIN deleted d ON i.OrderDetailID = d.OrderDetailID
        JOIN dbo.ProductStocks ps ON ps.ProductID = i.ProductID
        WHERE i.Quantity > d.Quantity
        GROUP BY ps.ProductID, ps.QuantityAvailable, ps.QuantityReserved
        HAVING SUM(i.Quantity - d.Quantity) >
               (ps.QuantityAvailable - ps.QuantityReserved)
    )
    BEGIN
        RAISERROR ('Brak wystarczającego stanu magazynowego przy zmianie ilości', 16, 1);
        ROLLBACK TRANSACTION;
        RETURN;
    END;

    UPDATE ps
    SET QuantityReserved = QuantityReserved + x.DiffQty
    FROM dbo.ProductStocks ps
    JOIN (
        SELECT i.ProductID,
               SUM(i.Quantity - d.Quantity) AS DiffQty
        FROM inserted i
        JOIN deleted d ON i.OrderDetailID = d.OrderDetailID
        GROUP BY i.ProductID
    ) x ON x.ProductID = ps.ProductID;
END;
```
## 4. Trigger trg_OrderDetails_SetInitialStatus
Ustawia on początkowy status zamówienia.
```sql
CREATE TRIGGER trg_OrderDetails_SetInitialStatus
ON dbo.OrderDetails
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;

    UPDATE od
    SET Status = 'Zarezerwowana'
    FROM dbo.OrderDetails od
    JOIN inserted i ON i.OrderDetailID = od.OrderDetailID;
END;
```
## 5. Trigger trg_InvoiceItems_RecalculateInvoice
Ten trigger przelicza wartość faktury po jakiejkolwiek zmianie w jej obrębie.
```sql
CREATE TRIGGER trg_InvoiceItems_RecalculateInvoice
ON dbo.InvoiceItems
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @Invoices TABLE (InvoiceID INT);

    INSERT INTO @Invoices
    SELECT DISTINCT InvoiceID FROM inserted
    UNION
    SELECT DISTINCT InvoiceID FROM deleted;

    UPDATE i
    SET
        TotalNet   = x.TotalNet,
        TotalVat   = x.TotalVat,
        TotalGross = x.TotalNet + x.TotalVat
    FROM dbo.Invoices i
    JOIN (
        SELECT
            InvoiceID,
            SUM(
                Quantity * UnitNetPrice *
                (1 - ISNULL(DiscountPercent,0)/100.0)
            ) AS TotalNet,
            SUM(
                Quantity * UnitNetPrice *
                (1 - ISNULL(DiscountPercent,0)/100.0) *
                ISNULL(VatRate, 23)/100.0
            ) AS TotalVat
        FROM dbo.InvoiceItems
        GROUP BY InvoiceID
    ) x ON x.InvoiceID = i.InvoiceID
    WHERE i.InvoiceID IN (SELECT InvoiceID FROM @Invoices);
END;
```
## 6. Trigger trg_Payments_PreventOverpayment
Ten trigger uniemożliwia nadpłatę faktury, w jej przypadku robi rollback transakcji.
```sql
CREATE TRIGGER trg_Payments_PreventOverpayment
ON dbo.Payments
AFTER INSERT, UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    IF EXISTS (
        SELECT 1
        FROM inserted p
        JOIN dbo.Invoices i ON i.InvoiceID = p.InvoiceID
        GROUP BY p.InvoiceID, i.TotalGross
        HAVING
            ISNULL(SUM(ISNULL(p.Amount,0)),0) +
            ISNULL((
                SELECT SUM(ISNULL(Amount,0))
                FROM dbo.Payments
                WHERE InvoiceID = p.InvoiceID
            ),0)
            > i.TotalGross
    )
    BEGIN
        RAISERROR ('Nadpłata faktury', 16, 1);
        ROLLBACK TRANSACTION;
        RETURN;
    END;
END;
```
## 7. Trigger trg_Payments_SetStatus
Ten trigger automatycznie ustawia status płatności.
```sql
CREATE TRIGGER trg_Payments_SetStatus
ON dbo.Payments
AFTER INSERT, UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    UPDATE p
    SET PaymentStatus =
        CASE
            WHEN ISNULL(p.Amount,0) = 0 THEN 'Nieopłacona'
            ELSE 'Zaksięgowana'
        END
    FROM dbo.Payments p
    JOIN inserted i ON i.PaymentID = p.PaymentID;
END;
```
## 8. Trigger trg_Invoices_CheckDates
Dzięki niemu mamy walidację dat faktury.
```sql
CREATE TRIGGER trg_Invoices_CheckDates
ON dbo.Invoices
AFTER INSERT, UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    IF EXISTS (
        SELECT 1
        FROM inserted
        WHERE DueDate < InvoiceDate
    )
    BEGIN
        RAISERROR ('Termin płatności nie może być wcześniejszy niż data faktury', 16, 1);
        ROLLBACK TRANSACTION;
        RETURN;
    END;
END;
```
## 9. Trigger trg_Payments_UpdateInvoiceStatus
Ten trigger aktualizuje status całej faktury na podstawie statusu płatności.
```sql
CREATE TRIGGER trg_Payments_UpdateInvoiceStatus
ON dbo.Payments
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @Invoices TABLE (InvoiceID INT);

    INSERT INTO @Invoices
    SELECT DISTINCT InvoiceID FROM inserted
    UNION
    SELECT DISTINCT InvoiceID FROM deleted;

    UPDATE i
    SET Status =
        CASE
            WHEN ISNULL(p.SumPaid,0) = 0 THEN 'Nieopłacona'
            WHEN p.SumPaid < i.TotalGross THEN 'Częściowo opłacona'
            ELSE 'Opłacona'
        END
    FROM dbo.Invoices i
    LEFT JOIN (
        SELECT InvoiceID, SUM(ISNULL(Amount,0)) AS SumPaid
        FROM dbo.Payments
        GROUP BY InvoiceID
    ) p ON p.InvoiceID = i.InvoiceID
    WHERE i.InvoiceID IN (SELECT InvoiceID FROM @Invoices);
END;
```
## 10. Trigger trg_InvoiceItems_MarkOrderDetails
Ten trigger oznacza poszczególne części zamówień jako zafakturowane.
```sql
CREATE TRIGGER trg_InvoiceItems_MarkOrderDetails
ON dbo.InvoiceItems
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;

    UPDATE od
    SET Status = 'Zafakturowana'
    FROM dbo.OrderDetails od
    JOIN dbo.Invoices i ON i.OrderID = od.OrderID
    JOIN inserted ii ON ii.InvoiceID = i.InvoiceID
                   AND ii.ProductID = od.ProductID;
END;
```

