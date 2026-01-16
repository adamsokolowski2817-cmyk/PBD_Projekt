# Tabele
Poniżej przedstawiono tabele utworzono w bazie danych. Na tej podstawie wygenerowany został schemat bazy.
Służą one do utworzenia bazy.
Poniższy kod został przygotowany przy użyciu narzędzia Redgate Data Modeler

## Tabele


### Table: BillOfMaterials
```sql
CREATE TABLE BillOfMaterials (
    ProductID int  NOT NULL,
    ComponentID int  NOT NULL,
    QuantityPerProduct decimal(12,4)  NOT NULL,
    CONSTRAINT QuantityPerProduct CHECK (QuantityPerProduct > 0),
    CONSTRAINT BillOfMaterials_pk PRIMARY KEY  (ProductID,ComponentID)
);

```


### Table: Complaints
```sql
CREATE TABLE Complaints (
    ComplaintID int  NOT NULL IDENTITY(1, 1),
    OrderID int  NOT NULL,
    ProductID int  NOT NULL,
    EmployeeID int  NOT NULL,
    ComplaintDate datetime  NOT NULL,
    Status varchar(30)  NOT NULL,
    CONSTRAINT Complaints_pk PRIMARY KEY  (ComplaintID)
);
```

### Table: ComponentCategories
``` sql
CREATE TABLE ComponentCategories (
    ComponentCategoryID int  NOT NULL IDENTITY(1, 1),
    CategoryName varchar(50)  NOT NULL,
    CONSTRAINT ComponentCategories_pk PRIMARY KEY  (ComponentCategoryID)
);
```

### Table: ComponentDeliveries
``` sql
CREATE TABLE ComponentDeliveries (
    ComponentDeliveryID int  NOT NULL IDENTITY(1, 1),
    ComponentID int  NOT NULL,
    Quantity decimal(12,4)  NOT NULL,
    DateOfDelivery date  NULL,
    Status varchar(30)  NOT NULL,
    CONSTRAINT Quantity CHECK (Quantity > 0),
    CONSTRAINT ComponentDeliveries_pk PRIMARY KEY  (ComponentDeliveryID)
);
```

### Table: ComponentReservations
```sql
CREATE TABLE ComponentReservations (
    ComponentReservationID int  NOT NULL,
    ComponentID int  NOT NULL,
    OrderDetailID int  NOT NULL,
    ProductionOrderID int  NOT NULL,
    ReservedQuantity decimal(12,4)  NOT NULL,
    CONSTRAINT ReservedQuantity CHECK (ReservedQuantity > 0),
    CONSTRAINT ComponentReservations_pk PRIMARY KEY  (ComponentReservationID)
);
```

### Table: ComponentStockRegister
```sql
CREATE TABLE ComponentStockRegister (
    ComponentStockEntryID int  NOT NULL IDENTITY(1, 1),
    ComponentID int  NOT NULL,
    EntryDate datetime  NULL,
    Amount decimal(12,4)  NOT NULL,
    CONSTRAINT ComponentStockRegister_pk PRIMARY KEY  (ComponentStockEntryID)
);
```

### Table: ComponentStocks
```sql
CREATE TABLE ComponentStocks (
    ComponentID int  NOT NULL,
    QuantityAvailable decimal(12,4)  NOT NULL DEFAULT (0),
    QuantityReserved decimal(12,4)  NULL DEFAULT (0),
    CONSTRAINT QuantityReserved CHECK (QuantityReserved > 0 ),
    CONSTRAINT ComponentStocks_pk PRIMARY KEY  (ComponentID)
);
```

### Table: Components
```sql
CREATE TABLE Components (
    ComponentID int  NOT NULL IDENTITY(1, 1),
    ComponentName varchar(100)  NOT NULL,
    ComponentCategoryID int  NOT NULL,
    UnitOfMeasure varchar(20)  NULL,
    UnitCost decimal(19,4)  NOT NULL,
    CONSTRAINT UnitCost CHECK (UnitCost > 0),
    CONSTRAINT Components_pk PRIMARY KEY  (ComponentID)
);
```

