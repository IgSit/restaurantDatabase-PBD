# Procedures and functions
### addReservation(orderID, completionDate)
    CREATE PROCEDURE addReservation
        @orderID int,
        @completionDate datetime,
    AS
    BEGIN
        SET NOCOUNT ON;
    
        INSERT INTO Reservations (OrderID, CompletionDate)  -- rest of the values will be null
        VALUES (@orderID, @completionDate)
    END
    GO

### addOrder(customerID, DishInfo, description, orderDate)

    CREATE PROCEDURE addOrder
        -- Add the parameters for the stored procedure here
        @customerID int,
        @dishes DishInfo READONLY,
        @description text,
        @orderDate date
    AS
    BEGIN
        SET NOCOUNT ON;

    IF (@orderDate IS NULL)
        BEGIN
            SET @orderDate = GETDATE()
        END

    -- customer not in base
    IF NOT (@customerID IN (SELECT customerID FROM Customers)) 
    BEGIN
        RETURN 'Invalid CustomerID. Customer not found';
    END

    -- not every dish in current menu
    IF EXISTS((SELECT DishID FROM @dishes) EXCEPT (SELECT DishID FROM currentMenu(@orderDate)))  
    BEGIN
        RETURN 'Invalid dishes. Some of the dishes are not available in current menu.';
    END

    -- seafood in order
    IF EXISTS(SELECT d.DishID FROM @dishes d INNER JOIN Dishes ON d.DishID = Dishes.DishID
              INNER JOIN Categories C ON Dishes.CategoryID = C.CategoryID
              WHERE CategoryName = 'Seafood')  
        BEGIN
            IF ( DATEPART(weekday, GETDATE()) < 4 OR  DATEPART(weekday, GETDATE()) = 7)
            BEGIN
                RETURN 'You can only order seafood from Thursday to Saturday.'
            END
            IF (DATEDIFF(day, @orderDate, GETDATE()) < 2)
            BEGIN
                RETURN 'Too late to order seafood. Postphone your order.'
            END
        END

    DECLARE @discount decimal(3,2)
    EXEC @discount = getDiscount @customerID
    -- Add new order
    BEGIN TRAN
        BEGIN TRY
            INSERT INTO Orders (OrderDate, CustomerID, Discount, Completed, Description)
            VALUES (@orderDate, @customerID, @discount, 0, @description)
            -- Add order details
            INSERT INTO OrderDetails( OrderID, DishID, Quantity, Price)
                SELECT IDENT_CURRENT('Orders'), * FROM @dishes
            COMMIT TRANSACTION
        END TRY
    BEGIN CATCH
        -- if there was error rollback changes
        ROLLBACK TRANSACTION
    END CATCH
    END 

### confirmReservation(orderID, estimatedTime, TableList, employeeID)

    CREATE PROCEDURE confirmReservation
      @order int,
      @estimatedTime datetime,
      @tables TableList READONLY,
      @employee int
    AS
    BEGIN
        SET NOCOUNT ON;

    -- Check if order exists in Orders and Reservations
    IF NOT (@order in (SELECT OrderID FROM Orders) AND @order in (SELECT OrderID FROM Reservations))
        BEGIN
            RETURN 'Invalid OrderID. OrderID not found.';
        END

    -- Check if given employee exists
    IF @employee NOT IN (SELECT EmployeeID FROM Employees)
        BEGIN
            RETURN 'Invalid EmployeeID. Employee not found.';
        END

    BEGIN TRAN
        BEGIN TRY
            -- Check if all tables are free in specified time
            DECLARE @startTime datetime;
            DECLARE @freeTables TABLE(TableID INT);

            SET @startTime = (SELECT CompletionDate 
            FROM Reservations WHERE @order = OrderID);
            INSERT INTO @freeTables EXEC free_tables @startTime, @estimatedTime;

            IF NOT ((SELECT COUNT(*) FROM @tables) = (SELECT COUNT(*) FROM @tables T
                    INNER JOIN @freeTables F on F.TableID = T.TableID))
            BEGIN
                RETURN 'Not all selected tables are available at that time.';
            END
            -- Update reservations
            UPDATE Reservations
            SET EmployeeID = @employee, EstimatedTime = @estimatedTime
            WHERE OrderID = @order

            -- Add reservation details
            INSERT INTO ReservationDetails (OrderID, TableID)
                SELECT @order, TableID FROM @tables
            -- if no errors commit transaction
            COMMIT TRANSACTION
        END TRY
        BEGIN CATCH
            -- if there was error rollback changes
            ROLLBACK TRANSACTION
        END CATCH
    END
    GO

