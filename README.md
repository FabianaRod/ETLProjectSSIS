# ETL Project with Visual Studio and SSIS

This repository reflects my (challenching 🦾) **ETL** (Extract, Transform, Load) project using Visual Studio, **SSIS** (SQL Server Integration Services), and **T-SQL**. 

The project encompasses tasks such as creating data flow tasks for slow-changing dimensions, implementing OLE DB sources and destinations, and incorporating various transformations. 
Also, features include handling multicasts, joins, conditional splits, and adhering to best practices for Slowly Changing Dimensions (SCD). 🤓

It is worth mentioning that I took advantage of Microsoft's **public data** to develop this SSIS project using 2 datasets:

- **📊 AdventureWorksDW2022**:

It is a data warehouse database that provides a dimensional model for business analysis and reporting. Focusing on business scenarios related to sales, marketing, inventory and finance.

- **📊 WideWorldImporters**:

Sample database that simulates a business management system for an import and distribution company and addresses broader use cases, including sales, purchasing, inventory, finance and business operations.

This has certainly been the most **challenging** project I have created using Visual Studio for ETL processes and here's why:

### Import and Export Task ✔️

Here's the overview for this first pack:

![image](https://github.com/FabianaRod/ETLProjectSSIS/assets/155020943/acfb0e6c-455e-4b50-9c8e-27186235612c)


#### 1. Basic Steps

First, I created a WideWorldImporters connection in ADO.NET and set the WWI_MartDemo table in SQL MS as the destination.

Then, created an ADO.NET source to extract the stock items and colors tables from the Warehouse structure, with an OLE DB destination to **populate** the "Stock Items" table in WWI_MartDemo **(1)**. 
Noticing repetitions, added a SQL command to remove duplicates **(2)**. 

And then, set up a flat file connection for territory data, and using a script component in Visual Basic created the "region" column to define each one.

```sql
 Select Case CType(Row.Columna2, Integer)
     Case 1
         Row.Region = "UK"
     Case 2
         Row.Region = "Africa"
     Case 3
         Row.Region = "America"
     Case 4
         Row.Region = "Asia"
     Case 5
         Row.Region = "Europa"
     Case 6
         Row.Region = "Oceania"

 End Select
```
 
Finally, created an OLEDB destination and "Territory" table to populate it, **adjusting** source data including the [Zipcode], [Territory], and [Region] columns **(3)**.


![(1)](https://github.com/FabianaRod/ETLProjectSSIS/assets/155020943/040b1a26-d997-42cb-a9a0-6f65a2c69ff7)


#### 2. Variable and Container Creation

Created two variables, **varFileName** and **varFileNationality**, and established two flat file connections: "National Clients" and "International Clients" using the supplied text files **(4)**.

Then, set up a **foreach loop container** called "Massive Clients Upload" and configured it as a foreach file enumerator. 

Inside the container, added a script task configured in Visual Basic to extract the last 3 digits starting from the 7th character of the file, assigning them to the varFileNationality variable.

```sql
Dim vars As Variables
Dim FileNationality As String
Dim varsNationality As Variables
Dts.VariableDispenser.LockOneForRead("User::varFileName", vars)
FileNationality =
    vars("varFileName").Value.ToString.Substring(vars("varFileName").Value.ToString.
    Length - 7, 3)
Dts.VariableDispenser.LockOneForWrite()("varFileNationality", varsNationality)
varsNationality("varFileNationality").Value = FileNationality
```
       
I also created two Clients-scope dependent data flow tasks, one for **Nac** and one for **Int**, with edited connections for each variable **(5)**. 

Lastly, configured both tasks with a flat file source and an OLE DB destination, creating the Clients Table, and joined to the preceding task to delete data.


![image](https://github.com/FabianaRod/ETLProjectSSIS/assets/155020943/fb893091-3391-4752-991e-7bb21be6fb35)

#### 3. Data Conversion

I added a data flow task called "Customers Load (Dimension)" and created a new OLE DB connection to WideworldImporters **(6)**.

Also established a flat file connection to the provided "Customers" table called "Customer file" **(7)**.

In the data flow task, set up a "Read Customers" flat file source with all default columns and **transformed** the data to a **NON-UNICODE format** for the destination file **(8)**.


![image](https://github.com/FabianaRod/ETLProjectSSIS/assets/155020943/6445fa45-1afd-4231-bd0b-2a59f9b03727)

#### 4. Lookups

Created a lookup called "Lookup CustomerCategory" using the [Sales][CustomerCategories] table, setting up the columns and relationships with the keys Customercategoryid, Buyinggroupid, and Primarycontactpersonid. 
Also created an OLE DB destination in WWI_MartDemo with the corresponding table, **checking the supplied mappings**:

```sql
CREATE TABLE [CustomersDimension] (
    [CustomerID Conversion] int,
    [CustomerName Conversion] nvarchar(50),
    [CustomerCategoryName] nvarchar(50),
    [BuyingGroupName] nvarchar(50),
    [PrimaryContactName] nvarchar(50),
    [PostalPostalCode Conversion] nvarchar(50),
    [ValidFrom Conversion] datetime,
    [ValidTo Conversion] nvarchar(50)
)
```

And (again) deleted the CustomerDimension table in the preceding flow task **(9)**.




![image](https://github.com/FabianaRod/ETLProjectSSIS/assets/155020943/3ef5e30b-d950-4e47-b576-4e6b85183ac2)

#### 5. Derived Column

Set up a dataflow task called "StateProvince Load" with an ADO.NET source using an SQL command to select specific data from the "StateProvinces(Application)" table **(10)**. 

In that task, I used a derived column tool to create new columns and concatenate values. 

Then, set up an OLE DB destination for the [DimStateProvince] table, and deleted the data from this table in the preceding task **(11)**.

``` sql
CREATE TABLE [DimStateProvince] (
    [StateProvinceID] int,
    [StateProvinceCode] nvarchar(5),
    [StateProvinceName] nvarchar(50),
    [SalesTerritory] nvarchar(50),
    [StateProvince FullName] varchar(50),
    [Updated Date] datetime
```

![image](https://github.com/FabianaRod/ETLProjectSSIS/assets/155020943/07b36b1f-51a6-432f-96b2-62573994ca8a)

#### 6. Aggregate and Sort

Created the "Group Sales by Geography" data flow task. I set up a connection from the "Sales by Geography" table in flat file format with specific data types **(12)**.

Next, created a flat file source and set up aggregations for cities, provinces, countries and continents, each with their respective sums, averages, minimums and maximums for quantities, tax rates and amounts, profits and totals.

Then added **four sorting tools**, sorting each **aggregation** by totals excluding and including taxes. 

Finally, I set up a flat file destination for each city, province, country and continent, generating files named *OrderedCities, OrderedProvinces, OrderedCountries and OrderedContinents*, respectively **(13)**.


![image](https://github.com/FabianaRod/ETLProjectSSIS/assets/155020943/659531ce-c840-495b-964c-c87bfa5ca3b8)


### Demo Join Conditional Split Task ✔️

Here you will see the steps to build this second pack.

![image](https://github.com/FabianaRod/ETLProjectSSIS/assets/155020943/5da3da98-d959-484a-b13e-227487b06c12)


#### 7. Joins and Condition Splits

Two SQL MS tables, "CityStaging" and "CityDimension", were created in the WWI_MartDemo database. Then, a new **SSIS pack** called "DemoJoinConditionalSplit" was also created (inheriting previous OLEDB and ADO.NET connections).

```  sql
USE [WWI_MartDemo]
GO
CREATE TABLE [CityStaging](
[City Staging Key] [int] IDENTITY(1,1) NOT NULL,
[WWI City ID] [int] NOT NULL,
[City] [nvarchar](50) NOT NULL,
[State Province] [nvarchar](50) NOT NULL,
[Country] [nvarchar](60) NOT NULL,
[Continent] [nvarchar](30) NOT NULL,
[Sales Territory] [nvarchar](50) NOT NULL,
[Region] [nvarchar](30) NOT NULL,
[Subregion] [nvarchar](30) NOT NULL,
[Latest Recorded Population] [bigint] NULL,
[Valid From] [datetime2](7) NOT NULL,
[Valid To] [datetime2](7) NOT NULL,
CONSTRAINT [PK_Integration_City_Staging] PRIMARY KEY CLUSTERED
 (
[City Staging Key] ASC
 )
 )
 ```

 ``` sql
USE [WWI_MartDemo]
GO
CREATE TABLE CityDimension(
[City Key] [int] NOT NULL,
[WWI City ID] [int] NOT NULL,
[City] [nvarchar](50) NOT NULL,
[State Province] [nvarchar](50) NOT NULL,
[Country] [nvarchar](60) NOT NULL,
[Continent] [nvarchar](30) NOT NULL,
[Sales Territory] [nvarchar](50) NOT NULL,
[Region] [nvarchar](30) NOT NULL,
[Subregion] [nvarchar](30) NOT NULL,
[Latest Recorded Population] [bigint] NULL,
[Valid From] [datetime2](7) NOT NULL,
[Valid To] [datetime2](7) NOT NULL,
CONSTRAINT [PK_Dimension_City] PRIMARY KEY CLUSTERED
(
[City Key] ASC
 )
)
```

In the package, *flat file and neon connections* were set up to read "Countries" file and "Cities" file, respectively. 

A stream container was implemented with two data flow tasks: Load city staging and Load dim city **(14)**.

For "Load City Staging" task, I performed **sort and merge operations** to populate the City Staging table in the WWI_MartDemo database. Type conversions and mappings were also performed on data coming from different sources. **(15)**

In "Load Dim City", data from the City Staging and City Dimension tables were **merged**, and a **conditional split** condition called "New Register" was applied **(16)**.
Finally, i included a data deletion task in the CityStaging and CityDimension tables in the WWI_MartDemo database.

![(1) (1)](https://github.com/FabianaRod/ETLProjectSSIS/assets/155020943/fea3afc5-22b3-477e-8f42-94ac3c274c14)


### Multicast Demo Task ✔️

Here I will show you the construction of the third and very short package.

![image](https://github.com/FabianaRod/ETLProjectSSIS/assets/155020943/1058f37b-51ef-420d-95aa-089124737b5a)


#### 8. Multicast

I created a **new SSIS** package called MulticastDemo. In addition and set up a new connection in the connection manager called WWI_Datawarehouse. 

In the Supplier Purchases data flow task **(17)**, an OLE DB source was set up from WWI_Datawarehouse, selecting the Suppliers dimensions and Purchases purchases with a JOIN query.

 ``` sql
SELECT Dimension.Supplier.Supplier, Dimension.Supplier.Category, Dimension.Supplier.[Payment Days], Fact.Purchase.[Ordered Outers], Fact.Purchase.[Ordered Quantity], Fact.Purchase.[Received Outers]
FROM     Dimension.Supplier INNER JOIN
                  Fact.Purchase ON Dimension.Supplier.[Supplier Key] = Fact.Purchase.[Supplier Key]
 ```

Within the task, I added a Multicast component to handle multiple destinations and created two destinations: an Excel file destination called PurchasesExcel and a flat file destination called PurchasesTxt **(18)**. 

Finally, the data was loaded into both files.

![image](https://github.com/FabianaRod/ETLProjectSSIS/assets/155020943/e4061d9c-a51f-49eb-b73a-e659f32434c4)


### Slowly Changing Dimension Task ✔️

![image](https://github.com/FabianaRod/ETLProjectSSIS/assets/155020943/d788c1cd-7178-4c42-86e6-98577b3c6238)

#### 9. SCD

I created the last SSIS package called "SlowChangeDim Demo" and established an OLE DB connection called "AdventureWorksDWW2022", then created a table in SQL named Employee_Source. 

``` sql
USE [WideWorldImporters]
GO
CREATE TABLE [dbo].[Empleado_Source](
       [ParentEmployeeKey] [int] NULL,
       [EmployeeNationalIDAlternateKey] [nvarchar](15) NULL,
       [ParentEmployeeNationalIDAlternateKey] [nvarchar](15) NULL,
       [SalesTerritoryKey] [int] NULL,
       [FirstName] [nvarchar](50) NULL,
       [LastName] [nvarchar](50) NULL,
       [MiddleName] [nvarchar](50) NULL,
       [NameStyle] [bit] NULL,
       [Title] [nvarchar](50) NULL,
       [HireDate] [date] NULL,
       [BirthDate] [date] NULL,
       [LoginID] [nvarchar](256) NULL,
       [EmailAddress] [nvarchar](50) NULL,
       [Phone] [nvarchar](25) NULL,
       [MaritalStatus] [nvarchar](1) NULL,
       [EmergencyContactName] [nvarchar](50) NULL,
       [EmergencyContactPhone] [nvarchar](25) NULL,
       [SalariedFlag] [bit] NULL,
       [Gender] [nvarchar](1) NULL,
       [PayFrequency] [tinyint] NULL,
       [BaseRate] [money] NULL,
       [VacationHours] [smallint] NULL,
       [SickLeaveHours] [smallint] NULL,
       [CurrentFlag] [bit] NULL,
       [SalesPersonFlag] [bit] NULL,
       [DepartmentName] [nvarchar](50) NULL,
             [StartDate] [datetime] NULL,
            [EndDate] [datetime] NULL,
       [Status] [nvarchar](50) NULL
) ON [PRIMARY]
GO
```

Next, inserted two rows of data into this table.

In Visual Studio, designed a data flow task called "Load Employee Dimension" that included an OLE DB source to read data from the AdventureWorksDW2022 and an OLE DB destination to load data into the EmployeeDimension table in the WWI_MartDemo **(19)**.

Next, implemented a new data flow task called Slow Changing Employee Dim. In it, I set up an OLE DB source to read data from the Employee_Source table and added an SCD task connected to the EmployeeDimension table in the WWI_MartDemo.

I defined the EmployeeNationalIDAlternateKey as business key, and selected BirthDate and LoginID as fixed attributes, plus DepartmentName as historical attribute. 
Set the option to fail transformation in case of change in fixed attributes and used startdate/enddate to identify records, with the System::CreationDate variable for date data **(20)**. 

Lastly, disabled the option to infer changes and executed the flow to populate the table with new records. 

![image](https://github.com/FabianaRod/ETLProjectSSIS/assets/155020943/4d5a39ed-dea1-4034-a63f-813798400dc0)


And that ends the ETL process and one more achievement for my portfolio.

Thanks for reading 💙.

