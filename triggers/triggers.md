# Triggers
### onCloseTable
**Makes all reservation on this table unconfirmed and deletes corresponding reservation details.**

    CREATE TRIGGER check_reservations_after_table_deactivation
      ON Tables
      AFTER UPDATE
      AS
      BEGIN
          DECLARE
          @TableID int,
          @InactiveDueDate date
          SET NOCOUNT ON
    
          SET @TableID = (SELECT TableID FROM inserted)
          SET @InactiveDueDate = (SELECT InactiveDueDate FROM inserted)
    
          IF @InactiveDueDate IS NOT NULL
              BEGIN
                  -- Mark reservations using this table as unconfirmed and 
                  -- add special description
                  UPDATE Reservations
                  SET EmployeeID = null
                  WHERE OrderID in (
                      SELECT R.OrderID as Oid FROM Reservations R
                      INNER JOIN ReservationDetails RD on R.OrderID = RD.OrderID
                      WHERE RD.TableID = @TableID
                      AND (R.CompletionDate BETWEEN GETDATE() AND @InactiveDueDate)
                  )
    
                  UPDATE Orders
                  SET Description = CONCAT(Description, 
                  CONCAT('Requires looking over tables after ', 
                  CONCAT(@TableID, ' was deactivated.')))
                  WHERE OrderID in (
                      SELECT R.OrderID as Oid FROM Reservations R
                      INNER JOIN ReservationDetails RD on R.OrderID = RD.OrderID
                      WHERE RD.TableID = @TableID
                      AND (R.CompletionDate BETWEEN GETDATE() AND @InactiveDueDate)
                  )
    
                  -- Delete matching rows from ReservationDetails
                  DELETE FROM ReservationDetails
                  WHERE OrderID in (
                      SELECT R.OrderID as Oid FROM Reservations R
                      INNER JOIN ReservationDetails RD on R.OrderID = RD.OrderID
                      WHERE RD.TableID = @TableID
                      AND (R.CompletionDate BETWEEN GETDATE() AND @InactiveDueDate)
                  )
              END
      END
    GO

### onAddOrder
**Increments number of orders and calculates whether the customer should be given
temporary discount for the next order.**

    CREATE TRIGGER increase_previous_orders
    ON Orders AFTER INSERT
    AS
    BEGIN
      DECLARE
      @CustomerID int
      SET NOCOUNT ON;
    
      SET @CustomerID = (SELECT CustomerID FROM inserted)
    
      IF @CustomerID IN (SELECT CustomerID FROM Individual)
       BEGIN
           UPDATE Individual
           SET PreviousOrders = PreviousOrders + 1
           IF (SELECT PreviousOrders FROM Individual WHERE CustomerID = @CustomerID) >=
              (SELECT Value FROM NumConstants WHERE Name = 'Z1')
               BEGIN
                   UPDATE Individual
                   SET PernamentDiscount = 1
                   WHERE CustomerID = @CustomerID
               END
       END
    END
    GO

### onDeleteOrder
**Negation of onAddOrder effect.**

    CREATE TRIGGER decrease_previous_orders
    ON Orders AFTER DELETE
    AS
    BEGIN
      DECLARE
      @CustomerID int
      SET NOCOUNT ON;
    
      SET @CustomerID = (SELECT CustomerID FROM deleted)
    
      IF @CustomerID IN (SELECT CustomerID FROM Individual)
       BEGIN
           UPDATE Individual
           SET PreviousOrders = PreviousOrders - 1
           IF (SELECT PreviousOrders FROM Individual WHERE CustomerID = @CustomerID) <
              (SELECT Value FROM NumConstants WHERE Name = 'Z1')
               BEGIN
                   UPDATE Individual
                   SET PernamentDiscount = 0
                   WHERE CustomerID = @CustomerID
               END
       END
    END
    GO