### addDish(name, categoryID)
**Add new dish to database.**

    CREATE PROCEDURE addDish
       @Name varchar(70),
       @CategoryID int,
      @Discontinued Bit = FALSE
    AS
    BEGIN
       SET NOCOUNT ON;
       INSERT INTO Dishes (Name, CategoryID, Discontinued, DishID)
       VALUES (@Name, @CategoryID, @Discontinued, (SELECT MAX(DishID) FROM Dishes 
       GROUP BY DishID)+1)
    END
    GO

### validateMenu(startDate, endDate, MenuList, employeeID)
**Auxiliary function to addMenu procedure (below).**

    CREATE FUNCTION validateMenu
    (
        @StartDate date,
        @EndDate date,
        @Dishes MenuList READONLY,
        @EmployeeID int
    
    )
    RETURNS BIT
    AS
    BEGIN
        IF (EXISTS(SELECT * FROM Menu WHERE StartDate BETWEEN @StartDate AND @EndDate))
        BEGIN
            RETURN 0
        END
        IF (EXISTS(SELECT * FROM Menu WHERE EndDate BETWEEN @StartDate AND @EndDate))
        BEGIN
            RETURN 0
        END
        IF ((SELECT Manager FROM Employees AS e WHERE e.EmployeeID = @EmployeeID) = 1)
        BEGIN
            RETURN 0
        END
        IF (SELECT COUNT(*) FROM @Dishes AS d WHERE d.DishID in 
           (SELECT DishID FROM MenuDetails as md WHERE md.MenuID in     
               (SELECT MenuID FROM Menu 
                WHERE EndDate = DATEADD(day, -1, @StartDate)))) <= (SELECT COUNT(*)/2 FROM @Dishes)
        BEGIN
            RETURN 0
        END
        IF  ((SELECT COUNT(*) FROM @Dishes AS d WHERE d.DishID in 
            (SELECT DishID FROM MenuDetails as md WHERE md.MenuID in 
            (SELECT MenuID FROM Menu 
            WHERE StartDate = DATEADD(day, 1, @EndDate)))) <= (SELECT COUNT(*)/2 FROM @Dishes))
        BEGIN
            RETURN 0
        END
        RETURN 1
    END
    GO

### addMenu(employeeID, description, startDate, endDate, MenuList)

    CREATE PROCEDURE [dbo].[addMenu]
       @EmployeeID int,
       @Description text,
       @StartDate date,
       @EndDate date = null,
       @Dishes MenuList READONLY
    
    AS
    IF (dbo.validateMenu(@StartDate, @EndDate, @Dishes, @EmployeeID) = 1)
     BEGIN
         BEGIN TRAN
         BEGIN TRY
             IF (@EndDate IS NULL)
                 BEGIN
                      SET @EndDate = DATEADD(day, 14, @StartDate)
                 END
              INSERT INTO Menu (EmployeeID, Description, StartDate, EndDate)
              VALUES (@EmployeeID, @Description, @StartDate, @EndDate)
              INSERT INTO MenuDetails
              SELECT IDENT_CURRENT('Menu'), DishID, Price FROM @Dishes
             -- if no erros commit transaction
             COMMIT TRANSACTION
         END TRY
          BEGIN CATCH
              -- if there was error rollback changes
              ROLLBACK TRANSACTION
          END CATCH
     END
    GO

