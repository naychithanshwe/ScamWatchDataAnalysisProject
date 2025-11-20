/******************************************************************************************
 Project  : ScamWatch Data Analysis Portfolio
 Author   : <Nay Chi Than Shwe>
 Date     : <29-08-2025>
******************************************************************************************/

------------------------------------------------------------------------------------------
-- Step 1: Combine all source tables (2020–2025) into one consolidated table
------------------------------------------------------------------------------------------

-- SELECT *
-- INTO ScamWatchCombined
-- FROM (
--     SELECT * FROM ScamWatchReport2020Q1Q2
--     UNION ALL SELECT * FROM ScamWatchReport2020Q3Q4
--     UNION ALL SELECT * FROM ScamWatchReport2021Q1Q2
--     UNION ALL SELECT * FROM ScamWatchReport2021Q3
--     UNION ALL SELECT * FROM ScamWatchReport2021Q4
--     UNION ALL SELECT * FROM ScamWatchReport2022Q1Q2
--     UNION ALL SELECT * FROM ScamWatchReport2022Q3
--     UNION ALL SELECT * FROM ScamWatchReport2022Q4
--     UNION ALL SELECT * FROM ScamWatchReport2023Q1Q2
--     UNION ALL SELECT * FROM ScamWatchReport2023Q3
--     UNION ALL SELECT * FROM ScamWatchReport2023Q4
--     UNION ALL SELECT * FROM ScamWatchReport2024Q1Q2
--     UNION ALL SELECT * FROM ScamWatchReport2024Q3Q4
--     UNION ALL SELECT * FROM ScamWatchReport2025
-- ) AS Combined;

------------------------------------------------------------------------------------------
-- Step 2: Clean the Amount_lost column
------------------------------------------------------------------------------------------

-- a) Add new column
-- ALTER TABLE ScamWatchCombined
-- ADD Amount_lost_clean FLOAT;

-- b) Remove '$' and commas, then cast to FLOAT
-- UPDATE ScamWatchCombined
-- SET Amount_lost_clean = TRY_CAST(REPLACE(REPLACE(Amount_lost, '$',''), ',','') AS FLOAT);

-- c) Drop the original column
-- ALTER TABLE ScamWatchCombined
-- DROP COLUMN Amount_lost;

-- d) Change data type from FLOAT to DECIMAL(18,2)
-- ALTER TABLE ScamWatchCombined
-- ALTER COLUMN Amount_lost_clean DECIMAL(18,2);

-- e) Rename Amount_lost_clean to Amount_lost
-- EXEC sp_rename 'ScamWatchCombined.Amount_lost_clean', 'Amount_lost', 'COLUMN';

------------------------------------------------------------------------------------------
-- Step 3: Standardize the reporting date column
------------------------------------------------------------------------------------------

-- a) Rename column StartOfMonth to Date
-- EXEC sp_rename 'dbo.ScamWatchCombined.StartOfMonth', 'Date', 'COLUMN';

-- b) Alter column data type to DATE
-- ALTER TABLE dbo.ScamWatchCombined
-- ALTER COLUMN Date DATE;

------------------------------------------------------------------------------------------
-- Step 4: Identify and assess duplicate records in the combined dataset
------------------------------------------------------------------------------------------

-- SELECT 
--     Date,
--     Address_State,
--     Scam_Contact_Mode,
--     Complainant_Age,
--     Complainant_Gender,
--     Category_Level_2,
--     Category_Level_3,
--     Number_of_reports,
--     COUNT(*) AS DuplicateCount
-- FROM 
--     ScamWatchCombined
-- GROUP BY 
--     Date,
--     Address_State,
--     Scam_Contact_Mode,
--     Complainant_Age,
--     Complainant_Gender,
--     Category_Level_2,
--     Category_Level_3,
--     Number_of_reports
-- HAVING 
--     COUNT(*) > 1;

------------------------------------------------------------------------------------------
-- Step 5: Extract unique scam category combinations for normalization
------------------------------------------------------------------------------------------

-- SELECT DISTINCT 
--     Category_Level_2, 
--     Category_Level_3
-- FROM 
--     ScamWatchCombined
-- ORDER BY 
--     Category_Level_3, 
--     Category_Level_2;

------------------------------------------------------------------------------------------
-- Step 6: Create dimension table for scam category classification
------------------------------------------------------------------------------------------

-- CREATE TABLE DimScamType (
--     ScamTypeID INT PRIMARY KEY IDENTITY(1,1),
--     Category_Level_2 NVARCHAR(100),
--     Category_Level_3 NVARCHAR(100)
-- );

-- INSERT INTO DimScamType (Category_Level_2, Category_Level_3)
-- SELECT DISTINCT 
--     Category_Level_2, 
--     Category_Level_3
-- FROM 
--     ScamWatchCombined
-- WHERE 
--     Category_Level_2 IS NOT NULL AND 
--     Category_Level_3 IS NOT NULL;

------------------------------------------------------------------------------------------
-- Step 7: Create normalized fact table with ScamTypeID reference
------------------------------------------------------------------------------------------

-- SELECT *
-- INTO FactScamReports
-- FROM ScamWatchCombined;

-- ALTER TABLE FactScamReports
-- ADD ScamTypeID INT;

