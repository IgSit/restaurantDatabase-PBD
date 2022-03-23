# Views
## UnconfirmedReservations 
**Reservations waiting to be confirmed.**

    CREATE VIEW dbo.UnconfirmedReservations
    AS
    SELECT dbo.Reservations.ReservationID, dbo.Reservations.CompletionDate, 
           dbo.Reservations.OrderID, dbo.OrderDetails.DishID, dbo.OrderDetails.Quantity
    FROM dbo.Reservations 
    INNER JOIN dbo.OrderDetails ON dbo.Reservations.OrderID = dbo.OrderDetails.OrderID
    WHERE (dbo.Reservations.EmployeeID IS NULL)
    go
## TodaysOrders
**Orders made for current day**

    CREATE VIEW dbo.TodaysOrders
    AS
    SELECT dbo.Orders.OrderID, dbo.Orders.Description, dbo.Orders.Discount, 
           dbo.OrderDetails.Quantity, dbo.Dishes.Name, dbo.ReservationDetails.TableID, 
           dbo.Reservations.CompletionDate
    FROM dbo.Dishes 
    INNER JOIN dbo.OrderDetails ON dbo.Dishes.DishID = dbo.OrderDetails.DishID 
    INNER JOIN dbo.Orders ON dbo.OrderDetails.OrderID = dbo.Orders.OrderID 
    INNER JOIN dbo.Reservations ON dbo.Orders.OrderID = dbo.Reservations.OrderID 
    INNER JOIN dbo.ReservationDetails ON dbo.Reservations.OrderID = dbo.ReservationDetails.OrderID
    WHERE (CAST(dbo.Reservations.CompletionDate AS date) = CAST({ fn NOW() } AS date))
    go
## ClosestWeekendSeafood
**Every seafood dish ordered on closest weekend with quantities of every dish.**

    CREATE VIEW dbo.ClosestWeekendSeafood
    AS
    SELECT dbo.Dishes.Name, SUM(dbo.OrderDetails.Quantity) AS 'Quantity'
    FROM dbo.Categories 
    INNER JOIN dbo.Dishes ON dbo.Categories.CategoryID = dbo.Dishes.CategoryID 
    INNER JOIN dbo.OrderDetails ON dbo.Dishes.DishID = dbo.OrderDetails.DishID 
    INNER JOIN dbo.Orders ON dbo.OrderDetails.OrderID = dbo.Orders.OrderID 
    INNER JOIN dbo.Reservations ON dbo.Orders.OrderID = dbo.Reservations.OrderID
    WHERE (DATEDIFF(day, dbo.Reservations.CompletionDate, { fn NOW() }) < 7) 
          AND (dbo.Categories.CategoryName = 'Seafood') AND 
          (DATEPART(weekday, dbo.Reservations.CompletionDate) = 6 
          OR DATEPART(weekday, dbo.Reservations.CompletionDate) = 7)
    GROUP BY dbo.Dishes.Name
    go