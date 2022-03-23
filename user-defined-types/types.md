#User-defined types
## DishInfo

    CREATE TYPE DishInfo as Table(dishID int, quantity int, price decimal(18, 2)) GO

## TableList

    CREATE TYPE TableList AS TABLE (TableID int) GO

## MenuList

    CREATE TYPE MenuList AS TABLE (DishID int, Price decimal(18, 2)) GO