### createMonthlyInvoice(customerID, date)
**Creates single invoice for every order in given month that didn't have an invoice already.**

    CREATE PROCEDURE [dbo].[createMonthlyInvoice]
      @CustomerID int,
      @Date date
    AS
    BEGIN
      SET NOCOUNT ON;
    
      IF (NOT @CustomerID IN (SELECT CustomerID FROM Customers))
      BEGIN
         RETURN 'Invalid CustomerID. Customer not found'
      END
    
      -- check if requested order exists
      IF (NOT EXISTS (SELECT 1 FROM Orders 
                      WHERE CustomerID = @CustomerID AND InvoiceID IS NULL 
                      AND MONTH(OrderDate) = MONTH(@Date) AND YEAR(OrderDate) = YEAR(@Date) 
                      AND Completed = 1 ) )
          BEGIN
               RETURN 'No orders to factorize';
          END
      
       BEGIN TRAN
          BEGIN TRY
              BEGIN
               DECLARE @CustomerName VARCHAR(70)
               IF (EXISTS (SELECT 1 FROM Individual WHERE CustomerID = @CustomerID) )
                  SET @CustomerName = (SELECT FirstName + ' ' + LastName 
                                       FROM Individual 
                                       WHERE CustomerID = @CustomerID)
               ELSE
                  SET @CustomerName = (SELECT CompanyName 
                                       FROM Company 
                                       WHERE CustomerID = @CustomerID)         
    
                  INSERT INTO Invoices (InvoiceDate, Address, CompanyName)
                  VALUES (CAST(GETDATE() AS Date), 
                  (SELECT Address FROM Customers WHERE CustomerID = @CustomerID), @CustomerName)
              
                  UPDATE Orders
                  SET InvoiceID = IDENT_CURRENT('Invoices')
                  WHERE CustomerID = @CustomerID AND InvoiceID IS NULL 
                  AND MONTH(OrderDate) = MONTH(@Date) AND YEAR(OrderDate) = YEAR(@Date) 
                  AND Completed = 1
               END
              -- if no errors commit transaction
              COMMIT TRANSACTION
          END TRY
           BEGIN CATCH
               -- if there was error rollback changes
               ROLLBACK TRANSACTION
           END CATCH
    END
    go

### createInvoice(orderID)
**Creates invoice for single order.**

    CREATE PROCEDURE createInvoice
      @OrderID int
    
    AS
    BEGIN
      SET NOCOUNT ON;
    
      -- check if requested order exists
      IF @OrderID NOT IN (SELECT OrderID FROM Orders)
          BEGIN
               RETURN 'Invalid OrderID. OrderID not found.';
           END
      
       BEGIN TRAN
          BEGIN TRY
              IF ((SELECT InvoiceID FROM Orders 
                  WHERE @OrderID = OrderID) IS NULL AND 
                  EXISTS(SELECT * FROM Orders WHERE OrderID = @OrderID AND Completed = 1))
                      BEGIN
                          INSERT INTO Invoices (InvoiceDate, Address)
                          VALUES (CAST(GETDATE() AS Date), 
                          (SELECT Address FROM Customers WHERE CustomerID IN 
                            (SELECT CustomerID from Orders where OrderID = @OrderID)))
                  
                          UPDATE Orders
                          SET InvoiceID = IDENT_CURRENT('Invoices')
                          WHERE OrderID = @OrderID
                      END
              -- if no errors commit transaction
              COMMIT TRANSACTION
          END TRY
           BEGIN CATCH
               -- if there was error rollback changes
               ROLLBACK TRANSACTION
           END CATCH
    END
    GO


### generateSummarizedInvoices(startDate, endDate, customerID)
**For each order generates single invoice in a given period of time.**

    CREATE PROCEDURE generateSummarizedInvoices
        @startDate date,
        @endDate date,
        @customerID int
    AS
    BEGIN
        SET NOCOUNT ON;
    
        IF (NOT @CustomerID IN (SELECT CustomerID FROM Customers))
        BEGIN
            RETURN 'Invalid CustomerID. Customer not found'
        END
    
        IF (NOT EXISTS(SELECT * FROM Orders 
           WHERE CustomerID = @customerID AND InvoiceID IS NULL AND Completed = 1 
           AND (OrderDate BETWEEN @startDate AND @endDate)))
                BEGIN
                    RETURN 'No orders to factorize';
                END
    
        DECLARE @param INT
    
        DECLARE curs CURSOR LOCAL FAST_FORWARD FOR
            SELECT OrderID FROM Orders WHERE CustomerID = @customerID AND 
            (OrderDate BETWEEN @startDate AND @endDate)
            AND OrderID NOT IN (SELECT OrderID FROM InvoiceDetails)
    
        OPEN curs
    
        FETCH NEXT FROM curs INTO @param
    
        WHILE @@FETCH_STATUS = 0
        BEGIN
            EXEC getInvoice @param
            FETCH NEXT FROM curs INTO @param
        END
    
        CLOSE curs
        DEALLOCATE curs
    END
    GO


