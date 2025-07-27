-- Step 1: Create the target table
IF OBJECT_ID('DimDate', 'U') IS NOT NULL
    DROP TABLE DimDate;

CREATE TABLE DimDate (
    SKDate INT PRIMARY KEY,
    KeyDate DATE,
    [Date] DATE,
    CalendarDay INT,
    CalendarMonth INT,
    CalendarQuarter INT,
    CalendarYear INT,
    DayNameLong VARCHAR(20),
    DayNameShort VARCHAR(10),
    DayNumberOfWeek INT,
    DayNumberOfYear INT,
    DaySuffix VARCHAR(5),
    FiscalWeek INT,
    FiscalPeriod INT,
    FiscalQuarter INT,
    FiscalYear INT,
    [Fiscal Year/Period] VARCHAR(20)
);

-- Step 2: Create the stored procedure
IF OBJECT_ID('PopulateDimDateForYear', 'P') IS NOT NULL
    DROP PROCEDURE PopulateDimDateForYear;

GO
CREATE PROCEDURE PopulateDimDateForYear
    @InputDate DATE
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @StartDate DATE = DATEFROMPARTS(YEAR(@InputDate), 1, 1);
    DECLARE @EndDate DATE = DATEFROMPARTS(YEAR(@InputDate), 12, 31);

    ;WITH DateSeries AS (
        SELECT @StartDate AS CurrentDate
        UNION ALL
        SELECT DATEADD(DAY, 1, CurrentDate)
        FROM DateSeries
        WHERE CurrentDate < @EndDate
    )
    INSERT INTO DimDate (
        SKDate, KeyDate, [Date], CalendarDay, CalendarMonth, CalendarQuarter, CalendarYear,
        DayNameLong, DayNameShort, DayNumberOfWeek, DayNumberOfYear, DaySuffix,
        FiscalWeek, FiscalPeriod, FiscalQuarter, FiscalYear, [Fiscal Year/Period]
    )
    SELECT
        CAST(CONVERT(CHAR(8), CurrentDate, 112) AS INT) AS SKDate,
        CurrentDate AS KeyDate,
        CurrentDate AS [Date],
        DAY(CurrentDate) AS CalendarDay,
        MONTH(CurrentDate) AS CalendarMonth,
        DATEPART(QUARTER, CurrentDate) AS CalendarQuarter,
        YEAR(CurrentDate) AS CalendarYear,
        DATENAME(WEEKDAY, CurrentDate) AS DayNameLong,
        LEFT(DATENAME(WEEKDAY, CurrentDate), 3) AS DayNameShort,
        DATEPART(WEEKDAY, CurrentDate) AS DayNumberOfWeek,
        DATEPART(DAYOFYEAR, CurrentDate) AS DayNumberOfYear,
        CAST(DAY(CurrentDate) AS VARCHAR) + 
            CASE 
                WHEN DAY(CurrentDate) IN (11, 12, 13) THEN 'th'
                WHEN DAY(CurrentDate) % 10 = 1 THEN 'st'
                WHEN DAY(CurrentDate) % 10 = 2 THEN 'nd'
                WHEN DAY(CurrentDate) % 10 = 3 THEN 'rd'
                ELSE 'th'
            END AS DaySuffix,
        DATEPART(WEEK, CurrentDate) AS FiscalWeek,
        MONTH(CurrentDate) AS FiscalPeriod,
        DATEPART(QUARTER, CurrentDate) AS FiscalQuarter,
        YEAR(CurrentDate) AS FiscalYear,
        CAST(YEAR(CurrentDate) AS VARCHAR) + RIGHT('0' + CAST(MONTH(CurrentDate) AS VARCHAR), 2)
            AS [Fiscal Year/Period]
    FROM DateSeries
    OPTION (MAXRECURSION 366);
END;
GO

-- Step 3: Example Execution
-- EXEC PopulateDimDateForYear '2020-07-14';