### Table: Customers
```sql
CREATE TABLE Customers (
    CustomerID int  NOT NULL IDENTITY(1, 1),
    CompanyName varchar(100)  NULL,
    ContactName varchar(60)  NULL,
    ContactTitle varchar(60)  NULL,
    IsBusiness bit  NOT NULL,
    NIP varchar(15)  NULL,
    Address varchar(100)  NULL,
    City varchar(50)  NULL,
    Region varchar(50)  NULL,
    PostalCode varchar(20)  NULL,
    Country varchar(50)  NULL,
    Phone varchar(30)  NULL,
    Email varchar(100)  NULL,
    CONSTRAINT NIP UNIQUE (NIP),
    CONSTRAINT Customers_pk PRIMARY KEY  (CustomerID)
);
```

### Table: Deliveries
```sql
CREATE TABLE Deliveries (
    DeliveryID int  NOT NULL IDENTITY(1, 1),
    ParcelNumber varchar(30)  NULL,
    ShippingDate datetime  NULL,
    EstimatedDeliveryDate datetime  NULL,
    ShipperID int  NOT NULL,
    OrderID int  NOT NULL,
    DeliveryStatus varchar(30)  NOT NULL,
    CONSTRAINT AK_0 UNIQUE (ParcelNumber),
    CONSTRAINT Deliveries_pk PRIMARY KEY  (DeliveryID)
);
```

### Table: Departments
```sql
CREATE TABLE Departments (
    DepartmentID int  NOT NULL IDENTITY(1, 1),
    DepartmentName varchar(100)  NOT NULL,
    Address varchar(100)  NOT NULL,
    CONSTRAINT Departments_pk PRIMARY KEY  (DepartmentID)
);
```

### Table: Employees
```sql
CREATE TABLE Employees (
    EmployeeID int  NOT NULL IDENTITY(1, 1),
    LastName varchar(50)  NULL,
    FirstName varchar(50)  NULL,
    Title varchar(50)  NULL,
    BirthDate datetime  NULL,
    HireDate datetime  NULL,
    Address varchar(100)  NULL,
    City varchar(50)  NULL,
    Region varchar(50)  NULL,
    PostalCode varchar(20)  NULL,
    Country varchar(50)  NULL,
    HomePhone varchar(30)  NULL,
    DepartmentID int  NOT NULL,
    CONSTRAINT Employees_pk PRIMARY KEY  (EmployeeID)
);
```

### Table: InvoiceItems
```sql
CREATE TABLE InvoiceItems (
    InvoiceItemID int NOT NULL IDENTITY(1,1),
    InvoiceID int NOT NULL,
    ProductID int NOT NULL,
    Description varchar(200) NOT NULL,
    Quantity int NOT NULL,
    UnitNetPrice decimal(19,4) NOT NULL,
    VatRate decimal(5,2) NULL,
    DiscountPercent decimal(5,2) NULL DEFAULT (0),

    CONSTRAINT CK_InvoiceItems_Quantity
        CHECK (Quantity > 0),

    CONSTRAINT CK_InvoiceItems_UnitNetPrice
        CHECK (UnitNetPrice > 0),

    CONSTRAINT PK_InvoiceItems
        PRIMARY KEY (InvoiceItemID)
);
```


### Table: Invoices
```sql
CREATE TABLE Invoices (
    InvoiceID int  NOT NULL IDENTITY(1, 1),
    OrderID int  NOT NULL,
    InvoiceNumber varchar(50)  NOT NULL,
    InvoiceDate datetime  NOT NULL,
    DueDate datetime  NOT NULL,
    TotalNet decimal(19,4)  NOT NULL,
    TotalVat decimal(19,4)  NULL,
    TotalGross decimal(19,4)  NOT NULL,
    CONSTRAINT InvoiceNumber UNIQUE (InvoiceNumber),
    CONSTRAINT TotalNet CHECK (TotalNet > 0),
    CONSTRAINT TotalGross CHECK (TotalGross > 0),
    CONSTRAINT Invoices_pk PRIMARY KEY  (InvoiceID)
);
```

