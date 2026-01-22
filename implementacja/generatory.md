# Generowanie przykładowych danych

Dane do tabel, które zawierają niedużo wierszy (np. Customers, Shippers) zostały ręcznie wprowadzone do bazy.
Przykładowe dane dla tych tabel wygenerowane zostały przy użyciu Gemini 3 Pro, a następnie manualnie wprowadzone do bazy. 

Zamówienia i zawartości zamówień wygenerowane zostały przy użyciu kody w python.


## Generowanie elementów tabeli Orders
``` python
import random
from datetime import date, timedelta

# Konfiguracja
FILENAME = "insert_orders.sql"
ILOSC_REKORDOW = 200

# Zakres dat: 2026-01-20 do 2026-01-30
START_DATE = date(2026, 1, 20)
END_DATE = date(2026, 1, 30)
DAYS_DIFF = (END_DATE - START_DATE).days

def generate_sql_file():
    with open(FILENAME, "w", encoding="utf-8") as f:
        # Nagłówek pliku
        f.write("USE [projekt]; -- Upewnij się, że to dobra baza\n")
        f.write("GO\n\n")
        f.write("DECLARE @MyNewOrderID INT;\n\n")
        
        print(f"Generowanie {ILOSC_REKORDOW} rekordów...")

        for i in range(ILOSC_REKORDOW):
            # 1. Losowanie danych
            customer_id = random.randint(35, 47)
            employee_id = random.randint(10, 18)
            shipper_id = random.randint(4, 6)
            
            # Losowanie daty: start + losowa liczba dni
            random_days_add = random.randint(0, DAYS_DIFF)
            required_date = START_DATE + timedelta(days=random_days_add)
            required_date_str = required_date.strftime("%Y-%m-%d")

            # 2. Budowanie tekstu zapytania
            sql_stm = (
                f"-- Rekord {i+1}\n"
                f"EXEC [dbo].[sp_NewOrder]\n"
                f"    @CustomerID = {customer_id},\n"
                f"    @EmployeeID = {employee_id},\n"
                f"    @RequiredDate = '{required_date_str}',\n"
                f"    @ShipperID = {shipper_id},\n"
                f"    @NewOrderID = @MyNewOrderID OUTPUT;\n\n"
            )
            
            # 3. Zapis do pliku
            f.write(sql_stm)

    print(f"Gotowe! Wynik zapisano w pliku: {FILENAME}")

if __name__ == "__main__":
    generate_sql_file()
```
Przykład
``` sql
EXEC [dbo].[sp_NewOrder]
    @CustomerID = 36,
    @EmployeeID = 14,
    @RequiredDate = '2026-01-25',
    @ShipperID = 4,
    @NewOrderID = @MyNewOrderID OUTPUT;
```

## Generowanie elementów tabeli OrderDetails
Do utworzonych zamówień dodane zostały produkty przy użyciu wygenerowanego w python kodu SQL