### generateBill(orderID)
**Calculate value of given order.**

    CREATE PROCEDURE generateBill
         @orderID int
    AS
    BEGIN
        SET NOCOUNT ON;
    
        UPDATE Orders
        SET Completed = 1
        WHERE OrderID = @orderID;
        
        DECLARE @orderValue decimal(18,2)
        SELECT @orderValue =  SUM((1 - Discount) * Quantity * Price)
        FROM OrderDetails
        INNER JOIN Orders on Orders.OrderID = OrderDetails.OrderID
        WHERE Orders.OrderID = @orderID;
    
        RETURN @orderValue;
    END

### getInvoice(orderID)
**Generate invoice and return its fields.**

    CREATE PROCEDURE getInvoice
      @OrderID int
    AS
    BEGIN
      SET NOCOUNT ON;
    
       -- Insert statements for procedure here
    
      EXEC createInvoice @OrderID
       DECLARE @billValue decimal(18,2)
       EXEC @billValue = generateBill @OrderID
    
       SELECT I.InvoiceID, @billValue AS Total, Address, InvoiceDate, 
       Name, Quantity, Price FROM Invoices I
       INNER JOIN Orders O on I.InvoiceID = O.InvoiceID
       INNER JOIN OrderDetails OD ON O.OrderID = OD.OrderID
       INNER JOIN Dishes ON Dishes.DishID = OD.DishID
    
       RETURN
    END
    GO

### closeTable (tableID, inactiveDueDate)
**Mark the table as inactive.**

    CREATE PROCEDURE closeTable
        @tableID int,
        @inactiveDue date
    AS
    BEGIN
        SET NOCOUNT ON;
    
        UPDATE Tables
        SET InactiveDueDate = @inactiveDue
        WHERE TableID = @tableID
    END
    GO