### Table: OrderDetails
```sql
CREATE TABLE OrderDetails (
    OrderDetailID int  NOT NULL IDENTITY(1, 1),
    OrderID int  NOT NULL,
    ProductID int  NOT NULL,
    UnitPrice decimal(19,4)  NOT NULL,
    Quantity int  NOT NULL,
    Discount decimal(5,2)  NULL DEFAULT (0),
    Status varchar(30)  NOT NULL,
    CONSTRAINT CK_OrderDetails_Quantity
        CHECK (Quantity > 0),

    CONSTRAINT CK_OrderDetails_UnitPrice
        CHECK (UnitPrice > 0),

    CONSTRAINT PK_OrderDetails
        PRIMARY KEY (OrderDetailID)
);
```

### Table: OrderStatus
```sql
CREATE TABLE OrderStatus (
    StatusID int  NOT NULL IDENTITY(1, 1),
    StatusCategory varchar(30)  NOT NULL,
    CONSTRAINT OrderStatus_pk PRIMARY KEY  (StatusID)
);
```

### Table: Orders
```sql
CREATE TABLE Orders (
    OrderID int  NOT NULL IDENTITY(1, 1),
    CustomerID int  NOT NULL,
    EmployeeID int  NOT NULL,
    OrderDate datetime  NOT NULL,
    RequiredDate datetime  NULL,
    ShippedDate datetime  NULL,
    ShipperID int  NOT NULL,
    Freight decimal(19,4)  NULL,
    ShipName varchar(100)  NULL,
    ShipAddress varchar(100)  NULL,
    ShipCity varchar(50)  NULL,
    ShipRegion varchar(50)  NULL,
    ShipPostalCode varchar(20)  NULL,
    ShipCountry varchar(50)  NULL,
    OrderDiscountPercent decimal(5,2)  NULL DEFAULT (0),
    StatusID int  NOT NULL,
    CONSTRAINT Orders_pk PRIMARY KEY  (OrderID)
);
```

### Table: Payments
```sql
CREATE TABLE Payments (
    PaymentID int  NOT NULL IDENTITY(1, 1),
    InvoiceID int  NOT NULL,
    PaymentDate datetime  NULL,
    Amount decimal(19,4)  NULL,
    PaymentMethod varchar(30)  NULL,
    PaymentStatus varchar(30)  NOT NULL,
    CONSTRAINT Payments_pk PRIMARY KEY  (PaymentID)
);
```

### Table: ProductCategories
```sql
CREATE TABLE ProductCategories (
    CategoryID int  NOT NULL IDENTITY(1, 1),
    CategoryName varchar(50)  NOT NULL,
    Description text  NULL,
    ProductVatRate float  NOT NULL,
    CONSTRAINT ProductCategories_pk PRIMARY KEY  (CategoryID)
);
```

### Table: ProductStockRegister
```sql
CREATE TABLE ProductStockRegister (
    ProductStockEntryID int  NOT NULL IDENTITY(1, 1),
    ProductID int  NOT NULL,
    EntryDate datetime  NOT NULL,
    Amount decimal(12,2)  NOT NULL,
    CONSTRAINT ProductStockRegister_pk PRIMARY KEY  (ProductStockEntryID)
);
```

### Table: ProductStocks
```sql
CREATE TABLE ProductStocks (
    ProductID int  NOT NULL,
    QuantityAvailable decimal(12,2)  NOT NULL DEFAULT (0),
    QuantityReserved decimal(12,2)  NOT NULL DEFAULT (0),
    CONSTRAINT CK_ProductStocks_QuantityReserved
        CHECK (QuantityReserved >= 0),

    CONSTRAINT PK_ProductStocks
        PRIMARY KEY (ProductID)
);
```