-- UPDATE FactScamReports
-- SET FactScamReports.ScamTypeID = DST.ScamTypeID
-- FROM FactScamReports FSR
-- JOIN DimScamType DST
-- ON FSR.Category_Level_2 = DST.Category_Level_2
-- AND FSR.Category_Level_3 = DST.Category_Level_3;

-- ALTER TABLE FactScamReports
-- DROP COLUMN Category_Level_2, Category_Level_3;

------------------------------------------------------------------------------------------
-- Step 8: Create normalized dimension for Age Group and link to fact table
------------------------------------------------------------------------------------------

-- a) CREATE TABLE DimAgeGroup (
--     AgeGroupID INT PRIMARY KEY,
--     AgeGroup NVARCHAR(100)
-- );

-- b) SET IDENTITY_INSERT DimAgeGroup ON;

-- INSERT INTO DimAgeGroup (AgeGroupID, AgeGroup)
-- VALUES
-- (0, 'Unspecified'),
-- (1, 'Under 18'),
-- (2, '18 - 24'),
-- (3, '25 - 34'),
-- (4, '35 - 44'),
-- (5, '45 - 54'),
-- (6, '55 - 64'),
-- (7, '65 and over');

-- SET IDENTITY_INSERT DimAgeGroup OFF;

-- c) ALTER TABLE FactScamReports
-- ADD AgeGroupID INT;

-- d) UPDATE FactScamReports
-- SET FactScamReports.AgeGroupID = DAG.AgeGroupID
-- FROM FactScamReports FSR
-- JOIN DimAgeGroup DAG
-- ON FSR.Complainant_Age = DAG.AgeGroup;

-- e) ALTER TABLE FactScamReports
-- DROP COLUMN Complainant_Age;

------------------------------------------------------------------------------------------
-- Step 9: Create normalized dimension for Region and link to fact table
------------------------------------------------------------------------------------------

-- a) CREATE TABLE DimRegion (
--     RegionID INT PRIMARY KEY,
--     Address_State NVARCHAR(100)
-- );

-- b) INSERT INTO DimRegion (RegionID, Address_State)
-- VALUES
-- (0, 'Unspecified'),
-- (1, 'Australian Capital Territory'),
-- (2, 'New South Wales'),
-- (3, 'Northern Territory'),
-- (4, 'Queensland'),
-- (5, 'South Australia'),
-- (6, 'Tasmania'),
-- (7, 'Victoria'),
-- (8, 'Western Australia'),
-- (9, 'Outside of Australia');

-- c) Add RegionID to FactScamReports
--ALTER TABLE FactScamReports
--ADD RegionID INT;

---- d) Populate RegionID using JOIN with DimRegion
--UPDATE FactScamReports
--SET FactScamReports.RegionID = DR.RegionID
--FROM FactScamReports FSR
--JOIN DimRegion DR 
--ON DR.Address_State = FSR.Address_State;

------------------------------------------------------------------------------------------
-- Step 10: Create normalized dimension for Gender and link to fact table
------------------------------------------------------------------------------------------

-- a) CREATE TABLE DimGender (
--     GenderID INT PRIMARY KEY,
--     Complainant_Gender NVARCHAR(50)
-- );

-- b) INSERT INTO DimGender (GenderID, Complainant_Gender)
-- VALUES
-- (0, 'X (Indeterminate/Intersex/Unspecified)'),
-- (1, 'Male'),
-- (2, 'Female');

---- c) Add GenderID to FactScamReports
--ALTER TABLE FactScamReports
--ADD GenderID INT;

---- d) Populate GenderID using JOIN with DimGender
--UPDATE FactScamReports
--SET FactScamReports.GenderID = DG.GenderID
--FROM FactScamReports FSR
--JOIN DimGender DG
--ON DG.Complainant_Gender = FSR.Complainant_Gender;

------------------------------------------------------------------------------------------
-- Step 11: Create normalized dimension for Scam Contact Mode and link to fact table
------------------------------------------------------------------------------------------

---- a) Create DimContactMode with ContactModeID as primary key
--CREATE TABLE DimContactMode (
--    ContactModeID INT PRIMARY KEY,
--    Scam_Contact_Mode NVARCHAR(100)
--);

---- b) Insert standardized contact modes with logical IDs
--INSERT INTO DimContactMode (ContactModeID, Scam_Contact_Mode)
--VALUES
--(0, 'unspecified'),
--(1, 'Email'),
--(2, 'Fax'),
--(3, 'In person'),
--(4, 'Internet'),
--(5, 'Mail'),
--(6, 'Mobile apps'),
--(7, 'Phone call'),
--(8, 'Social media/Online forums'),
--(9, 'Text message');

---- c) Add ContactModeID to FactScamReports
--ALTER TABLE FactScamReports
--ADD ContactModeID INT;

---- d) Populate ContactModeID using JOIN with DimContactMode
--UPDATE FactScamReports
--SET FactScamReports.ContactModeID = DCM.ContactModeID
--FROM FactScamReports FSR
--JOIN DimContactMode DCM
--ON FSR.Scam_Contact_Mode = DCM.Scam_Contact_Mode;

---- e) Drop original columns
--ALTER TABLE FactScamReports
--DROP COLUMN Scam_Contact_Mode, Address_State, Complainant_Age, Complainant_Gender, Category_Level_2, Category_Level_3

------------------------------------------------------------------------------------------
-- Final Check: View normalized Fact Table
------------------------------------------------------------------------------------------

SELECT *
FROM FactScamReports;
