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
- **ProductID**
- **QuantityAvailable**
- **QuantityReserved**

## ProductionOrderComponents
- **ProductionOrderID**
- **ComponentID**
- **QuantityRequired**
- **QuantityIssued**

## ProductionOrderStatus
- **ProductionOrderStatusID**
- **StatusName**

## ProductionOrders
- **ProductionOrderID**
- **ProductID**
- **QuantityToProduce**
- **PlannedStartDate**
- **PlannedEndDate**
- **SourceOrderID**
- **WorkCenterID**
- **ProductionOrderStatusID**

## Products
- **ProductID**
- **ProductName**
- **CategoryID**
- **UnitPriceNet**
- **ProductionTimeInHours**
- **ProductionCost**

## Shippers
- **ShipperID**
- **CompanyName**
- **Address**
- **Phone**
- **ContactEmail**

## WorkCenterCapacity
- **WorkCenterID**
- **CapacityHoursPerDay**

## WorkCenters
- **WorkCenterID**
- **WorkCenterName**
- **IsActive**

## WorkingCostParameters
- **CostParametersID**
- **LabourRatePerHour**