### Table: ProductionOrderComponents
```sql
CREATE TABLE ProductionOrderComponents (
    ProductionOrderID int  NOT NULL,
    ComponentID int  NOT NULL,
    QuantityRequired decimal(12,4)  NOT NULL,
    QuantityIssued decimal(12,4)  NOT NULL DEFAULT (0),
    CONSTRAINT QuantityRequired CHECK (QuantityRequired > 0),
    CONSTRAINT ProductionOrderComponents_pk PRIMARY KEY  (ProductionOrderID,ComponentID)
);
```

### Table: ProductionOrderStatus
```sql
CREATE TABLE ProductionOrderStatus (
    ProductionOrderStatusID int  NOT NULL,
    StatusName varchar(30)  NOT NULL,
    CONSTRAINT ProductionOrderStatusID_pk PRIMARY KEY  (ProductionOrderStatusID)
);
```

### Table: ProductionOrders
```sql
CREATE TABLE ProductionOrders (
    ProductionOrderID int  NOT NULL IDENTITY(1, 1),
    ProductID int  NOT NULL,
    QuantityToProduce decimal(12,2)  NOT NULL,
    PlannedStartDate datetime  NULL,
    PlannedEndDate datetime  NULL,
    SourceOrderID int  NOT NULL,
    WorkCenterID int  NOT NULL,
    ProductionOrderStatusID int  NOT NULL,
    CONSTRAINT QuantityToProduce CHECK (QuantityToProduce > 0 ),
    CONSTRAINT ProductionOrders_pk PRIMARY KEY  (ProductionOrderID)
);
```

### Table: Products
```sql
CREATE TABLE Products (
    ProductID int  NOT NULL IDENTITY(1, 1),
    ProductName varchar(100)  NOT NULL,
    CategoryID int  NOT NULL,
    UnitPriceNet decimal(19,4)  NOT NULL,
    ProductionTimeInHours int  NOT NULL,
    ProductionCost decimal(19,4)  NOT NULL,
    CONSTRAINT UnitPriceNet CHECK (UnitPriceNet > 0),
    CONSTRAINT ProductionTime CHECK (ProductionTimeInHours > 0),
    CONSTRAINT ProductionCost CHECK (ProductionCost > 0),
    CONSTRAINT Products_pk PRIMARY KEY  (ProductID)
);
```

### Table: Shippers
```sql
CREATE TABLE Shippers (
    ShipperID int  NOT NULL IDENTITY(1, 1),
    CompanyName varchar(100)  NULL,
    Address varchar(100)  NULL,
    Phone varchar(30)  NULL,
    ContactEmail varchar(100)  NULL,
    CONSTRAINT Shippers_pk PRIMARY KEY  (ShipperID)
);
```

### Table: WorkCenterCapacity
```sql
CREATE TABLE WorkCenterCapacity (
    WorkCenterID int  NOT NULL,
    CapacityHoursPerDay float  NOT NULL,
    CONSTRAINT work_center_id PRIMARY KEY  (WorkCenterID)
);
```

### Table: WorkCenters
```sql
CREATE TABLE WorkCenters (
    WorkCenterID int  NOT NULL,
    WorkCenterName varchar(80)  NOT NULL,
    IsActive bit  NOT NULL,
    CONSTRAINT UQ_WorkCenters_Name UNIQUE (WorkCenterName),
    CONSTRAINT WorkCenters_pk PRIMARY KEY  (WorkCenterID)
);
```

### Table: WorkingCostParameters
```sql
CREATE TABLE WorkingCostParameters (
    CostParametersID int  NOT NULL,
    LabourRatePerHour money  NOT NULL,
    CONSTRAINT cost_parameters_pk PRIMARY KEY  (CostParametersID)
);
```

## Klucze obce

## ComponentReservations_Components (table: ComponentReservations)
``` sql
ALTER TABLE ComponentReservations ADD CONSTRAINT ComponentReservations_Components
    FOREIGN KEY (ComponentID)
    REFERENCES Components (ComponentID);
```

