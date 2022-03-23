# Tables

## NumConstants
**Numerical constants, e.g. value of discounts.**
* Name - name of a constant,
* Value - value of a constant.


    CREATE TABLE [dbo].[NumConstants](
    [Name] [varchar](70) NOT NULL,
    [Value] [int] NOT NULL,
     CONSTRAINT [PK_NumConstants] PRIMARY KEY CLUSTERED
    (
        [Name] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, 
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF)
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO

## StrConstants
**Text constants, e.g. name of a restaurant.**
* Name - name of a constant,
* Value - value of a constant.


    CREATE TABLE [dbo].[StrConstants](
    [Name] [varchar](70) NOT NULL,
    [Value] [varbinary](225) NOT NULL,
     CONSTRAINT [PK_StrConstants] PRIMARY KEY CLUSTERED
    (
        [Name] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF)
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO

## TimeConstants
**Time constants, e.g. opening hours.**
* Name - name of a constant,
* Value - value of a constant.


    CREATE TABLE [dbo].[TimeConstants](
    [Name] [varchar](70) NOT NULL,
    [Value] [time](7) NOT NULL,
     CONSTRAINT [PK_TimeConstants] PRIMARY KEY CLUSTERED
    (
        [Name] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF)
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO

## Customers

**Basic info about all customers.**

* CustomerID - primary key, exactly once repeats in Company or Individual table,
* Email - customer email,
* Phone - customer telephone,
* Address - whole customer address (city, street, number as a string).


    CREATE TABLE [dbo].[Customers](
        [CustomerID] [int] IDENTITY(1,1) NOT NULL,
        [Email] [varchar](70) NOT NULL,
        [Phone] [varchar](9) NOT NULL,
        [Address] [varchar](100) NOT NULL,
     CONSTRAINT [PK_Customers] PRIMARY KEY CLUSTERED
    (
        [CustomerID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF)
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO

## Company
**Table about company customers.**
* CustomerID - primary key,
* CompanyName - name of a company,
* NIP - customer NIP.


    CREATE TABLE [dbo].[Company](
        [CustomerID] [int] NOT NULL,
        [CompanyName] [varchar](70) NOT NULL,
        [NIP] [varchar](10) NULL,
     CONSTRAINT [PK_Company] PRIMARY KEY CLUSTERED
    (
        [CustomerID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF)
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO
    
    ALTER TABLE [dbo].[Company]  WITH CHECK ADD  CONSTRAINT [FK_Company_Customers] 
    FOREIGN KEY([CustomerID])
    REFERENCES [dbo].[Customers] ([CustomerID])
    GO
    
    ALTER TABLE [dbo].[Company] CHECK CONSTRAINT [FK_Company_Customers]
    GO

## Individual
**Table about individual customers.**
* CustomerID - primary key,
* FirstName, LastName - personal data,
* PermanentDiscount - boolean whether a customer has permanent discount,
* TemporaryDiscount - date of expiration of single discount (realised on first order),
* OrdersTotalValue - total value of orders (value held for temporary discount check - 
it is zeroed when temporary discount is granted. Then the customer has to collect this
value again to have another discount),
* PreviousOrders - number of previous order, held to decide whether the customer is able
to pay in the restaurant for online order.


    CREATE TABLE [dbo].[Individual](
        [CustomerID] [int] NOT NULL,
        [FirstName] [varchar](70) NOT NULL,
        [LastName] [varchar](70) NOT NULL,
        [PernamentDiscount] [bit] NOT NULL,
        [TemporaryDiscount] [date] NOT NULL,
        [OrdersTotalValue] [decimal](18, 2) NOT NULL,
        [PreviousOrders] [int] NOT NULL,
     CONSTRAINT [PK_Individual] PRIMARY KEY CLUSTERED
    (
        [CustomerID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF)
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO
    
    ALTER TABLE [dbo].[Individual]  WITH CHECK ADD  CONSTRAINT 
    [FK_Individual_Customers] FOREIGN KEY([CustomerID])
    REFERENCES [dbo].[Customers] ([CustomerID])
    GO
    
    ALTER TABLE [dbo].[Individual] CHECK CONSTRAINT [FK_Individual_Customers]
    GO
    
    ALTER TABLE [dbo].[Individual]  WITH CHECK ADD  CONSTRAINT [previous_orders_min]
    CHECK  (([PreviousOrders]>=(0)))
    GO
    
    ALTER TABLE [dbo].[Individual] CHECK CONSTRAINT [previous_orders_min]
    GO
    
    ALTER TABLE [dbo].[Individual]  WITH CHECK ADD  CONSTRAINT [temp_disc_due_date]
    CHECK  (([TemporaryDiscount]>=getdate()))
    GO
    
    ALTER TABLE [dbo].[Individual] CHECK CONSTRAINT [temp_disc_due_date]
    GO
    
    ALTER TABLE [dbo].[Individual]  WITH CHECK ADD  CONSTRAINT [total_value_min] 
    CHECK  (([OrdersTotalValue]>=(0)))
    GO
    
    ALTER TABLE [dbo].[Individual] CHECK CONSTRAINT [total_value_min]
    GO

## Tables
**Table containing table info to help managing reservations in a restaurant.**

* TableID - primary key,
* Seats - number of seats (we have one table with 0 seats to manage take-out orders),
* InactiveDueDate - date till when table is out of use (null value means active table).


    CREATE TABLE [dbo].[Tables](
    [TableID] [int] IDENTITY(1,1) NOT NULL,
    [Seats] [int] NOT NULL,
    [InactiveDueDate] [date] NULL,
     CONSTRAINT [PK_Tables] PRIMARY KEY CLUSTERED
    (
        [TableID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) 
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO
    
    ALTER TABLE [dbo].[Tables]  WITH CHECK ADD  CONSTRAINT [seat_num] CHECK  
    (([Seats]>(0)))
    GO
    
    ALTER TABLE [dbo].[Tables] CHECK CONSTRAINT [seat_num]
    GO
    
    ALTER TABLE [dbo].[Tables]  WITH CHECK ADD  CONSTRAINT [table_due_date] CHECK  
    (([InactiveDueDate]>=getdate()))
    GO
    
    ALTER TABLE [dbo].[Tables] CHECK CONSTRAINT [table_due_date]
    GO

## Reservations
**Table containing basic reservation info.**

* ReservationID - primary key,
* OrderID - ID of order,
* CompletionDate - start of reservation,
* EstimatedTime - estimated end of reservation,
* EmployeeID - ID of employee who accepted the reservation.


    CREATE TABLE [dbo].[Reservations](
    [ReservationID] [int] IDENTITY(1,1) NOT NULL,
    [OrderID] [int] NOT NULL,
    [EmployeeID] [int] NULL,
    [CompletionDate] [datetime] NOT NULL,
    [EstimatedTime] [datetime] NULL,
     CONSTRAINT [PK_Reservations] PRIMARY KEY CLUSTERED
    (
        [ReservationID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF)
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO
    
    ALTER TABLE [dbo].[Reservations]  WITH CHECK ADD  CONSTRAINT 
    [FK_Reservations_Employees] FOREIGN KEY([EmployeeID])
    REFERENCES [dbo].[Employees] ([EmployeeID])
    GO
    
    ALTER TABLE [dbo].[Reservations] CHECK CONSTRAINT [FK_Reservations_Employees]
    GO
    
    ALTER TABLE [dbo].[Reservations]  WITH CHECK ADD  CONSTRAINT 
    [FK_Reservations_Orders] FOREIGN KEY([OrderID])
    REFERENCES [dbo].[Orders] ([OrderID])
    GO
    
    ALTER TABLE [dbo].[Reservations] CHECK CONSTRAINT [FK_Reservations_Orders]
    GO
    
    ALTER TABLE [dbo].[Reservations]  WITH CHECK ADD  CONSTRAINT [reserv_date] CHECK
    (([EstimatedTime]>[CompletionDate]))
    GO
    
    ALTER TABLE [dbo].[Reservations] CHECK CONSTRAINT [reserv_date]
    GO

## ReservationDetails

**Table to hold data which tables are reserved by each reservation.**

* OrderID, TableID - primary keys.


    CREATE TABLE [dbo].[ReservationDetails](
    [OrderID] [int] NOT NULL,
    [TableID] [int] NOT NULL,
     CONSTRAINT [PK_ReservationDetails] PRIMARY KEY CLUSTERED
    (
        [OrderID] ASC,
        [TableID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, 
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF)
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO
    
    ALTER TABLE [dbo].[ReservationDetails]  WITH CHECK ADD  CONSTRAINT 
    [FK_ReservationDetails_Reservations] FOREIGN KEY([OrderID])
    REFERENCES [dbo].[Reservations] ([OrderID])
    GO
    
    ALTER TABLE [dbo].[ReservationDetails] CHECK CONSTRAINT 
    [FK_ReservationDetails_Reservations]
    GO
    
    ALTER TABLE [dbo].[ReservationDetails]  WITH CHECK ADD  CONSTRAINT 
    [FK_ReservationDetails_Tables] FOREIGN KEY([TableID])
    REFERENCES [dbo].[Tables] ([TableID])
    GO
    
    ALTER TABLE [dbo].[ReservationDetails] CHECK CONSTRAINT 
    [FK_ReservationDetails_Tables]
    GO

## Orders
**Basic info about the orders.**
* OrderId - primary key,
* OrderDate - date of creating the order,
* CustomerId - customer that made this order,
* Completed - boolean whether order is completed,
* Discount - number between 0 and 1 to determine value of total discount,
* Description - special notes from customer,
* InvoiceID - ID of invoice (null means no invoice)


    CREATE TABLE [dbo].[Orders](
        [OrderID] [int] IDENTITY(1,1) NOT NULL,
        [OrderDate] [date] NOT NULL,
        [CustomerID] [int] NOT NULL,
        [Completed] [bit] NOT NULL,
        [Discount] [decimal](3, 2) NOT NULL,
        [Description] [text] NOT NULL,
        [InvoiceID] [int] NULL,
     CONSTRAINT [PK_Orders] PRIMARY KEY CLUSTERED
    (
        [OrderID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) 
    ON [PRIMARY]
    ) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
    GO
    
    ALTER TABLE [dbo].[Orders]  WITH CHECK ADD  CONSTRAINT [FK_Orders_Customers] 
    FOREIGN KEY([CustomerID])
    REFERENCES [dbo].[Customers] ([CustomerID])
    GO
    
    ALTER TABLE [dbo].[Orders] CHECK CONSTRAINT [FK_Orders_Customers]
    GO
    
    ALTER TABLE [dbo].[Orders]  WITH CHECK ADD  CONSTRAINT [FK_Orders_Invoices]
    FOREIGN KEY([InvoiceID])
    REFERENCES [dbo].[Invoices] ([InvoiceID])
    GO
    
    ALTER TABLE [dbo].[Orders] CHECK CONSTRAINT [FK_Orders_Invoices]
    GO
    
    ALTER TABLE [dbo].[Orders]  WITH CHECK ADD  CONSTRAINT [discount_range] CHECK  
    (([Discount]<=(1) AND [Discount]>=(0)))
    GO
    
    ALTER TABLE [dbo].[Orders] CHECK CONSTRAINT [discount_range]
    GO

## OrderDetails
**Dished of each order.**
* OrderID, DishID - primary keys,
* Quantity - amount of certain dish,
* Price - price of a dish.


    CREATE TABLE [dbo].[OrderDetails](
        [OrderID] [int] NOT NULL,
        [DishID] [int] NOT NULL,
        [Quantity] [int] NOT NULL,
        [Price] [decimal](18, 2) NOT NULL,
     CONSTRAINT [PK_OrderDetails] PRIMARY KEY CLUSTERED
    (
        [OrderID] ASC,
        [DishID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF)
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO
    
    ALTER TABLE [dbo].[OrderDetails]  WITH CHECK ADD  CONSTRAINT 
    [FK_OrderDetails_Dishes] FOREIGN KEY([DishID])
    REFERENCES [dbo].[Dishes] ([DishID])
    GO
    
    ALTER TABLE [dbo].[OrderDetails] CHECK CONSTRAINT [FK_OrderDetails_Dishes]
    GO
    
    ALTER TABLE [dbo].[OrderDetails]  WITH CHECK ADD  CONSTRAINT
    [FK_OrderDetails_Orders] FOREIGN KEY([OrderID])
    REFERENCES [dbo].[Orders] ([OrderID])
    ON UPDATE CASCADE
    ON DELETE CASCADE
    GO
    
    ALTER TABLE [dbo].[OrderDetails] CHECK CONSTRAINT [FK_OrderDetails_Orders]
    GO
    
    ALTER TABLE [dbo].[OrderDetails]  WITH CHECK ADD  CONSTRAINT [order_price_min]
    CHECK  (([Price]>=(0)))
    GO
    
    ALTER TABLE [dbo].[OrderDetails] CHECK CONSTRAINT [order_price_min]
    GO
    
    ALTER TABLE [dbo].[OrderDetails]  WITH CHECK ADD  CONSTRAINT [order_quantity_min]
    CHECK  (([Quantity]>(0)))
    GO
    
    ALTER TABLE [dbo].[OrderDetails] CHECK CONSTRAINT [order_quantity_min]
    GO

## Dishes
**Table holding dish info.**
* DishID- - primary key,
* Name - name of a dish,
* CategoryID - ID of dish category.


    CREATE TABLE [dbo].[Dishes](
        [DishID] [int] IDENTITY(1,1) NOT NULL,
        [Name] [varchar](70) NOT NULL,
        [CategoryID] [int] NOT NULL,
        [Discontinued] [bit] NOT NULL,
     CONSTRAINT [PK_Dishes] PRIMARY KEY CLUSTERED
    (
        [DishID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) 
    ON [PRIMARY],
     CONSTRAINT [IX_Dishes] UNIQUE NONCLUSTERED
    (
        [DishID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, 
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF)
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO
    
    ALTER TABLE [dbo].[Dishes]  WITH CHECK ADD  CONSTRAINT [FK_Dishes_Categories] 
    FOREIGN KEY([CategoryID])
    REFERENCES [dbo].[Categories] ([CategoryID])
    GO
    
    ALTER TABLE [dbo].[Dishes] CHECK CONSTRAINT [FK_Dishes_Categories]
    GO

## Categories
**Dish categories.**
* CategoryID - primary key,
* CategoryName - name of a category.


    CREATE TABLE [dbo].[Categories](
        [CategoryID] [int] IDENTITY(1,1) NOT NULL,
        [CategoryName] [varchar](70) NOT NULL,
     CONSTRAINT [PK_Categories] PRIMARY KEY CLUSTERED
    (
        [CategoryID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF)
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO

## Menu
**Table for basic info about menu.**
* MenuID - primary key,
* EmployeeId - ID of the creator of menu,
* Description - short menu description,
* StartDate, EndDate - interval of time when this menu is in use (no more than 2 weeks). Different menus cannot have
these intervals overlapping.
    

    CREATE TABLE [dbo].[Menu](
    [MenuID] [int] IDENTITY(1,1) NOT NULL,
    [EmployeeID] [int] NOT NULL,
    [Description] [text] NULL,
    [StartDate] [date] NOT NULL,
    [EndDate] [date] NOT NULL,
     CONSTRAINT [PK_Menu] PRIMARY KEY CLUSTERED
    (
        [MenuID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, 
    ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
    ) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
    
    GO
    
    ALTER TABLE [dbo].[Menu]  WITH CHECK ADD  CONSTRAINT [FK_Menu_Employees] FOREIGN KEY([EmployeeID])
    REFERENCES [dbo].[Employees] ([EmployeeID])
    GO
    
    ALTER TABLE [dbo].[Menu] CHECK CONSTRAINT [FK_Menu_Employees]
    GO
    
    ALTER TABLE [dbo].[Menu]  WITH CHECK ADD  CONSTRAINT [max_menu_interval] CHECK  
    ((datediff(day,[EndDate],[StartDate])<=(14)))
    GO
    
    ALTER TABLE [dbo].[Menu] CHECK CONSTRAINT [max_menu_interval]
    GO
    
    ALTER TABLE [dbo].[Menu]  WITH CHECK ADD  CONSTRAINT [menu_date] CHECK  
    (([EndDate]>[StartDate]))
    GO
    
    ALTER TABLE [dbo].[Menu] CHECK CONSTRAINT [menu_date]
    GO

## MenuDetails

**Table containing dishes for every menu.**
* MenuID, DishID - primary keys,
* Price - price of a dish


    CREATE TABLE [dbo].[MenuDetails](
    [MenuID] [int] NOT NULL,
    [DishID] [int] NOT NULL,
    [Price] [decimal](18, 2) NOT NULL,
     CONSTRAINT [PK_MenuDetails] PRIMARY KEY CLUSTERED
    (
        [MenuID] ASC,
        [DishID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, 
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) 
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO
    
    ALTER TABLE [dbo].[MenuDetails]  WITH CHECK ADD  CONSTRAINT 
    [FK_MenuDetails_Dishes] FOREIGN KEY([DishID])
    REFERENCES [dbo].[Dishes] ([DishID])
    GO
    
    ALTER TABLE [dbo].[MenuDetails] CHECK CONSTRAINT [FK_MenuDetails_Dishes]
    GO
    
    ALTER TABLE [dbo].[MenuDetails]  WITH CHECK ADD  CONSTRAINT 
    [FK_MenuDetails_Menu] FOREIGN KEY([MenuID])
    REFERENCES [dbo].[Menu] ([MenuID])
    GO
    
    ALTER TABLE [dbo].[MenuDetails] CHECK CONSTRAINT [FK_MenuDetails_Menu]
    GO
    
    ALTER TABLE [dbo].[MenuDetails]  WITH CHECK ADD  CONSTRAINT [menu_price_min] 
    CHECK  (([Price]>(0)))
    GO
    
    ALTER TABLE [dbo].[MenuDetails] CHECK CONSTRAINT [menu_price_min]
    GO

## Employees
**Info about restaurant staff.**
* EmployeeID - primary key,
* FirstName, LastName - personal data,
* Title- title of employee,
* BirthDate - date of birth,
* HireDate - date of start of work,
* Address - Street, number,
* City - city of living,
* Phone - phone number,
* Manager - whether the employee is a manager or not.


    CREATE TABLE [dbo].[Employees](
        [EmployeeID] [int] IDENTITY(1,1) NOT NULL,
        [FirstName] [varchar](70) NOT NULL,
        [LastName] [varchar](70) NOT NULL,
        [Title] [varchar](70) NOT NULL,
        [BirthDate] [date] NOT NULL,
        [HireDate] [date] NOT NULL,
        [Address] [varchar](75) NOT NULL,
        [City] [varchar](70) NOT NULL,
        [Phone] [varchar](9) NOT NULL,
        [Manager] [bit] NOT NULL,
     CONSTRAINT [PK_Employees] PRIMARY KEY CLUSTERED
    (
        [EmployeeID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) 
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO
    
    ALTER TABLE [dbo].[Employees]  WITH CHECK ADD  CONSTRAINT [DateConstraint] CHECK
    (([HireDate]>[BirthDate]))
    GO
    
    ALTER TABLE [dbo].[Employees] CHECK CONSTRAINT [DateConstraint]
    GO

#Invoices
**Table of order invoices.**
* InvoiceID - primary key,
* InvoiceDate - date when the invoice was generated,
* Address - city, street, number of customer.


    CREATE TABLE [dbo].[Invoices](
        [InvoiceID] [int] IDENTITY(1,1) NOT NULL,
        [InvoiceDate] [date] NOT NULL,
        [Address] [varchar](100) NOT NULL,
     CONSTRAINT [PK_Invoices] PRIMARY KEY CLUSTERED
    (
        [InvoiceID] ASC
    )
    WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF,
    ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF)
    ON [PRIMARY]
    ) ON [PRIMARY]
    GO