### freeTables(startDateTime, endDateTime)
**Returns free tables in a period of time.**

    CREATE FUNCTION freeTables
    (    
        @start datetime,
        @end datetime
    )
    RETURNS TABLE
    AS
    RETURN
    (
        SELECT TableID FROM Tables
        WHERE (InactiveDueDate is null OR InactiveDueDate < @start) AND 
        TableID NOT IN   (
            SELECT TableID FROM ReservationDetails
            INNER JOIN Reservations on Reservations.OrderID = ReservationDetails.OrderID
            WHERE (@start BETWEEN CompletionDate AND EstimatedTime) OR 
            (@end BETWEEN CompletionDate AND EstimatedTime)
    )
    GO


### currentMenu(date)
**Menu active during given date.**

    CREATE FUNCTION currentMenu(@date date)
     
    RETURNS TABLE
    AS
    RETURN
    (
        SELECT Dishes.Name, MenuDetails.Price
        FROM Dishes
        INNER JOIN MenuDetails on MenuDetails.DishID = Dishes.DishID     
        INNER JOIN Menu ON Menu.MenuID = MenuDetails.MenuID
        WHERE @date BETWEEN Menu.StartDate AND Menu.EndDate)
    GO

### getDiscount(customerID)
**Find the customer's discount.**

    CREATE PROCEDURE getDiscount
        @customerID int
        AS
        BEGIN
        SET NOCOUNT ON;
        DECLARE @discount decimal(3, 2)
        IF ((SELECT TemporaryDiscount FROM Individual 
           WHERE CustomerID = @customerID) > GETDATE())
                BEGIN
                SELECT @discount = Value
                FROM NumConstants
                WHERE Name = 'R2'
        
                UPDATE Individual
                SET TemporaryDiscount = GETDATE()
                WHERE CustomerID = @customerID
                END
        ELSE
            BEGIN
            IF ((SELECT PernamentDiscount FROM Individual 
               WHERE CustomerID = @customerID) = 1)
                    BEGIN
                    SELECT @discount = Value
                    FROM NumConstants
                    WHERE Name = 'R1'
                    END
            ELSE
                BEGIN
                SELECT @discount = 0
                END
            END
        RETURN @discount
        END    
    GO

### tablesReport(startDate, endDate)
**Report of reservations per table in a period of time.**

    CREATE FUNCTION tablesReport
    ( 
      @start date,
      @end date
    )
    RETURNS TABLE
    AS
    RETURN
    (
      SELECT T.TableID, COUNT(*) as TotalReservations FROM Tables as T
      INNER JOIN ReservationDetails RD on T.TableID = RD.TableID
       INNER JOIN Reservations R on R.OrderID = RD.OrderID
       WHERE R.CompletionDate BETWEEN @start AND @end
       GROUP BY T.TableID
    )
    GO

### givenDiscountReport(startDate, endDate)

    CREATE FUNCTION givenDiscountsReport
    ( 
      @start date,
      @end date
    )
    RETURNS TABLE
    AS
    RETURN
    (
       WITH OrderValue as (
           SELECT O.OrderID as OD, SUM(Price) as Value FROM Orders as O
           INNER JOIN OrderDetails D on O.OrderID = D.OrderID
           GROUP BY O.OrderID
       )
    
       SELECT O.Discount, COUNT(*) as TotalOrders, SUM(Value * O.Discount) as TotalDiscount
       FROM Orders as O
       INNER JOIN OrderValue on OrderValue.OD = O.OrderID
       WHERE O.OrderDate BETWEEN @start AND @end
       GROUP BY O.Discount
    )
    GO

### menuReport(startDate, endDate)
**Report of orders for each dish in menu.**

    CREATE FUNCTION menuReport
    ( 
      @start date,
      @end date
    )
    RETURNS TABLE
    AS
    RETURN
    (
       SELECT OrderDetails.DishID, Name, SUM((1 - Discount) * Price) as Revenue, 
       COUNT(*) as TotalOrders 
       FROM OrderDetails
       INNER JOIN Orders O on O.OrderID = OrderDetails.OrderID
       INNER JOIN Dishes D on D.DishID = OrderDetails.DishID
       WHERE O.OrderDate BETWEEN @start AND @end
       GROUP BY OrderDetails.DishID, Name
    )
    GO

### individualReport(startDate, endDate)
**Report for individual customer for given period of time.**

    CREATE FUNCTION individualReport
    ( 
      @start date,
      @end date
    )
    RETURNS TABLE
    AS
    RETURN
    (
       WITH OrderValue as (
          SELECT O.OrderID as OD, SUM(Price) as Value FROM Orders as O
          INNER JOIN OrderDetails D on O.OrderID = D.OrderID
          GROUP BY O.OrderID
      )
    
       SELECT I.CustomerID, I.FirstName, I.LastName, OrderDate, Value 
    FROM Individual as I
       INNER JOIN Orders O on I.CustomerID = O.CustomerID
       INNER JOIN OrderValue on OrderValue.OD = O.OrderID
       WHERE OrderDate BETWEEN @start AND @end
       GROUP BY ROLLUP (I.CustomerID, I.FirstName, I.LastName, 
       (OrderID, OrderDate, Value))
    )
    GO

### companyReport(startDate, endDate)
**Report for company for a period of time.**

    CREATE FUNCTION companyReport
    ( 
      @start date,
      @end date
    )
    RETURNS TABLE
    AS
    RETURN
    (
       WITH OrderValue as (
          SELECT O.OrderID as OD, SUM(Price) as Value FROM Orders as O
          INNER JOIN OrderDetails D on O.OrderID = D.OrderID
          GROUP BY O.OrderID
      )
    
       SELECT C.CustomerID, C.CompanyName, OrderDate, Value FROM Company as C
       INNER JOIN Orders O on C.CustomerID = O.CustomerID
       INNER JOIN OrderValue on OrderValue.OD = O.OrderID
       WHERE OrderDate BETWEEN @start AND @end
       GROUP BY ROLLUP (C.CustomerID, C.CompanyName, (OrderID, OrderDate, Value))
    )
    GO

### updateNumConstant(name, newValue)

    CREATE PROCEDURE updateNumConstant
    (
       @Name Varchar(70),
       @newValue int
    )
    AS
    IF (EXISTS(SELECT * FROM NumConstants WHERE Name = @Name))
      BEGIN
          UPDATE NumConstants
          SET Value = @newValue
          WHERE Name = @Name
      END
    ELSE
      BEGIN
          INSERT INTO NumConstants
          VALUES (@Name, @newValue)
      end
    GO

### updateTimeConstant(name, newValue)

    CREATE PROCEDURE updateTimeConstant
    (
       @Name Varchar(70),
       @newValue time(7)
    )
    AS
    IF (EXISTS(SELECT * FROM TimeConstants WHERE Name = @Name))
      BEGIN
          UPDATE TimeConstants
          SET Value = @newValue
          WHERE Name = @Name
      END
    ELSE
      BEGIN
          INSERT INTO TimeConstants
          VALUES (@Name, @newValue)
      end
    GO