## ComponentReservations_OrderDetails (table: ComponentReservations)
``` sql
ALTER TABLE ComponentReservations ADD CONSTRAINT ComponentReservations_OrderDetails
    FOREIGN KEY (OrderDetailID)
    REFERENCES OrderDetails (OrderDetailID);
```

## ComponentReservations_ProductionOrders (table: ComponentReservations)
``` sql
ALTER TABLE ComponentReservations ADD CONSTRAINT ComponentReservations_ProductionOrders
    FOREIGN KEY (ProductionOrderID)
    REFERENCES ProductionOrders (ProductionOrderID);
```

## FK_0 (table: Employees)
``` sql
ALTER TABLE Employees ADD CONSTRAINT FK_0
    FOREIGN KEY (DepartmentID)
    REFERENCES Departments (DepartmentID);
```

## FK_1 (table: Products)
``` sql
ALTER TABLE Products ADD CONSTRAINT FK_1
    FOREIGN KEY (CategoryID)
    REFERENCES ProductCategories (CategoryID);
```

## FK_10 (table: Orders)
``` sql
ALTER TABLE Orders ADD CONSTRAINT FK_10
    FOREIGN KEY (CustomerID)
    REFERENCES Customers (CustomerID);
```

## FK_11 (table: Orders)
``` sql
ALTER TABLE Orders ADD CONSTRAINT FK_11
    FOREIGN KEY (EmployeeID)
    REFERENCES Employees (EmployeeID);
```

## FK_12 (table: Orders)
``` sql
ALTER TABLE Orders ADD CONSTRAINT FK_12
    FOREIGN KEY (ShipperID)
    REFERENCES Shippers (ShipperID);

## FK_13 (table: Orders)
``` sql
ALTER TABLE Orders ADD CONSTRAINT FK_13
    FOREIGN KEY (StatusID)
    REFERENCES OrderStatus (StatusID);
```

## FK_14 (table: OrderDetails)
``` sql
ALTER TABLE OrderDetails ADD CONSTRAINT FK_14
    FOREIGN KEY (OrderID)
    REFERENCES Orders (OrderID);
```

## FK_15 (table: OrderDetails)
``` sql
ALTER TABLE OrderDetails ADD CONSTRAINT FK_15
    FOREIGN KEY (ProductID)
    REFERENCES Products (ProductID);
```

## FK_16 (table: Complaints)
``` sql
ALTER TABLE Complaints ADD CONSTRAINT FK_16
    FOREIGN KEY (OrderID)
    REFERENCES Orders (OrderID);
```

## FK_17 (table: Complaints)
``` sql
ALTER TABLE Complaints ADD CONSTRAINT FK_17
    FOREIGN KEY (ProductID)
    REFERENCES Products (ProductID);
```

## FK_18 (table: Complaints)
``` sql
ALTER TABLE Complaints ADD CONSTRAINT FK_18
    FOREIGN KEY (EmployeeID)
    REFERENCES Employees (EmployeeID);
```

## FK_19 (table: Deliveries)
``` sql
ALTER TABLE Deliveries ADD CONSTRAINT FK_19
    FOREIGN KEY (ShipperID)
    REFERENCES Shippers (ShipperID);
```

## FK_2 (table: ProductStocks)
``` sql
ALTER TABLE ProductStocks ADD CONSTRAINT FK_2
    FOREIGN KEY (ProductID)
    REFERENCES Products (ProductID);
```

## FK_20 (table: Deliveries)
``` sql
ALTER TABLE Deliveries ADD CONSTRAINT FK_20
    FOREIGN KEY (OrderID)
    REFERENCES Orders (OrderID);
```

## FK_21 (table: Invoices)
``` sql
ALTER TABLE Invoices ADD CONSTRAINT FK_21
    FOREIGN KEY (OrderID)
    REFERENCES Orders (OrderID);
```

## FK_22 (table: InvoiceItems)
``` sql
ALTER TABLE InvoiceItems ADD CONSTRAINT FK_22
    FOREIGN KEY (InvoiceID)
    REFERENCES Invoices (InvoiceID);