```python
import random

# Konfiguracja zakresów
FILENAME = "insert_positions.sql"

ORDER_START = 4
ORDER_END = 204  # Włącznie

PRODUCT_START = 12
PRODUCT_END = 25 # Włącznie

QTY_MIN = 1
QTY_MAX = 100

POSITIONS_PER_ORDER_MIN = 1
POSITIONS_PER_ORDER_MAX = 10

def generate_positions_sql():
    # Przygotowanie puli wszystkich możliwych produktów
    all_products_pool = list(range(PRODUCT_START, PRODUCT_END + 1))
    
    total_inserts = 0

    with open(FILENAME, "w", encoding="utf-8") as f:
        f.write("USE [projekt];\n")
        f.write("GO\n\n")
        
        print(f"Generowanie pozycji dla zamówień {ORDER_START}-{ORDER_END}...")

        # Pętla po zamówieniach
        for order_id in range(ORDER_START, ORDER_END + 1):
            
            # 1. Liczba pozycji ile ma mieć to zamówienie
            num_positions = random.randint(POSITIONS_PER_ORDER_MIN, POSITIONS_PER_ORDER_MAX)
            
            # 2. UNIKALNE produkty z puli dla tego zamówienia
            selected_products = random.sample(all_products_pool, k=num_positions)
            
            f.write(f"-- Zamówienie ID: {order_id} (Liczba pozycji: {num_positions})\n")
            
            # 3. Generowanie insertów dla wylosowanych produktów
            for product_id in selected_products:
                quantity = random.randint(QTY_MIN, QTY_MAX)
                
                sql_stm = (
                    f"EXEC [dbo].[sp_AddPositionToOrder]\n"
                    f"    @OrderID = {order_id},\n"
                    f"    @ProductID = {product_id},\n"
                    f"    @Quantity = {quantity};\n"
                )
                f.write(sql_stm)
                total_inserts += 1
            
            f.write("\n")

    print(f"Gotowe! Wygenerowano {total_inserts} instrukcji INSERT w pliku: {FILENAME}")

if __name__ == "__main__":
    generate_positions_sql()
```
Przykład
```sql
EXEC [dbo].[sp_AddPositionToOrder]
    @OrderID = 4,
    @ProductID = 20,
    @Quantity = 53;
```

## Generowanie elementów tabeli Delieveries wraz z aktualizacją elementów w tabeli Orders (status zamówienia)
```python
import random
import string

# Konfiguracja
FILENAME = "register_deliveries.sql"

# Zakres dostępnych OrderID
ORDER_START = 4
ORDER_END = 204 

# Ile zamówień z tego zakresu chcemy wysłać
AMOUNT_TO_SHIP = 150

# Zakres kurierów (ShipperID)
SHIPPER_MIN = 4
SHIPPER_MAX = 6

def generate_deliveries_file():
    all_orders = list(range(ORDER_START, ORDER_END + 1))
    
    orders_to_ship = random.sample(all_orders, k=min(AMOUNT_TO_SHIP, len(all_orders)))

    orders_to_ship.sort()

    with open(FILENAME, "w", encoding="utf-8") as f:
        f.write("USE [projekt]; -- Pamiętaj o ustawieniu bazy\n")
        f.write("GO\n\n")
        
        f.write("DECLARE @OutputDeliveryID INT;\n\n")
        
        print(f"Generowanie wysyłek dla {len(orders_to_ship)} losowych zamówień...")

        for order_id in orders_to_ship:
            # Losowanie danych
            shipper_id = random.randint(SHIPPER_MIN, SHIPPER_MAX)
            estimated_days = random.randint(1, 5)
            
            rand_part = random.randint(1000, 9999)
            parcel_number = f"WBL-{rand_part}-{order_id}"
            
            sql_stm = (
                f"-- Wysyłka dla zamówienia {order_id}\n"
                f"EXEC [dbo].[sp_RegisterDelivery]\n"
                f"    @OrderID = {order_id},\n"
                f"    @ShipperID = {shipper_id},\n"
                f"    @ParcelNumber = '{parcel_number}',\n"
                f"    @EstimatedDays = {estimated_days},\n"
                f"    @NewDeliveryID = @OutputDeliveryID OUTPUT;\n"
            )
            
            f.write(sql_stm)
            f.write("\n")

    print(f"Gotowe! Skrypt zapisano w pliku: {FILENAME}")

if __name__ == "__main__":
    generate_deliveries_file()
```
Przykład
``` sql
EXEC [dbo].[sp_RegisterDelivery]
    @OrderID = 5,
    @ShipperID = 4,
    @ParcelNumber = 'WBL-9218-5',
    @EstimatedDays = 4,
    @NewDeliveryID = @OutputDeliveryID OUTPUT;
```


## Pozostałe generatory
Pozostałe wygenerowane elementy bazy zostały zaimplementowane do bazy i są dostępne do wyeksportowania po poprawnym połączeniem z bazą na serwerze Azure