```

## FK_23 (table: InvoiceItems)
``` sql
ALTER TABLE InvoiceItems ADD CONSTRAINT FK_23
    FOREIGN KEY (ProductID)
    REFERENCES Products (ProductID);
```

## FK_24 (table: Payments)
``` sql
ALTER TABLE Payments ADD CONSTRAINT FK_24
    FOREIGN KEY (InvoiceID)
    REFERENCES Invoices (InvoiceID);
```

## FK_25 (table: ProductionOrders)
``` sql
ALTER TABLE ProductionOrders ADD CONSTRAINT FK_25
    FOREIGN KEY (ProductID)
    REFERENCES Products (ProductID);
```

## FK_26 (table: ProductionOrders)
``` sql
ALTER TABLE ProductionOrders ADD CONSTRAINT FK_26
    FOREIGN KEY (SourceOrderID)
    REFERENCES Orders (OrderID);
```

## FK_27 (table: ProductionOrderComponents)
``` sql
ALTER TABLE ProductionOrderComponents ADD CONSTRAINT FK_27
    FOREIGN KEY (ProductionOrderID)
    REFERENCES ProductionOrders (ProductionOrderID);
```

## FK_28 (table: ProductionOrderComponents)
``` sql
ALTER TABLE ProductionOrderComponents ADD CONSTRAINT FK_28
    FOREIGN KEY (ComponentID)
    REFERENCES Components (ComponentID);
```

## FK_3 (table: ProductStockRegister)
``` sql
ALTER TABLE ProductStockRegister ADD CONSTRAINT FK_3
    FOREIGN KEY (ProductID)
    REFERENCES Products (ProductID);
```

## FK_4 (table: Components)
``` sql
ALTER TABLE Components ADD CONSTRAINT FK_4
    FOREIGN KEY (ComponentCategoryID)
    REFERENCES ComponentCategories (ComponentCategoryID);
```

## FK_5 (table: ComponentStocks)
``` sql
ALTER TABLE ComponentStocks ADD CONSTRAINT FK_5
    FOREIGN KEY (ComponentID)
    REFERENCES Components (ComponentID);
```

## FK_6 (table: ComponentStockRegister)
``` sql
ALTER TABLE ComponentStockRegister ADD CONSTRAINT FK_6
    FOREIGN KEY (ComponentID)
    REFERENCES Components (ComponentID);
```

## FK_7 (table: ComponentDeliveries)
``` sql
ALTER TABLE ComponentDeliveries ADD CONSTRAINT FK_7
    FOREIGN KEY (ComponentID)
    REFERENCES Components (ComponentID);
```

## FK_8 (table: BillOfMaterials)
``` sql
ALTER TABLE BillOfMaterials ADD CONSTRAINT FK_8
    FOREIGN KEY (ProductID)
    REFERENCES Products (ProductID);
```

## FK_9 (table: BillOfMaterials)
``` sql
ALTER TABLE BillOfMaterials ADD CONSTRAINT FK_9
    FOREIGN KEY (ComponentID)
    REFERENCES Components (ComponentID);
```

## ProductionOrders_ProductionOrderStatus (table: ProductionOrders)
``` sql
ALTER TABLE ProductionOrders ADD CONSTRAINT ProductionOrders_ProductionOrderStatus
    FOREIGN KEY (ProductionOrderStatusID)
    REFERENCES ProductionOrderStatus (ProductionOrderStatusID);
```

## ProductionOrders_WorkCenters (table: ProductionOrders)
``` sql
ALTER TABLE ProductionOrders ADD CONSTRAINT ProductionOrders_WorkCenters
    FOREIGN KEY (WorkCenterID)
    REFERENCES WorkCenters (WorkCenterID);
```

## WorkCenterCapacity_WorkCenters (table: WorkCenterCapacity)
``` sql
ALTER TABLE WorkCenterCapacity ADD CONSTRAINT WorkCenterCapacity_WorkCenters
    FOREIGN KEY (WorkCenterID)
    REFERENCES WorkCenters (WorkCenterID);
```

