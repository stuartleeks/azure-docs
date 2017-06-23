---
title: U-SQL programmability guide for Azure Data Lake | Microsoft Docs
description: Learn about the set of services in Azure Data Lake that enable you to create a cloud-based big data platform.
services: data-lake-analytics
documentationcenter: ''
author: MikeRys
manager: arindamc


ms.assetid: 63be271e-7c44-4d19-9897-c2913ee9599d
ms.service: data-lake-analytics
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: na
ms.workload: big-data
ms.date: 11/15/2016
ms.author: mrys

---

# U-SQL programmability guide

## Azure Data Lake
Azure Data Lake includes capabilities that make it easy for developers, data scientists, and analysts to store data of any size, shape, and speed. It also enables developers to use many types of processing and analytics across platforms and languages. It removes the complexities of ingesting and storing all your data while making it faster to use batches, streaming, and interactive analytics.

U-SQL is a query language that's designed for big data-type of workloads. One of the unique features of U-SQL is the combination of the SQL-like declarative language with the extensibility and programmability that's provided by C#. It also provides the ability to access and manipulate schema metadata, and to create custom components such as data processors and reducers.

In this guide, we concentrate on the extensibility and programmability of the U-SQL language that's enabled by C#.

## Requirements

Download and install [Azure Data Lake Tools for Visual Studio](https://www.microsoft.com/download/details.aspx?id=49504).

## Get started with U-SQL  
The description of U-SQL language is outside the scope of this document. However, we describe basic U-SQL constructs to gradually introduce U-SQL programmability features. For more information, see the [U-SQL Language Reference](http://aka.ms/usql_reference) guide.

Let’s look at the following example:

```sql
DECLARE @input_file string = @"\usql-programmability\input_file.tsv";
DECLARE @output_file string = @"\usql-programmability\output_file.tsv";

@rs0 =
	EXTRACT
        guid Guid,
	    dt DateTime,
        user String,
        des String
	FROM @input_file USING Extractors.Tsv();

@rs1 =
    SELECT
        MAX(guid) AS start_id,
	    MIN(dt) AS start_time,
        user,
        des
	FROM @rs0
    GROUP BY user, des;

OUTPUT @rs1 TO @output_file USING Outputters.Text();
```

In this example, we have an **Input File**: file input_file.tsv, defined by **Local Variable** @input_file.

The following actions are performed by the U-SQL script show in this example:

* The initial **EXTRACT** statement loads data to memory by converting Input File to the **Memory RowSet**.
* **SELECT** operates on data rowset to aggregate data and prepare for exporting.
* **OUTPUT** exports data rowset to **Output File**, which is an external file.

First, let’s look at some options for using a C# expression directly in a U-SQL Script.

## C# types and expressions in U-SQL script
A U-SQL C# expression, similar to an expression in general C#, is a sequence of one or more operators that can be evaluated to a single value, object, method, or namespace. Expressions can consist of a literal value, a method invocation, an operator, or a simple name. Simple names can be the name of a variable, type member, method parameter, namespace, or type.

When we talk about U-SQL C# expressions, we specifically refer to the U-SQL base script C# expressions. The underlying C# code-behind section, discussed later in this document, can also contain C# expressions as regular C# code-based elements.

Expressions can use operators that use other expressions as parameters. They can also use method calls with parameters that are other method calls. Following are examples of an expression:  

```
Convert.ToDateTime(Convert.ToDateTime(dt).ToString("yyyy-MM-dd"))
```

U-SQL language enables the usage of standard C# expressions from built-in namespaces.  

```
Microsoft.Analytics.Interfaces;  
Microsoft.Analytics.Types.Sql;  
System;  
System.Collections.Generic;  
System.Linq;  
System.Text;  
```

General C# expressions can be used in U-SQL SELECT, EXTRACT.

The C# expressions can also be used in DECLARE or IF statements. An example of this type of expression is as follows:   

```
DateTime.Today.Day   
Convert.ToDateTime
```

U-SQL-based script example:  

```
DECLARE @default_dt DateTime = Convert.ToDateTime("06/01/2016");
```

C# expressions can provide extended functionality when you're manipulating columns as part of a rowset. For example, if you want to convert a datetime column to the date with zero hours, you can use the following SELECT part of a U-SQL-based script:

```
@rs1 =
    SELECT
        MAX(guid) AS start_id,
	 MIN(dt) AS start_time,
        MIN(Convert.ToDateTime(Convert.ToDateTime(dt).ToString("yyyy-MM-dd")))
AS start_zero_time,
        user,
        des
	FROM @rs0
    GROUP BY user, des;
```

As you can see, we use `System.Convert.ToDateTime` method to run through the conversion.

Following is a slightly more complicated scenario that demonstrates the use of some basic C# operators:

```
@rs1 =
    SELECT
        MAX(guid) AS start_id,
	    MIN(dt) AS start_time,
          MIN(Convert.ToDateTime(Convert.ToDateTime(dt<@default_dt?@default_dt:dt).ToString("yyyy-MM-dd"))) AS start_zero_time,
        user,
        des
	FROM @rs0
    GROUP BY user, des;
```

This shows an example of a C# conditional operator expression.

The previous examples demonstrate the use of C# expressions in the base U-SQL script. However, U-SQL enables more extensible programmability features that are covered later in this document.  

Full script:

```
DECLARE @input_file string = @"\usql-programmability\input_file.tsv";
DECLARE @output_file string = @"\usql-programmability\output_file.tsv";

@rs0 =
	EXTRACT
        guid Guid,
	    dt DateTime,
        user String,
        des String
	FROM @input_file USING Extractors.Tsv();

DECLARE @default_dt DateTime = Convert.ToDateTime("06/01/2016");

@rs1 =
    SELECT
        MAX(guid) AS start_id,
	MIN(dt) AS start_time,
        MIN(Convert.ToDateTime(Convert.ToDateTime(dt<@default_dt?@default_dt:dt).ToString("yyyy-MM-dd"))) AS start_zero_time,
        user,
        des
	FROM @rs0
    GROUP BY user, des;

OUTPUT @rs1 TO @output_file USING Outputters.Text();
```

### Use C# expressions for data type conversions
The following example demonstrates how you can do a datetime data conversion by using C# expressions. In this particular scenario, string datetime data is converted to standard datetime with midnight 00:00:00 time notation.

```
DECLARE @dt String = "2016-07-06 10:23:15";
@rs1 =
    SELECT Convert.ToDateTime(Convert.ToDateTime(@dt).ToString("yyyy-MM-dd")) AS dt,
           dt AS olddt
    FROM @rs0;
OUTPUT @rs1 TO @output_file USING Outputters.Text();
```

### Use C# expressions for today’s date
To pull today’s date, we can use the following C# expression:

```
DateTime.Now.ToString("M/d/yyyy")
```

Here's an example of how to use this expression in a script:

```
@rs1 =
    SELECT
        MAX(guid) AS start_id,
	    MIN(dt) AS start_time,
        MIN(Convert.ToDateTime(Convert.ToDateTime(dt<@default_dt?@default_dt:dt).ToString("yyyy-MM-dd"))) AS start_zero_time,
        MIN(USQL_Programmability.CustomFunctions.GetFiscalPeriod(dt)) AS start_fiscalperiod,
        DateTime.Now.ToString("M/d/yyyy") AS Nowdate,
        user,
        des
	FROM @rs0
    GROUP BY user, des;
```

## Use inline C# function expressions
U-SQL allows the use of inline function expressions definition as part of C# expressions. This creates additional possibilities for using C# functions with output reference parameters.

Following is an example of a general inline function expression definition:

```
(Func<type of param1, type of param2>)
(param1 =>
     { … return param2}
) (Rowset Column)
```

Example:

```
(Func<string, DateTime?>)
(input_p =>
     {
	DateTime dt_result;
	return DateTime.TryParse(input_p, out dt_result)
	       ? (DateTime?) dt_result : (DateTime?) null;
     }
)
) (dt)
```

In this example, we define an inline function with the string input parameter input_p. Inside this function, we verify if input string is a valid datetime value. If it is, return it, otherwise return null.

The inline function is needed in this scenario because the DateTime.TryParse function contains the output parameter `out dt_result`. We define it as the `DateTime dt_result;`.


## Verify data type values
Some C# functions cannot be used directly in base U-SQL scripts as C# expressions. Specifically, the functions that can't be used directly are those that require an output reference parameter. However, these functions can be defined and used as part of an inline function expression, as discussed earlier.

### Use inline function expressions
To verify if DateTime value is valid, we can use `DateTime.TryParse`.

Here is a complete example that shows how to use an inline function expression to verify data type value with `DateTime.TryParse`.

In this scenario, we verify DateTime data type value:

```
DECLARE @input_file string = @"\usql-programmability\input_file.tsv";
DECLARE @output_file string = @"\usql-programmability\output_file.tsv";

@rs0 =
	EXTRACT
        guid Guid,
	    dt string,
        user String,
        des String
	FROM @input_file USING Extractors.Tsv();

@rs1 =
    SELECT
        MAX(guid) AS start_id,
        MIN((
             (Func<string, DateTime?>)
             (input_parameter =>
                { DateTime dt_result;
                  return
                     DateTime.TryParse(input_parameter, out dt_result)
                       ?(DateTime?) dt_result : (DateTime?) null;
                }
             )
             ) (dt)) AS start_time,
        DateTime.Now.ToString("M/d/yyyy") AS Nowdate,
        user,
        des
	FROM @rs0
    GROUP BY user, des;

OUTPUT @rs1 TO @output_file USING Outputters.Text();
```

### Use code-behind
To use the same functionality in the code-behind section of the U-SQL program, we define the C# function ToDateTime.

Here is the section of base U-SQL script in which we made the necessary changes:

```
@rs1 =
SELECT
  MAX(guid) AS start_id,
  MIN(USQL_Programmability.CustomFunctions.ToDateTime(dt)) AS start_time,
  DateTime.Now.ToString("M/d/yyyy") AS Nowdate,
  user,
  des
  FROM @rs0
GROUP BY user, des;
```

Here is the code-behind function:

```
static public DateTime? ToDateTime(string dt)
{
    DateTime dtValue;

    if (!DateTime.TryParse(dt, out dtValue))
	return Convert.ToDateTime(dt);
    else
	return null;
}
```

## Use code-behind
Code-behind is a C# programmability section of the U-SQL project. Conceptually, code-behind is a compiled assembly (DLL) that's referenced in U-SQL script. Visual Studio Tools enables you to manage and debug a C# programmability section as part of a U-SQL project.

When a typical U-SQL project is created in Visual Studio, there are two parts of the project: base script and a file with the extension **.usql**.

![Base script file](./media/data-lake-analytics-u-sql-programmability-guide/base-script-file.png)


Typical solution project:   
![Typical solution project](./media/data-lake-analytics-u-sql-programmability-guide/typical-solution-project.png)


In the second part of the project, we call a code-behind file: Script.usql.cs.  
![Code-behind](./media/data-lake-analytics-u-sql-programmability-guide/code-behind.png)

This file contains a default namespace definition for programmability objects.

```
using Microsoft.Analytics.Interfaces;
using Microsoft.Analytics.Types.Sql;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;

namespace USQL_Programmability
{

}
```

The preview code-behind template is generated automatically. This file contains a default namespace definition for programmability objects. During project execution, it gets compiled and referenced in base U-SQL script.

To start developing programmability objects, we need to create a **public** class.

```c#
namespace USQL_Programmability
{
    public class MyClass
    {
        public static string MyFunction(string param1)
        {
            return "my result";
        }

    }

}
```

The programmability objects can be user-defined functions, UDF, User-Defined Types, UDT, PROCESS, or REDUCER, and so on.

## Register U-SQL assemblies
U-SQL’s extensibility model relies heavily on the ability to add custom code. Currently, U-SQL provides you with easy ways to add your own Microsoft .NET-based code (in particular, C#). However, you can also add custom code that's written in other .NET languages, such as VB.NET or F#.

You can even deploy your own runtime for other languages, but you still need to provide the interoperability through a .NET layer yourself. If you want us to support a specific language, file a feature request or leave a comment at http://aka.ms/adlfeedback.

### Learn the difference between code-behind and assembly registration through Azure Data Lake Tools in Visual Studio
The easiest way to use custom code is to use the Azure Data Lake Tools for Visual Studio’s code-behind capabilities.

As we mentioned earlier, you fill in the custom code for the script (for example, Script.usql) into its code-behind file (for example, Script.usql.cs).

![Code-behind example](./media/data-lake-analytics-u-sql-programmability-guide/code-behind-example.jpg)
**Figure 1**: Code-behind example in Azure Data Lake Tools in Visual Studio. (Click the image to enlarge it. [Sample code](https://github.com/Azure/usql/tree/master/Examples/TweetAnalysis) is available.)


The advantage of code-behind is that the tooling takes care of the following steps for you when you submit your script:  

1. It builds the assembly for the code-behind file.  

2. It adds a prologue to the script that uses the [CREATE ASSEMBLY](https://msdn.microsoft.com/library/azure/mt763293.aspx) statement to register the assembly file. It also uses [REFERENCE ASSEMBLY]  (https://msdn.microsoft.com/library/azure/mt763294.aspx) to load the assembly into the script’s context.

3. It adds an epilogue to the script, which uses [DROP ASSEMBLY](https://msdn.microsoft.com/library/azure/mt763295.aspx) to remove the temporarily registered assembly again.

You can see the generated prologue and epilogue when you open the script:

![Generated prologue](./media/data-lake-analytics-u-sql-programmability-guide/generated-prologue.png)

**Figure 2**: Auto-generated prologue and epilogue for code-behind
<br />

The following are some of the drawbacks of code-behind:

* The code gets uploaded for every script submission.
* The functionality cannot be shared with others.

Thus, you can add a separate C# Class Library (for U-SQL) to your solution (see Figure 3), develop the code, or copy existing code-behind code (no changes in the C# code required, as shown in Figure 4). Then use the **Register Assembly** menu option on the project to register the assembly (as shown in Step 1 of Figure 5).

![Creating project](./media/data-lake-analytics-u-sql-programmability-guide/creating-project.png)
**Figure 3**: Creating a U-SQL C# code project  
<br />

![Class library](./media/data-lake-analytics-u-sql-programmability-guide/class-library.png)
**Figure 4**: The U-SQL C# class library next to the code-behind file
<br />

![Register code](./media/data-lake-analytics-u-sql-programmability-guide/register-code.png)
**Figure 5**: How to register the U-SQL C# code project
<br />

The registration dialog box (see Step 2 of Figure 5) gives you the options for how to register the assembly (for example, which Data Lake Analytics account to use, which database to use). It also gives you information about how to name the assembly. (The local assembly path gets filled in by the tool.) In addition, it provides an option to re-register an already registered assembly, and two options for adding additional dependencies:

**Managed dependencies**: Shows the managed assemblies that are needed. Each selected assembly is registered individually and becomes referenceable in scripts. You use this for other .NET assemblies.

**Additional files**: Enables you to add additional resource files that are needed by the assembly. They are registered together with the assembly and automatically loaded when the assembly gets referenced. You use this for config files, native assemblies, other language runtimes, their resources, and so on.

We use both of these options in the following examples. The [recent blog post about image processing](https://blogs.msdn.microsoft.com/azuredatalake/2016/08/18/introducing-image-processing-in-u-sql/) is another example that shows the use of a predefined assembly that can use these options for registration.

Now you can refer to the registered assemblies from any U-SQL script that has permissions for the database of the registered assemblies. (For more information, see the code in the U-SQL script in Figure 4.) You have to add a reference for every assembly that's registered separately. The additional resource files are automatically deployed. That script should not have a code-behind file for the code that's in the referenced assemblies anymore, but the code-behind file can still provide other code.

### Register assemblies via Azure Data Lake Tools in Visual Studio and in U-SQL scripts
While the Azure Data Lake Tools in Visual Studio make it easy to register an assembly, you can also do it with a script (in the same way that the tools do it for you) if you are (for example) developing on a different platform or have already compiled assemblies that you want to upload and register. You take the following steps:

1. Upload your assembly DLL and also all required non-system DLLs and resource files into a location that you choose. You can also upload it to your Azure Data Lake storage account or even a Microsoft Azure Blob storage account that's linked to your Azure Data Lake account. You can use any of the upload tools that are available to you (for example, Windows PowerShell commands, Visual Studio’s Azure Data Lake Tool Data Lake Explorer upload, your favorite SDK’s upload command, or tools that you access through the Azure portal).

1. After you have uploaded the DLLs, use the [CREATE ASSEMBLY](https://msdn.microsoft.com/library/azure/mt763293.aspx) statements to register them.

We use this approach in the following spatial example.

### Register assemblies that use other .NET assemblies (based on the JSON and XML sample library)
Our [U-SQL GitHub site](https://github.com/Azure/usql/) offers a set of shared example assemblies for you to use. One of the assemblies, called [Microsoft.Analytics.Samples.Formats](https://github.com/Azure/usql/tree/master/Examples/DataFormats), provides extractors, functions, and outputters to handle both JSON and XML documents. The Microsoft.Analytics.Samples.Formats assembly depends on two existing domain-specific assemblies to do the processing of the JSON and XML respectively. It uses the [Newtonsoft Json.Net](http://www.newtonsoft.com/) library for processing the JSON documents and the [System.Xml](https://msdn.microsoft.com/data/bb291078.aspx) assembly for processing XML. We use it to show how to register them and use the assemblies in our scripts.

First we download the [Visual Studio project](https://github.com/Azure/usql/tree/master/Examples/DataFormats) to our local development environment (for example, by making a local copy with the GitHub tool for Windows). Then we open the solution in Visual Studio and right-click the project (as explained previously) to register the assembly.

Though this assembly has two dependencies, we only have to include the Newtonsoft dependency because System.Xml is available in Azure Data Lake already (it has to be explicitly referenced, however). Figure 6 shows how we name the assembly (note that you can choose a different name without dots as well) and add the Newtonsoft DLL. Each of the two assemblies is individually registered in the specified database (for example, JSONBlog).

![Register assembly](./media/data-lake-analytics-u-sql-programmability-guide/register-assembly.png)

**Figure 6**: How to register the Microsoft.Analytics.Samples.Formats assembly from Visual Studio
<br />

If you or others (with whom you shared the registered assemblies by giving them read access to the database) now want to use the JSON capability in your scripts, add the following two references to your script:

```
REFERENCE ASSEMBLY JSONBlog.[NewtonSoft.Json];
REFERENCE ASSEMBLY JSONBlog.[Microsoft.Analytics.Samples.Formats];
```

And if you want to use the XML functionality, add a system assembly reference and a reference to the registered assembly:

```
REFERENCE SYSTEM ASSEMBLY [System.Xml];
REFERENCE ASSEMBLY JSONBlog.[Microsoft.Analytics.Samples.Formats];
```

For more information about how to use the JSON functionality, see [this blog post](https://blogs.msdn.microsoft.com/mrys/?p=755).

### Register assemblies that use native C++ assemblies (using the SQL Server 2016 spatial type assembly from the feature pack)
Now let’s look at a slightly different scenario. Let’s assume the assembly that we want to use has a dependency on code that is not .NET based. More specifically, the assembly has a dependency on a native C++ assembly. An example of such an assembly is the SQL Server type assembly [Microsoft.SqlServer.Types.dll](https://www.microsoft.com/download/details.aspx?id=52676). It provides .NET-based implementations of the SQL Server hierarchyID, geometry, and geography types to be used by SQL Server client-side applications for handling the SQL Server types. (It was also originally the assembly that provided the implementation for the SQL Server spatial types before the SQL Server 2016 release.)

Let’s look at how to register this assembly in U-SQL.

First we download and install the assembly from the [SQL Server 2016 feature pack](https://www.microsoft.com/download/details.aspx?id=52676). To ensure that you have the 64-bit version of the libraries, select the 64-bit version of the installer (ENU\x64\SQLSysClrTypes.msi).

The installer installs the managed assembly Microsoft.SqlServer.Types.dll into C:\Program Files (x86)\Microsoft SQL Server\130\SDK\Assemblies and the native assembly SqlServerSpatial130.dll into \Windows\System32\. Now we upload the assemblies into our Azure Data Lake Store (for example, into a folder called /upload/asm/spatial).

Because the installer has installed the native library into the system folder c:\Windows\System32, we have to make sure that we either copy SqlServerSpatial130.dll out from that folder before uploading it, or that the tool we use does not perform the [file system redirection](https://msdn.microsoft.com/library/windows/desktop/aa384187(v=vs.85).aspx) of system folders.

For example, if you want to upload it with the current Visual Studio Azure Data Lake File Explorer, you have to copy the file into another directory first. Otherwise--as of the time of the writing of this article--you upload the 32-bit version. The reason for this is that Visual Studio is a 32-bit application, which does File System Redirection in its Azure Data Lake upload file selection window. Then, when you run a U-SQL script that calls into the native assembly, you get the following (inner) error at runtime:

**Inner exception from user expression: An attempt was made to load a program with an incorrect format. (Exception from HRESULT: 0x8007000B)**

After uploading the two assembly files, we now register them in the database SQLSpatial with the following script:

```
DECLARE @ASSEMBLY_PATH string = "/upload/asm/spatial/";
DECLARE @SPATIAL_ASM string = @ASSEMBLY_PATH+"Microsoft.SqlServer.Types.dll";
DECLARE @SPATIAL_NATIVEDLL string = @ASSEMBLY_PATH+"SqlServerSpatial130.dll";

CREATE DATABASE IF NOT EXISTS SQLSpatial;
USE DATABASE SQLSpatial;

DROP ASSEMBLY IF EXISTS SqlSpatial;
CREATE ASSEMBLY SqlSpatial
FROM @SPATIAL_ASM
WITH ADDITIONAL_FILES =
     (
         @SPATIAL_NATIVEDLL
     );
```

In this case, we only register one U-SQL assembly and include the native assembly as a string dependency to the U-SQL assembly. To use the spatial assemblies, we only need to reference the U-SQL assembly. The additional file is automatically made available for the assembly. Here's a simple sample script that uses the spatial assembly:

```sql
REFERENCE SYSTEM ASSEMBLY [System.Xml];
REFERENCE ASSEMBLY SQLSpatial.SqlSpatial;

USING Geometry = Microsoft.SqlServer.Types.SqlGeometry;
USING Geography = Microsoft.SqlServer.Types.SqlGeography;
USING SqlChars = System.Data.SqlTypes.SqlChars;

@spatial =
    SELECT * FROM (VALUES
                   // The following expression is not using the native DDL
                   ( Geometry.Point(1.0,1.0,0).ToString()),    
                   // The following expression is using the native DDL
                   ( Geometry.STGeomFromText(new SqlChars("LINESTRING (100 100, 20 180, 180 180)"), 0).ToString())
                  ) AS T(geom);

OUTPUT @spatial
TO "/output/spatial.csv"
USING Outputters.Csv();
```

The SQL Types library has a dependency on the System.XML assembly, so we need to reference it. Also, some of the methods use the System.Data.SqlTypes types instead of the built-in C# types. Because System.Data is already included by default, we can reference the needed SQL type. The previous code is available on our [GitHub site](https://github.com/Azure/usql/tree/master/Examples/SQLSpatialExample).


### Use assembly versioning
Currently, U-SQL uses the .NET Framework version 4.5. So ensure that your own assemblies are compatible with that version of the runtime.

As mentioned earlier, U-SQL runs code in a 64-bit (x64) format. So make sure that your code is compiled to run on x64. Otherwise you get the incorrect format error shown earlier.

Each uploaded assembly DLL and resource file, such as a different runtime, a native assembly, or a config file, can be at most 400 MB. The total size of deployed resources, either via DEPLOY RESOURCE or via references to assemblies and their additional files, cannot exceed 3 GB.

Finally, note that each U-SQL database can only contain one version of any given assembly. For example, if you need both version 7 and version 8 of the NewtonSoft Json.Net library, you need to register them in two different databases. Furthermore, each script can only refer to one version of a given assembly DLL. In this respect, U-SQL follows the C# assembly management and versioning semantics.


## Use user-defined functions: UDF
U-SQL user-defined functions, or UDF, are programming routines that accept parameters, perform an action (such as a complex calculation), and return the result of that action as a value. The return value of UDF can only be a single scalar. U-SQL UDF can be called in U-SQL base script like any other C# scalar function.

We recommend that you initialize U-SQL user-defined functions as **public** and **static**.

```
public static string MyFunction(string param1)
{
    return "my result";
}
```

First let’s look at the simple example of creating a UDF.

In this use-case scenario, we need to determine the fiscal period, including the fiscal quarter and fiscal month of the first sign-in for the specific user. The first fiscal month of the year in our scenario is June.

To calculate fiscal period, we introduce the following C# function:

```
public static string GetFiscalPeriod(DateTime dt)
{
    int FiscalMonth=0;
    if (dt.Month < 7)
    {
	FiscalMonth = dt.Month + 6;
    }
    else
    {
	FiscalMonth = dt.Month - 6;
    }

    int FiscalQuarter=0;
    if (FiscalMonth >=1 && FiscalMonth<=3)
    {
	FiscalQuarter = 1;
    }
    if (FiscalMonth >= 4 && FiscalMonth <= 6)
    {
	FiscalQuarter = 2;
    }
    if (FiscalMonth >= 7 && FiscalMonth <= 9)
    {
	FiscalQuarter = 3;
    }
    if (FiscalMonth >= 10 && FiscalMonth <= 12)
    {
	FiscalQuarter = 4;
    }

    return "Q" + FiscalQuarter.ToString() + ":P" + FiscalMonth.ToString();
}
```

It simply calculates fiscal month and quarter and returns a string value. For June, the first month of the first fiscal quarter, we use “Q1:P1”. For July, we use “Q1:P2”, and so on.

This is a regular C# function that we are going to use in our U-SQL project.

Here is how the code-behind section looks in this scenario:

```
using Microsoft.Analytics.Interfaces;
using Microsoft.Analytics.Types.Sql;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;

namespace USQL_Programmability
{
    public class CustomFunctions
    {
        public static string GetFiscalPeriod(DateTime dt)
        {
            int FiscalMonth=0;
            if (dt.Month < 7)
            {
                FiscalMonth = dt.Month + 6;
            }
            else
            {
                FiscalMonth = dt.Month - 6;
            }

            int FiscalQuarter=0;
            if (FiscalMonth >=1 && FiscalMonth<=3)
            {
                FiscalQuarter = 1;
            }
            if (FiscalMonth >= 4 && FiscalMonth <= 6)
            {
                FiscalQuarter = 2;
            }
            if (FiscalMonth >= 7 && FiscalMonth <= 9)
            {
                FiscalQuarter = 3;
            }
            if (FiscalMonth >= 10 && FiscalMonth <= 12)
            {
                FiscalQuarter = 4;
            }

            return "Q" + FiscalQuarter.ToString() + ":" + FiscalMonth.ToString();
        }

    }

}
```

Now we are going to call this function from the base U-SQL script. To do this, we have to provide a fully qualified name for the function, including the namespace, which in this case is NameSpace.Class.Function(parameter).

```
USQL_Programmability.CustomFunctions.GetFiscalPeriod(dt)
```

Following is the actual U-SQL base script:

```
DECLARE @input_file string = @"\usql-programmability\input_file.tsv";
DECLARE @output_file string = @"\usql-programmability\output_file.tsv";

@rs0 =
	EXTRACT
        guid Guid,
	    dt DateTime,
        user String,
        des String
	FROM @input_file USING Extractors.Tsv();

DECLARE @default_dt DateTime = Convert.ToDateTime("06/01/2016");

@rs1 =
    SELECT
        MAX(guid) AS start_id,
	    MIN(dt) AS start_time,
        MIN(Convert.ToDateTime(Convert.ToDateTime(dt<@default_dt?@default_dt:dt).ToString("yyyy-MM-dd"))) AS start_zero_time,
        MIN(USQL_Programmability.CustomFunctions.GetFiscalPeriod(dt)) AS start_fiscalperiod,
        user,
        des
	FROM @rs0
    GROUP BY user, des;

OUTPUT @rs1 TO @output_file USING Outputters.Text();
```

Following is the output file of the script execution:

```
0d8b9630-d5ca-11e5-8329-251efa3a2941,2016-02-11T07:04:17.2630000-08:00,2016-06-01T00:00:00.0000000,"Q3:8","User1",""

20843640-d771-11e5-b87b-8b7265c75a44,2016-02-11T07:04:17.2630000-08:00,2016-06-01T00:00:00.0000000,"Q3:8","User2",""

301f23d2-d690-11e5-9a98-4b4f60a1836f,2016-02-11T09:01:33.9720000-08:00,2016-06-01T00:00:00.0000000,"Q3:8","User3",""
```

This example demonstrates a simple usage of inline UDF in U-SQL.

### Keep state between UDF invocations
U-SQL C# programmability objects can be more sophisticated, utilizing interactivity through the code-behind global variables. Let’s look at the following business use-case scenario.

In large organizations, users can switch between varieties of internal applications. These can include Microsoft Dynamics CRM, PowerBI, and so on. Customers might want to apply a telemetry analysis of how users switch between different applications, what the usage trends are, and so on. The goal for the business is to optimize application usage. They also might want to combine different applications or specific sign-on routines.

To achieve this goal, we have to determine session IDs and lag time between the last session that occurred.

We need to find a previous sign-in and then assign this sign-in to all sessions that are being generated to the same application. The first challenge is that U-SQL base script doesn't allow us to apply calculations over already-calculated columns with LAG function. The second challenge is that we have to keep the specific session for all sessions within the same time period.

To solve this problem, we use a global variable inside a code-behind section: `static public string globalSession;`.

This global variable is applied to the entire rowset during our script execution.

Here is the code-behind section of our U-SQL program:

```
using Microsoft.Analytics.Interfaces;
using Microsoft.Analytics.Types.Sql;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;

namespace USQLApplication21
{
    public class UserSession
    {
        static public string globalSession;
        static public string StampUserSession(string eventTime, string PreviousRow, string Session)
        {

            if (!string.IsNullOrEmpty(PreviousRow))
            {
                double timeGap = Convert.ToDateTime(eventTime).Subtract(Convert.ToDateTime(PreviousRow)).TotalMinutes;
                if (timeGap <= 60)
                    return Session;
                else
                    return Guid.NewGuid().ToString();
            }
            else
                return Guid.NewGuid().ToString();

        }

        static public string getStampUserSession(string Session)
        {
            if (Session != globalSession && !string.IsNullOrEmpty(Session))
            {
                globalSession = Session;
            }
            return globalSession;
        }

    }
}
```

This example shows the global variable `static public string globalSession;` used inside the `getStampUserSession` function and getting reinitialized each time the Session parameter is changed.

The U-SQL base script is as follows:

```
DECLARE @in string = @"\UserSession\test1.tsv";
DECLARE @out1 string = @"\UserSession\Out1.csv";
DECLARE @out2 string = @"\UserSession\Out2.csv";
DECLARE @out3 string = @"\UserSession\Out3.csv";

@records =
    EXTRACT DataId string,
            EventDateTime string,           
            UserName string,
            UserSessionTimestamp string

    FROM @in
    USING Extractors.Tsv();

@rs1 =
    SELECT EventDateTime,
           UserName,
LAG(EventDateTime, 1) OVER(PARTITION BY UserName ORDER BY EventDateTime ASC) AS prevDateTime,          
           string.IsNullOrEmpty(LAG(EventDateTime, 1) OVER(PARTITION BY UserName ORDER BY EventDateTime ASC)) AS Flag,           
           USQLApplication21.UserSession.StampUserSession
           (
           	EventDateTime,
           	LAG(EventDateTime, 1) OVER(PARTITION BY UserName ORDER BY EventDateTime ASC),
           	LAG(UserSessionTimestamp, 1) OVER(PARTITION BY UserName ORDER BY EventDateTime ASC)
           )
           AS UserSessionTimestamp
    FROM @records;

@rs2 =
    SELECT EventDateTime,
           UserName,
           LAG(EventDateTime, 1) OVER(PARTITION BY UserName ORDER BY EventDateTime ASC) AS prevDateTime,
           string.IsNullOrEmpty(LAG(EventDateTime, 1) OVER(PARTITION BY UserName ORDER BY EventDateTime ASC)) AS Flag,
           USQLApplication21.UserSession.getStampUserSession(UserSessionTimestamp) AS UserSessionTimestamp
    FROM @rs1
    WHERE UserName != "UserName";

OUTPUT @rs2
TO @out2
ORDER BY UserName, EventDateTime ASC
USING Outputters.Csv();
```

Function `USQLApplication21.UserSession.getStampUserSession(UserSessionTimestamp)` is called here during the second memory rowset calculation. It passes the `UserSessionTimestamp` column and returns the value until `UserSessionTimestamp` has changed.

The output file is as follows:

```
"2016-02-19T07:32:36.8420000-08:00","User1",,True,"72a0660e-22df-428e-b672-e0977007177f"
"2016-02-17T11:52:43.6350000-08:00","User2",,True,"4a0cd19a-6e67-4d95-a119-4eda590226ba"
"2016-02-17T11:59:08.8320000-08:00","User2","2016-02-17T11:52:43.6350000-08:00",False,"4a0cd19a-6e67-4d95-a119-4eda590226ba"
"2016-02-11T07:04:17.2630000-08:00","User3",,True,"51860a7a-1610-4f74-a9ea-69d5eef7cd9c"
"2016-02-11T07:10:33.9720000-08:00","User3","2016-02-11T07:04:17.2630000-08:00",False,"51860a7a-1610-4f74-a9ea-69d5eef7cd9c"
"2016-02-15T21:27:41.8210000-08:00","User3","2016-02-11T07:10:33.9720000-08:00",False,"4d2bc48d-bdf3-4591-a9c1-7b15ceb8e074"
"2016-02-16T05:48:49.6360000-08:00","User3","2016-02-15T21:27:41.8210000-08:00",False,"dd3006d0-2dcd-42d0-b3a2-bc03dd77c8b9"
"2016-02-16T06:22:43.6390000-08:00","User3","2016-02-16T05:48:49.6360000-08:00",False,"dd3006d0-2dcd-42d0-b3a2-bc03dd77c8b9"
"2016-02-17T16:29:53.2280000-08:00","User3","2016-02-16T06:22:43.6390000-08:00",False,"2fa899c7-eecf-4b1b-a8cd-30c5357b4f3a"
"2016-02-17T16:39:07.2430000-08:00","User3","2016-02-17T16:29:53.2280000-08:00",False,"2fa899c7-eecf-4b1b-a8cd-30c5357b4f3a"
"2016-02-17T17:20:39.3220000-08:00","User3","2016-02-17T16:39:07.2430000-08:00",False,"2fa899c7-eecf-4b1b-a8cd-30c5357b4f3a"
"2016-02-19T05:23:54.5710000-08:00","User3","2016-02-17T17:20:39.3220000-08:00",False,"6ca7ed80-c149-4c22-b24b-94ff5b0d824d"
"2016-02-19T05:48:37.7510000-08:00","User3","2016-02-19T05:23:54.5710000-08:00",False,"6ca7ed80-c149-4c22-b24b-94ff5b0d824d"
"2016-02-19T06:40:27.4830000-08:00","User3","2016-02-19T05:48:37.7510000-08:00",False,"6ca7ed80-c149-4c22-b24b-94ff5b0d824d"
"2016-02-19T07:27:37.7550000-08:00","User3","2016-02-19T06:40:27.4830000-08:00",False,"6ca7ed80-c149-4c22-b24b-94ff5b0d824d"
"2016-02-19T19:35:40.9450000-08:00","User3","2016-02-19T07:27:37.7550000-08:00",False,"3f385f0b-3e68-4456-ac74-ff6cef093674"
"2016-02-20T00:07:37.8250000-08:00","User3","2016-02-19T19:35:40.9450000-08:00",False,"685f76d5-ca48-4c58-b77d-bd3a9ddb33da"
"2016-02-11T09:01:33.9720000-08:00","User4",,True,"9f0cf696-c8ba-449a-8d5f-1ca6ed8f2ee8"
"2016-02-17T06:30:38.6210000-08:00","User4","2016-02-11T09:01:33.9720000-08:00",False,"8b11fd2a-01bf-4a5e-a9af-3c92c4e4382a"
"2016-02-17T22:15:26.4020000-08:00","User4","2016-02-17T06:30:38.6210000-08:00",False,"4e1cb707-3b5f-49c1-90c7-9b33b86ca1f4"
"2016-02-18T14:37:27.6560000-08:00","User4","2016-02-17T22:15:26.4020000-08:00",False,"f4e44400-e837-40ed-8dfd-2ea264d4e338"
"2016-02-19T01:20:31.4800000-08:00","User4","2016-02-18T14:37:27.6560000-08:00",False,"2136f4cf-7c7d-43c1-8ae2-08f4ad6a6e08"
```

This example demonstrates a more complicated use-case scenario in which we use a global variable inside a code-behind section that's applied to the entire memory rowset.

## Use user-defined types: UDT
User-defined types, or UDT, is another programmability feature of U-SQL. U-SQL UDT acts like a regular C# user-defined type. C# is a strongly typed language that allows the use of built-in and custom user-defined types.

U-SQL cannot implicitly serialize or de-serialize arbitrary UDTs when the UDT is passed between vertices in rowsets. This means that the user has to provide an explicit formatter by using the IFormatter interface. This provides U-SQL with the serialize and de-serialize methods for the UDT.

> [!NOTE]
> U-SQL’s built-in extractors and outputters currently cannot serialize or de-serialize UDT data to or from files even with the IFormatter set. So when you're writing UDT data to a file with the OUTPUT statement, or reading it with an extractor, you have to pass it as a string or byte array. Then you call the serialization and deserialization code (that is, the UDT’s ToString() method) explicitly. User-defined extractors and outputters, on the other hand, can read and write UDTs.

If we try to use UDT in EXTRACTOR or OUTPUTTER (out of previous SELECT), as shown here:

```
@rs1 =
    SELECT MyNameSpace.Myfunction_Returning_UDT(filed1) AS myfield
    FROM @rs0;

OUTPUT @rs1 TO @output_file USING Outputters.Text();
```

We receive the following error:

```
Error	1	E_CSC_USER_INVALIDTYPEINOUTPUTTER: Outputters.Text was used to output column myfield of type
MyNameSpace.Myfunction_Returning_UDT.

Description:

Outputters.Text only supports built-in types.

Resolution:

Implement a custom outputter that knows how to serialize this type, or call a serialization method on the type in
the preceding SELECT.	C:\Users\sergeypu\Documents\Visual Studio 2013\Projects\USQL-Programmability\
USQL-Programmability\Types.usql	52	1	USQL-Programmability
```

To work with UDT in outputter, we either have to serialize it to string with the ToString() method or create a custom outputter.

UDTs currently cannot be used in GROUP BY. If UDT is used in GROUP BY, the following error is thrown:

```
Error	1	E_CSC_USER_INVALIDTYPEINCLAUSE: GROUP BY doesn't support type MyNameSpace.Myfunction_Returning_UDT
for column myfield

Description:

GROUP BY doesn't support UDT or Complex types.

Resolution:

Add a SELECT statement where you can project a scalar column that you want to use with GROUP BY.
C:\Users\sergeypu\Documents\Visual Studio 2013\Projects\USQL-Programmability\USQL-Programmability\Types.usql
62	5	USQL-Programmability
```

To define a UDT, we have to:

* Add the following namespaces:

```c#
	using Microsoft.Analytics.Interfaces
	using System.IO;
```

* Add `Microsoft.Analytics.Interfaces`, which is required for the UDT interfaces. In addition, `System.IO` might be needed to define the IFormatter interface.

* Define a used-defined type with SqlUserDefinedType attribute.

**SqlUserDefinedType** is used to mark a type definition in an assembly as a user-defined type (UDT) in U-SQL. The properties on the attribute reflect the physical characteristics of the UDT. This class cannot be inherited.

SqlUserDefinedType is a required attribute for UDT definition.

The constructor of the class:  

* SqlUserDefinedTypeAttribute (type formatter)

* Type formatter: Required parameter to define an UDT formatter--specifically, the type of the `IFormatter` interface must be passed here.

```
[SqlUserDefinedType(typeof(MyTypeFormatter))]
public class MyType
{
…
}
```

* Typical UDT also requires definition of the IFormatter interface, as shown in the following example:

```
public class MyTypeFormatter : IFormatter<MyType>
{
    public void Serialize(MyType instance, IColumnWriter writer,
		   ISerializationContext context)
    {
	…
    }

    public MyType Deserialize(IColumnReader reader,
			  ISerializationContext context)
    {
	…
    }
}
```

The `IFormatter` interface serializes and de-serializes an object graph with the root type of \<typeparamref name="T">.

\<typeparam name="T">The root type for the object graph to serialize and de-serialize.

* **Deserialize**: De-serializes the data on the provided stream and reconstitutes the graph of objects.

* **Serialize**: Serializes an object, or graph of objects, with the given root to the provided stream.

`MyType` instance: Instance of the type.  
`IColumnWriter` writer / `IColumnReader` reader: The underlying column stream.  
`ISerializationContext` context: Enum that defines a set of flags that specifies the source or destination context for the stream during serialization.

* **Intermediate**: Specifies that the source or destination context is not a persisted store.

* **Persistence**: Specifies that the source or destination context is a persisted store.

As a regular C# type, a U-SQL UDT definition can include overrides for operators such as +/==/!=. It can also include static methods. For example, if we are going to use this UDT as a parameter to a U-SQL MIN aggregate function, we have to define < operator override.

Earlier in this guide, we demonstrated an example for fiscal period identification from the specific date in the format Qn:Pn (Q1:P10). The following example shows how to define a custom type for fiscal period values.

Following is an example of a code-behind section with custom UDT and IFormatter interface:

```
[SqlUserDefinedType(typeof(FiscalPeriodFormatter))]
public struct FiscalPeriod
{
    public int Quarter { get; private set; }

    public int Month { get; private set; }

    public FiscalPeriod(int quarter, int month):this()
    {
	this.Quarter = quarter;
	this.Month = month;
    }

    public override bool Equals(object obj)
    {
	if (ReferenceEquals(null, obj))
	{
	    return false;
	}

	return obj is FiscalPeriod && Equals((FiscalPeriod)obj);
    }

    public bool Equals(FiscalPeriod other)
    {
return this.Quarter.Equals(other.Quarter) && this.Month.Equals(other.Month);
    }

    public bool GreaterThan(FiscalPeriod other)
    {
return this.Quarter.CompareTo(other.Quarter) > 0 || this.Month.CompareTo(other.Month) > 0;
    }

    public bool LessThan(FiscalPeriod other)
    {
return this.Quarter.CompareTo(other.Quarter) < 0 || this.Month.CompareTo(other.Month) < 0;
    }

    public override int GetHashCode()
    {
	unchecked
	{
	    return (this.Quarter.GetHashCode() * 397) ^ this.Month.GetHashCode();
	}
    }

    public static FiscalPeriod operator +(FiscalPeriod c1, FiscalPeriod c2)
    {
return new FiscalPeriod((c1.Quarter + c2.Quarter) > 4 ? (c1.Quarter + c2.Quarter)-4 : (c1.Quarter + c2.Quarter), (c1.Month + c2.Month) > 12 ? (c1.Month + c2.Month) - 12 : (c1.Month + c2.Month));
    }

    public static bool operator ==(FiscalPeriod c1, FiscalPeriod c2)
    {
	return c1.Equals(c2);
    }

    public static bool operator !=(FiscalPeriod c1, FiscalPeriod c2)
    {
	return !c1.Equals(c2);
    }
    public static bool operator >(FiscalPeriod c1, FiscalPeriod c2)
    {
	return c1.GreaterThan(c2);
    }
    public static bool operator <(FiscalPeriod c1, FiscalPeriod c2)
    {
	return c1.LessThan(c2);
    }
    public override string ToString()
    {
	return (String.Format("Q{0}:P{1}", this.Quarter, this.Month));
    }

}

public class FiscalPeriodFormatter : IFormatter<FiscalPeriod>
{
public void Serialize(FiscalPeriod instance, IColumnWriter writer, ISerializationContext context)
    {
	using (var binaryWriter = new BinaryWriter(writer.BaseStream))
	{
	    binaryWriter.Write(instance.Quarter);
	    binaryWriter.Write(instance.Month);
	    binaryWriter.Flush();
	}
    }

public FiscalPeriod Deserialize(IColumnReader reader, ISerializationContext context)
    {
	using (var binaryReader = new BinaryReader(reader.BaseStream))
	{
var result = new FiscalPeriod(binaryReader.ReadInt16(), binaryReader.ReadInt16());
	    return result;
	}
    }
}
```

The defined type includes two numbers: quarter and month. Operators ==/!=/>/< and static method ToString() are defined here.

As mentioned earlier, UDT can be used in SELECT expressions, but cannot be used in OUTPUTTER/EXTRACTOR without custom serialization. It either has to be serialized as a string with ToString() or used with a custom OUTPUTTER/EXTRACTOR.

Now let’s discuss usage of UDT. In a code-behind section, we changed our GetFiscalPeriod function to the following:

```
public static FiscalPeriod GetFiscalPeriodWithCustomType(DateTime dt)
{
    int FiscalMonth = 0;
    if (dt.Month < 7)
    {
	FiscalMonth = dt.Month + 6;
    }
    else
    {
	FiscalMonth = dt.Month - 6;
    }

    int FiscalQuarter = 0;
    if (FiscalMonth >= 1 && FiscalMonth <= 3)
    {
	FiscalQuarter = 1;
    }
    if (FiscalMonth >= 4 && FiscalMonth <= 6)
    {
	FiscalQuarter = 2;
    }
    if (FiscalMonth >= 7 && FiscalMonth <= 9)
    {
	FiscalQuarter = 3;
    }
    if (FiscalMonth >= 10 && FiscalMonth <= 12)
    {
	FiscalQuarter = 4;
    }

    return new FiscalPeriod(FiscalQuarter, FiscalMonth);
}
```

As you can see, it returns the value of our FiscalPeriod type.

Here we provide an example of how to use it further in U-SQL base script. This example demonstrates different forms of UDT invocation from U-SQL script.

```
DECLARE @input_file string = @"c:\work\cosmos\usql-programmability\input_file.tsv";
DECLARE @output_file string = @"c:\work\cosmos\usql-programmability\output_file.tsv";

@rs0 =
	EXTRACT
        guid string,
	    dt DateTime,
        user String,
        des String
	FROM @input_file USING Extractors.Tsv();

@rs1 =
    SELECT guid AS start_id,
           dt,
           DateTime.Now.ToString("M/d/yyyy") AS Nowdate,
           USQL_Programmability.CustomFunctions.GetFiscalPeriodWithCustomType(dt).Quarter AS fiscalquarter,
           USQL_Programmability.CustomFunctions.GetFiscalPeriodWithCustomType(dt).Month AS fiscalmonth,
           USQL_Programmability.CustomFunctions.GetFiscalPeriodWithCustomType(dt) + new USQL_Programmability.CustomFunctions.FiscalPeriod(1,7) AS fiscalperiod_adjusted,
           user,
           des
    FROM @rs0;

@rs2 =
    SELECT start_id,
           dt,
           DateTime.Now.ToString("M/d/yyyy") AS Nowdate,
           fiscalquarter,
           fiscalmonth,
           USQL_Programmability.CustomFunctions.GetFiscalPeriodWithCustomType(dt).ToString() AS fiscalperiod,

	   // This user-defined type was created in the prior SELECT.  Passing the UDT to this subsequent SELECT would have failed if the UDT was not annotated with an IFormatter.
           fiscalperiod_adjusted.ToString() AS fiscalperiod_adjusted,
           user,
           des
    FROM @rs1;

OUTPUT @rs2 TO @output_file USING Outputters.Text();
```

Here's an example of a full code-behind section:

```
using Microsoft.Analytics.Interfaces;
using Microsoft.Analytics.Types.Sql;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.IO;

namespace USQL_Programmability
{
    public class CustomFunctions
    {
        static public DateTime? ToDateTime(string dt)
        {
            DateTime dtValue;

            if (!DateTime.TryParse(dt, out dtValue))
                return Convert.ToDateTime(dt);
            else
                return null;
        }

        public static FiscalPeriod GetFiscalPeriodWithCustomType(DateTime dt)
        {
            int FiscalMonth = 0;
            if (dt.Month < 7)
            {
                FiscalMonth = dt.Month + 6;
            }
            else
            {
                FiscalMonth = dt.Month - 6;
            }

            int FiscalQuarter = 0;
            if (FiscalMonth >= 1 && FiscalMonth <= 3)
            {
                FiscalQuarter = 1;
            }
            if (FiscalMonth >= 4 && FiscalMonth <= 6)
            {
                FiscalQuarter = 2;
            }
            if (FiscalMonth >= 7 && FiscalMonth <= 9)
            {
                FiscalQuarter = 3;
            }
            if (FiscalMonth >= 10 && FiscalMonth <= 12)
            {
                FiscalQuarter = 4;
            }

            return new FiscalPeriod(FiscalQuarter, FiscalMonth);
        }



        [SqlUserDefinedType(typeof(FiscalPeriodFormatter))]
        public struct FiscalPeriod
        {
            public int Quarter { get; private set; }

            public int Month { get; private set; }

            public FiscalPeriod(int quarter, int month):this()
            {
                this.Quarter = quarter;
                this.Month = month;
            }

            public override bool Equals(object obj)
            {
                if (ReferenceEquals(null, obj))
                {
                    return false;
                }

                return obj is FiscalPeriod && Equals((FiscalPeriod)obj);
            }

            public bool Equals(FiscalPeriod other)
            {
return this.Quarter.Equals(other.Quarter) &&    this.Month.Equals(other.Month);
            }

            public bool GreaterThan(FiscalPeriod other)
            {
return this.Quarter.CompareTo(other.Quarter) > 0 || this.Month.CompareTo(other.Month) > 0;
            }

            public bool LessThan(FiscalPeriod other)
            {
return this.Quarter.CompareTo(other.Quarter) < 0 || this.Month.CompareTo(other.Month) < 0;
            }

            public override int GetHashCode()
            {
                unchecked
                {
                    return (this.Quarter.GetHashCode() * 397) ^ this.Month.GetHashCode();
                }
            }

            public static FiscalPeriod operator +(FiscalPeriod c1, FiscalPeriod c2)
            {
return new FiscalPeriod((c1.Quarter + c2.Quarter) > 4 ? (c1.Quarter + c2.Quarter)-4 : (c1.Quarter + c2.Quarter), (c1.Month + c2.Month) > 12 ? (c1.Month + c2.Month) - 12 : (c1.Month + c2.Month));
            }

            public static bool operator ==(FiscalPeriod c1, FiscalPeriod c2)
            {
                return c1.Equals(c2);
            }

            public static bool operator !=(FiscalPeriod c1, FiscalPeriod c2)
            {
                return !c1.Equals(c2);
            }
            public static bool operator >(FiscalPeriod c1, FiscalPeriod c2)
            {
                return c1.GreaterThan(c2);
            }
            public static bool operator <(FiscalPeriod c1, FiscalPeriod c2)
            {
                return c1.LessThan(c2);
            }
            public override string ToString()
            {
                return (String.Format("Q{0}:P{1}", this.Quarter, this.Month));
            }

        }

        public class FiscalPeriodFormatter : IFormatter<FiscalPeriod>
        {
public void Serialize(FiscalPeriod instance, IColumnWriter writer, ISerializationContext context)
            {
                using (var binaryWriter = new BinaryWriter(writer.BaseStream))
                {
                    binaryWriter.Write(instance.Quarter);
                    binaryWriter.Write(instance.Month);
                    binaryWriter.Flush();
                }
            }

public FiscalPeriod Deserialize(IColumnReader reader, ISerializationContext context)
            {
                using (var binaryReader = new BinaryReader(reader.BaseStream))
                {
var result = new FiscalPeriod(binaryReader.ReadInt16(), binaryReader.ReadInt16());
                    return result;
                }
            }
        }

    }
}
```

## Use user-defined aggregates: UDAGG
User-defined aggregates are any aggregation-related functions that are not shipped out-of-the-box with U-SQL. The example can be an aggregate to perform custom math calculations, string concatenations, manipulations with strings, and so on.

The user-defined aggregate base class definition is as follows:

```c#
    [SqlUserDefinedAggregate]
    public abstract class IAggregate<T1, T2, TResult> : IAggregate
    {
        protected IAggregate();

        public abstract void Accumulate(T1 t1, T2 t2);
        public abstract void Init();
        public abstract TResult Terminate();
    }
```

**SqlUserDefinedAggregate** indicates that the type should be registered as a user-defined aggregate. This class cannot be inherited.

SqlUserDefinedType attribute is **optional** for UDAGG definition.


The base class allows you to pass three abstract parameters: two as input parameters and one as the result. The data types are variable and should be defined during class inheritance.

```
public class GuidAggregate : IAggregate<string, string, string>
{
string guid_agg;

public override void Init()
{
	…
}

public override void Accumulate(string guid, string user)
{
	…
}

public override string Terminate()
{
	…
}

}
```

* **Init** invokes once for each group during computation. It provides an initialization routine for each aggregation group.  
* **Accumulate** is executed once for each value. It provides the main functionality for the aggregation algorithm. It can be used to aggregate values with various data types that are defined during class inheritance. It can accept two parameters of variable data types.
* **Terminate** is executed once per aggregation group at the end of processing to output the result for each group.

To declare correct input and output data types, use the class definition as follows:

```
public abstract class IAggregate<T1, T2, TResult> : IAggregate
```

* T1: First parameter to accumulate
* T2: First parameter to accumulate
* TResult: Return type of terminate

For example:

```
public class GuidAggregate : IAggregate<string, int, int>
```

or

```
public class GuidAggregate : IAggregate<string, string, string>
```

### Use UDAGG in U-SQL
To use UDAGG, first define it in code-behind or reference it from the existent programmability DLL as discussed earlier.

Then use the following syntax:

```
AGG<UDAGG_functionname>(param1,param2)
```

Here is an example of UDAGG:

```
public class GuidAggregate : IAggregate<string, string, string>
{
string guid_agg;

public override void Init()
{
    guid_agg = "";
}

public override void Accumulate(string guid, string user)
{
    if (user.ToUpper()== "USER1")
    {
	guid_agg += "{" + guid + "}";
    }
}

public override string Terminate()
{
    return guid_agg;
}

}
```

And base U-SQL script:

```
DECLARE @input_file string = @"\usql-programmability\input_file.tsv";
DECLARE @output_file string = @" \usql-programmability\output_file.tsv";

@rs0 =
	EXTRACT
        guid string,
	    dt DateTime,
        user String,
        des String
	FROM @input_file USING Extractors.Tsv();

@rs1 =
    SELECT
        user,
        AGG<USQL_Programmability.GuidAggregate>(guid,user) AS guid_list
	FROM @rs0
    GROUP BY user;

OUTPUT @rs1 TO @output_file USING Outputters.Text();
```

In this use-case scenario, we concatenate class GUIDs for the specific users.

## Use user-defined objects: UDO
U-SQL enables you to define custom programmability objects, which are called user-defined objects or UDO.

The following is a list of UDO in U-SQL:

* User-defined extractors
	* Extract row by row
	* Used to implement data extraction from custom structured files

* User-defined outputters
	* Output row by row
	* Used to output custom data types or custom file formats

* User-defined processors
	* Take one row and produce one row
	* Used to reduce the number of columns or produce new columns with values that are derived from an existing column set

* User-defined appliers
	* Take one row and produce 0 to n rows
	* Used with OUTER/CROSS APPLY

* User-defined combiners
	* Combines rowsets--user-defined JOINs

* User-defined reducers
	* Take n rows and produce one row
	* Used to reduce the number of rows

UDO is typically called explicitly in U-SQL script as part of the following U-SQL statements:

* EXTRACT
* OUTPUT
* PROCESS
* COMBINE
* REDUCE

## Use user-defined extractors
U-SQL allows you to import external data by using an EXTRACT statement. An EXTRACT statement can use built-in UDO extractors:  

* *Extractors.Text()*: Provides extraction from delimited text files of different encodings.

* *Extractors.Csv()*: Provides extraction from comma-separated value (CSV) files of
different encodings.

* *Extractors.Tsv()*: Provides extraction from tab-separated value (TSV) files of different encodings.

It can be useful to develop a custom extractor. This can be helpful during data import if we want to do any of the following tasks:

* Modify input data by splitting columns and modifying individual values. The PROCESSOR functionality is better for combining columns.
* Parse unstructured data such as Web pages and emails, or semi-unstructured data such as XML/JSON.
* Parse data in unsupported encoding.

To define a user-defined extractor, or UDE, we need to create an `IExtractor` interface. All input parameters to the extractor, such as column/row delimiters, and encoding, need to be defined in the constructor of the class. The `IExtractor`  interface should also contain a definition for the `IEnumerable<IRow>` override as follows:

```c#
     [SqlUserDefinedExtractor]
     public class SampleExtractor : IExtractor
     {
         public SampleExtractor(string row_delimiter, char col_delimiter)
         {
            …

         }

         public override IEnumerable<IRow> Extract(IUnstructuredReader input, IUpdatableRow output)
         {
             …
         }
     }
```

The **SqlUserDefinedExtractor** attribute indicates that the type should be registered as a user-defined extractor. This class cannot be inherited.

SqlUserDefinedExtractor is an optional attribute for UDE definition. It used to define AtomicFileProcessing property for the UDE object.

* bool     AtomicFileProcessing   

* **true** = Indicates that this extractor requires atomic input files (JSON, XML, ...)
* **false** = Indicates that this extractor can deal with split / distributed files (CSV, SEQ, ...)

The main UDE programmability objects are **input** and **output**. The input object is used to enumerate input data as `IUnstructuredReader`. The output object is used to set output data as a result of the extractor activity.

The input data is accessed through `System.IO.Stream` and `System.IO.StreamReader`.

For input columns enumeration, we first split the input stream by using a row delimiter.

```
foreach (Stream current in input.Split(my_row_delimiter))
{
…
}
```

Then, further split input row into column parts.

```
foreach (Stream current in input.Split(my_row_delimiter))
{
…
	string[] parts = line.Split(my_column_delimiter);
	foreach (string part in parts)
	{
	…
	}
}
```

To set output data, we use the `output.Set` method.

It's important to understand that the custom extractor only outputs columns and values that are defined with the output. Set method call.

```
output.Set<string>(count, part);
```

The actual extractor output is triggered by calling `yield return output.AsReadOnly();`.

Following is the extractor example:

```
[SqlUserDefinedExtractor(AtomicFileProcessing = true)]
public class FullDescriptionExtractor : IExtractor
{
 private Encoding _encoding;
 private byte[] _row_delim;
 private char _col_delim;

public FullDescriptionExtractor(Encoding encoding, string row_delim = "\r\n", char col_delim = '\t')
{
     this._encoding = ((encoding == null) ? Encoding.UTF8 : encoding);
     this._row_delim = this._encoding.GetBytes(row_delim);
     this._col_delim = col_delim;

}

public override IEnumerable<IRow> Extract(IUnstructuredReader input, IUpdatableRow output)
{
     string line;
     //Read the input line by line
     foreach (Stream current in input.Split(_encoding.GetBytes("\r\n")))
     {
	using (System.IO.StreamReader streamReader = new StreamReader(current, this._encoding))
	 {
	     line = streamReader.ReadToEnd().Trim();
	     //Split the input by the column delimiter
	     string[] parts = line.Split(this._col_delim);
	     int count = 0; // start with first column
	     foreach (string part in parts)
	     {
if (count == 0)
		 {  // for column “guid”, re-generated guid
		     Guid new_guid = Guid.NewGuid();
		     output.Set<Guid>(count, new_guid);
		 }
		 else if (count == 2)
		 {
		     // for column “user”, convert to UPPER case
		     output.Set<string>(count, part.ToUpper());

		 }
		 else
		 {
		     // keep the rest of the columns as-is
		     output.Set<string>(count, part);
		 }
		 count += 1;
	     }

	 }
	 yield return output.AsReadOnly();
     }
     yield break;
 }
}
```

In this use-case scenario, the extractor regenerates the GUID for “guid” column and converts the values of “user” column to upper case. Custom extractors can produce more complicated results by parsing input data and manipulating it.

Following is base U-SQL script that uses a custom extractor:

```
DECLARE @input_file string = @"\usql-programmability\input_file.tsv";
DECLARE @output_file string = @"\usql-programmability\output_file.tsv";

@rs0 =
	EXTRACT
        guid Guid,
	    dt String,
        user String,
        des String
	FROM @input_file
    USING new USQL_Programmability.FullDescriptionExtractor(Encoding.UTF8);

OUTPUT @rs0 TO @output_file USING Outputters.Text();
```

## Use user-defined outputters
User-defined outputter is another U-SQL UDO that allows you to extend built-in U-SQL functionality. Similar to the extractor, there are several built-in outputters.

* *Outputters.Text()*: Writes data to delimited text files of different encodings.
* *Outputters.Csv()*: Writes data to comma-separated value (CSV) files of
different encodings.
* *Outputters.Tsv()*: Writes data to tab-separated value (TSV) files of different encodings.

Custom outputter allows you to write data in a custom defined format. This can be useful for the following tasks:

* Writing data to semi-structured or unstructured files.
* Writing data not supported encodings.
* Modifying output data or adding custom attributes.

To define user-defined outputter, we need to create the `IOutputter` interface.

Following is the base `IOutputter` class implementation:

```
public abstract class IOutputter : IUserDefinedOperator
{
protected IOutputter();

public virtual void Close();
public abstract void Output(IRow input, IUnstructuredWriter output);
}
```

All input parameters to the outputter, such as column/row delimiters, encoding, and so on, need to be defined in the constructor of the class. The `IOutputter` interface should also contain a definition for `void Output` override. The attribute `[SqlUserDefinedOutputter(AtomicFileProcessing = true)` can optionally be set for atomic file processing. For more information, see the following details.

```
[SqlUserDefinedOutputter(AtomicFileProcessing = true)]
public class MyOutputter : IOutputter
{

    public MyOutputter(myparam1, myparam2)
    {
      …
    }

    public override void Close()
    {
      …
    }

    public override void Output(IRow row, IUnstructuredWriter output)
    {
      …
    }
}
```

* `Output` is called for each input row. It returns the `IUnstructuredWriter output` rowset.
* The Constructor class is used to pass parameters to the user-defined outputter.
* `Close` is used to optionally override to release expensive state or determine when the last row was written.

**SqlUserDefinedOutputter** attribute indicates that the type should be registered as a user-defined outputter. This class cannot be inherited.

SqlUserDefinedOutputter is an optional attribute for a user-defined outputter definition. It's used to define the AtomicFileProcessing property.

* bool     AtomicFileProcessing   

* **true** = Indicates that this outputter requires atomic output files (JSON, XML, ...)
* **false** = Indicates that this outputter can deal with split / distributed files (CSV, SEQ, ...)

The main programmability objects are **row** and **output**. The **row** object is used to enumerate output data as `IRow` interface. **Output** is used to set output data to the target file.

The output data is accessed through the `IRow` interface. Output data is passed a row at a time.

The individual values are enumerated by calling the Get method of the IRow interface:

```
row.Get<string>("column_name")
```

Individual column names can be determined by calling `row.Schema`:

```
ISchema schema = row.Schema;
var col = schema[i];
string val = row.Get<string>(col.Name)
```

This approach enables you to build a flexible outputter for any metadata schema.

The output data is written to file by using `System.IO.StreamWriter`. The stream parameter is set to `output.BaseStrea` as part of `IUnstructuredWriter output`.

Note that it's important to flush the data buffer to the file after each row iteration. In addition, the `StreamWriter` object must be used with the Disposable attribute enabled (default) and with the **using** keyword:

```
using (StreamWriter streamWriter = new StreamWriter(output.BaseStream, this._encoding))
{
…
}
```

Otherwise, call Flush() method explicitly after each iteration. We show this in the following example.

### Set headers and footers for user-defined outputter
To set a header, use single iteration execution flow.

```
public override void Output(IRow row, IUnstructuredWriter output)
{
 …
if (isHeaderRow)
{
    …                
}

 …
if (isHeaderRow)
{
    isHeaderRow = false;
}
 …
}
}
```

The code in the first `if (isHeaderRow)` block is executed only once.

For the footer, use the reference to the instance of `System.IO.Stream` object (`output.BaseStream`). Write the footer in the Close() method of the `IOutputter` interface.  (For more information, see the following example.)

Following is an example of a user-defined outputter:

```
[SqlUserDefinedOutputter(AtomicFileProcessing = true)]
public class HTMLOutputter : IOutputter
{
    // Local variables initialization
    private string row_delimiter;
    private char col_delimiter;
    private bool isHeaderRow;
    private Encoding encoding;
    private bool IsTableHeader = true;
    private Stream g_writer;

    // Parameters definition            
    public HTMLOutputter(bool isHeader = false, Encoding encoding = null)
    {
	this.isHeaderRow = isHeader;
	this.encoding = ((encoding == null) ? Encoding.UTF8 : encoding);
    }

    // The Close method is used to write the footer to the file. It's executed only once, after all rows
    public override void Close().
    {
	//Reference to IO.Stream object - g_writer
	StreamWriter streamWriter = new StreamWriter(g_writer, this.encoding);
	streamWriter.Write("</table>");
	streamWriter.Flush();
	streamWriter.Close();
    }

    public override void Output(IRow row, IUnstructuredWriter output)
    {
	System.IO.StreamWriter streamWriter = new StreamWriter(output.BaseStream, this.encoding);

	// Metadata schema initialization to enumerate column names
	ISchema schema = row.Schema;

	// This is a data-independent header--HTML table definition
	if (IsTableHeader)
	{
	    streamWriter.Write("<table border=1>");
	    IsTableHeader = false;
	}

	// HTML table attributes
	string header_wrapper_on = "<th>";
	string header_wrapper_off = "</th>";
	string data_wrapper_on = "<td>";
	string data_wrapper_off = "</td>";

	// Header row output--runs only once
	if (isHeaderRow)
	{
	    streamWriter.Write("<tr>");
	    for (int i = 0; i < schema.Count(); i++)
	    {
		var col = schema[i];
		streamWriter.Write(header_wrapper_on + col.Name + header_wrapper_off);
	    }
	    streamWriter.Write("</tr>");
	}

	// Data row output
	streamWriter.Write("<tr>");                
	for (int i = 0; i < schema.Count(); i++)
	{
	    var col = schema[i];
	    string val = "";
	    try
	    {
		// Data type enumeration--required to match the distinct list of types from OUTPUT statement
		switch (col.Type.Name.ToString().ToLower())
		{
		    case "string": val = row.Get<string>(col.Name).ToString(); break;
		    case "guid": val = row.Get<Guid>(col.Name).ToString(); break;
		    default: break;
		}
	    }
	    // Handling NULL values--keeping them empty
	    catch (System.NullReferenceException)
	    {
	    }
	    streamWriter.Write(data_wrapper_on + val + data_wrapper_off);
	}
	streamWriter.Write("</tr>");

	if (isHeaderRow)
	{
	    isHeaderRow = false;
	}
	// Reference to the instance of the IO.Stream object for footer generation
	g_writer = output.BaseStream;
	streamWriter.Flush();
    }
}

// Define the factory classes
public static class Factory
{
    public static HTMLOutputter HTMLOutputter(bool isHeader = false, Encoding encoding = null)
    {
	return new HTMLOutputter(isHeader, encoding);
    }
}
```

And U-SQL base script:

```
DECLARE @input_file string = @"\usql-programmability\input_file.tsv";
DECLARE @output_file string = @"\usql-programmability\output_file.html";

@rs0 =
	EXTRACT
        guid Guid,
	    dt String,
        user String,
        des String
	FROM @input_file
    USING new USQL_Programmability.FullDescriptionExtractor(Encoding.UTF8);

OUTPUT @rs0 TO @output_file USING new USQL_Programmability.HTMLOutputter(isHeader: true);
```

This is an HTML outputter, which creates an HTML file with table data.

### Call outputter from U-SQL base script
To call a custom outputter from the base U-SQL script, the new instance of the outputter object has to be created.

```sql
OUTPUT @rs0 TO @output_file USING new USQL_Programmability.HTMLOutputter(isHeader: true);
```

To avoid creating an instance of the object in base script, we can create a function wrapper, as shown in our earlier example:

```c#
        // Define the factory classes
        public static class Factory
        {
            public static HTMLOutputter HTMLOutputter(bool isHeader = false, Encoding encoding = null)
            {
                return new HTMLOutputter(isHeader, encoding);
            }
        }
```

In this case, the original call looks like the following:

```
OUTPUT @rs0 
TO @output_file 
USING USQL_Programmability.Factory.HTMLOutputter(isHeader: true);
```

## Use user-defined processors
User-defined processor, or UDP, is a type of U-SQL UDO that enables you to process the incoming rows by applying programmability features. UDP enables you to combine columns, modify values, and add new columns if necessary. Basically, it helps to process a rowset to produce required data elements.

To define a UDP, we need to create an `IProcessor` interface with the `SqlUserDefinedProcessor` attribute, which is optional for UDP.

This interface should contain the definition for the `IRow` interface rowset override, as shown in the following example:

```
[SqlUserDefinedProcessor]
public class MyProcessor: IProcessor
{
public override IRow Process(IRow input, IUpdatableRow output)
 {
		…
 }
}
```

**SqlUserDefinedProcessor** indicates that the type should be registered as a user-defined processor. This class cannot be inherited.

The SqlUserDefinedProcessor attribute is **optional** for UDP definition.

The main programmability objects are **input** and **output**. The input object is used to enumerate input columns and output, and to set output data as a result of the processor activity.

For input columns enumeration, we use the `input.Get` method.

```
string column_name = input.Get<string>("column_name");
```

The parameter for `input.Get` method is a column that's passed as part of the `PRODUCE` clause of the `PROCESS` statement of the U-SQL base script. We need to use the correct data type here.

For output, use the `output.Set` method.

It's important to note that custom producer only outputs columns and values that are defined with the `output.Set` method call.

```
output.Set<string>("mycolumn", mycolumn);
```

The actual processor output is triggered by calling `return output.AsReadOnly();`.

Following is a processor example:

```
[SqlUserDefinedProcessor]
public class FullDescriptionProcessor : IProcessor
{
public override IRow Process(IRow input, IUpdatableRow output)
 {
     string user = input.Get<string>("user");
     string des = input.Get<string>("des");
     string full_description = user.ToUpper() + "=>" + des;
     output.Set<string>("dt", input.Get<string>("dt"));
     output.Set<string>("full_description", full_description);
     output.Set<Guid>("new_guid", Guid.NewGuid());
     output.Set<Guid>("guid", input.Get<Guid>("guid"));
     return output.AsReadOnly();
 }
}
```

In this use-case scenario, the processor is generating a new column called “full_description” by combining the existing columns--in this case, “user” in upper case, and “des”. It also regenerates a GUID and returns the original and new GUID values.

As you can see from the previous example, you can call C# methods during `output.Set` method call.

Following is an example of base U-SQL script that uses a custom processor:

```
DECLARE @input_file string = @"\usql-programmability\input_file.tsv";
DECLARE @output_file string = @"\usql-programmability\output_file.tsv";

@rs0 =
	EXTRACT
        guid Guid,
	    dt String,
        user String,
        des String
	FROM @input_file USING Extractors.Tsv();

@rs1 =
     PROCESS @rs0
     PRODUCE dt String,
             full_description String,
             guid Guid,
             new_guid Guid
     USING new USQL_Programmability.FullDescriptionProcessor();

OUTPUT @rs1 TO @output_file USING Outputters.Text();
```

## Use user-defined appliers
A U-SQL user-defined applier enables you to invoke a custom C# function for each row that's returned by the outer table expression of a query. The right input is evaluated for each row from the left input, and the rows that are produced are combined for the final output. The list of columns that are produced by the APPLY operator are the combination of the set of columns in the left and the right input.

User-defined applier is being invoked as part of the USQL SELECT expression.

The typical call to the user-defined applier looks like the following:

```
SELECT …
FROM …
CROSS APPLYis used to pass parameters
new MyScript.MyApplier(param1, param2) AS alias(output_param1 string, …);
```

For more information about using appliers in a SELECT expression, see [U-SQL SELECT Selecting from CROSS APPLY and OUTER APPLY](https://msdn.microsoft.com/library/azure/mt621307.aspx).

The user-defined applier base class definition is as follows:

```
public abstract class IApplier : IUserDefinedOperator
{
protected IApplier();

public abstract IEnumerable<IRow> Apply(IRow input, IUpdatableRow output);
}
```

To define a user-defined applier, we need to create the `IApplier` interface with the [`SqlUserDefinedApplier`] attribute, which is optional for a user-defined applier definition.

```
[SqlUserDefinedApplier]
public class ParserApplier : IApplier
{

public ParserApplier()
{
	…
}

public override IEnumerable<IRow> Apply(IRow input, IUpdatableRow output)
{
	…
}
}
```

* Apply is called for each row of the outer table. It returns the `IUpdatableRow` output rowset.
* The Constructor class is used to pass parameters to the user-defined applier.

**SqlUserDefinedApplier** indicates that the type should be registered as a user-defined applier. This class cannot be inherited.

**SqlUserDefinedApplier** is **optional** for a user-defined applier definition.


The main programmability objects are as follows:

```
public override IEnumerable<IRow> Apply(IRow input, IUpdatableRow output)
```

Input rowsets are passed as `IRow` input. The output rows are generated as `IUpdatableRow` output interface.

Individual column names can be determined by calling the `IRow` Schema method.

```
ISchema schema = row.Schema;
var col = schema[i];
string val = row.Get<string>(col.Name)
```

To get the actual data values from the incoming `IRow`, we use the Get() method of `IRow` interface.

```
mycolumn = row.Get<int>("mycolumn")
```

Or we use the schema column name:

```
row.Get<int>(row.Schema[0].Name)
```

The output values must be set with `IUpdatableRow` output:

```
output.Set<int>("mycolumn", mycolumn)
```

It is important to understand that custom appliers only output columns and values that are defined with `output.Set` method call.

The actual output is triggered by calling `yield return output.AsReadOnly();`.

The user-defined applier parameters can be passed to the constructor. Applier can return a variable number of columns that need to be defined during the applier call in base U-SQL Script.

```
new USQL_Programmability.ParserApplier ("all") AS properties(make string, model string, year string, type string, mileage int);
```

Here is the user-defined applier example:

```
[SqlUserDefinedApplier]
public class ParserApplier : IApplier
{
private string parsingPart;

public ParserApplier(string parsingPart)
{
    if (parsingPart.ToUpper().Contains("ALL")
	|| parsingPart.ToUpper().Contains("MAKE")
	|| parsingPart.ToUpper().Contains("MODEL")
	|| parsingPart.ToUpper().Contains("YEAR")
	|| parsingPart.ToUpper().Contains("TYPE")
	|| parsingPart.ToUpper().Contains("MILEAGE")
	)
    {
	this.parsingPart = parsingPart;
    }
    else
    {
	throw new ArgumentException("Incorrect parameter. Please use: 'ALL[MAKE|MODEL|TYPE|MILEAGE]'");
    }
}

public override IEnumerable<IRow> Apply(IRow input, IUpdatableRow output)
{

    string[] properties = input.Get<string>("properties").Split(',');

    //  only process with correct number of properties
    if (properties.Count() == 5)
    {

	string make = properties[0];
	string model = properties[1];
	string year = properties[2];
	string type = properties[3];
	int mileage = -1;

	// Only return millage if it is number, otherwise, -1
	if (!int.TryParse(properties[4], out millage))
	{
	    millage = -1;
	}

	if (parsingPart.ToUpper().Contains("MAKE") || parsingPart.ToUpper().Contains("ALL")) output.Set<string>("make", make);
	if (parsingPart.ToUpper().Contains("MODEL") || parsingPart.ToUpper().Contains("ALL")) output.Set<string>("model", model);
	if (parsingPart.ToUpper().Contains("YEAR") || parsingPart.ToUpper().Contains("ALL")) output.Set<string>("year", year);
	if (parsingPart.ToUpper().Contains("TYPE") || parsingPart.ToUpper().Contains("ALL")) output.Set<string>("type", type);
	if (parsingPart.ToUpper().Contains("MILEAGE") || parsingPart.ToUpper().Contains("ALL")) output.Set<int>("mileage", mileage);
    }
    yield return output.AsReadOnly();            
}
}
```

Following is the base U-SQL script for this user-defined applier:

```
DECLARE @input_file string = @"c:\usql-programmability\car_fleet.tsv";
DECLARE @output_file string = @"c:\usql-programmability\output_file.tsv";

@rs0 =
	EXTRACT
        stocknumber int,
	    vin String,
        properties String
	FROM @input_file USING Extractors.Tsv();

@rs1 =
    SELECT
        r.stocknumber,
	    r.vin,
        properties.make,
        properties.model,
        properties.year,
        properties.type,
        properties.mileage
	FROM @rs0 AS r
    CROSS APPLY
    new USQL_Programmability.ParserApplier ("all") AS properties(make string, model string, year string, type string, mileage int);

OUTPUT @rs1 TO @output_file USING Outputters.Text();
```

In this use case scenario, user-defined applier acts as a comma-delimited value parser for the car fleet properties. The input file rows look like the following:

```
103	Z1AB2CD123XY45889	Ford,Explorer,2005,SUV,152345
303	Y0AB2CD34XY458890	Shevrolet,Cruise,2010,4Dr,32455
210	X5AB2CD45XY458893	Nissan,Altima,2011,4Dr,74000
```

It is a typical tab-delimited TSV file with a properties column that contains car properties such as make and model. Those properties must be parsed to the table columns. The applier that's provided also enables you to generate a dynamic number of properties in the result rowset, based on the parameter that's passed. You can generate either all properties or a specific set of properties only.

	…USQL_Programmability.ParserApplier ("all")
	…USQL_Programmability.ParserApplier ("make")
	…USQL_Programmability.ParserApplier ("make&model")

The user-defined applier can be called as a new instance of applier object:

```
CROSS APPLY new MyNameSpace.MyApplier (parameter: “value”) AS alias([columns types]…);
```

Or with the invocation of a wrapper factory method:

```c#
	CROSS APPLY MyNameSpace.MyApplier (parameter: “value”) AS alias([columns types]…);
```

## Use user-defined combiners
User-defined combiner, or UDC, enables you to combine rows from left and right rowsets, based on custom logic. User-defined combiner is used with COMBINE expression.

A combiner is being invoked with the COMBINE expression that provides the necessary information about both the input rowsets, the grouping columns, the expected result schema, and additional information.

To call a combiner in a base U-SQL script, we use the following syntax:

```
Combine_Expression :=
	'COMBINE' Combine_Input
	'WITH' Combine_Input
	Join_On_Clause
	Produce_Clause
	[Readonly_Clause]
	[Required_Clause]
	USING_Clause.
```

For more information, see [COMBINE Expression (U-SQL)](https://msdn.microsoft.com/library/azure/mt621339.aspx).

To define a user-defined combiner, we need to create the `ICombiner` interface with the [`SqlUserDefinedCombiner`] attribute, which is optional for a user-defined Combiner definition.

Base `ICombiner` class definition:

```
public abstract class ICombiner : IUserDefinedOperator
{
protected ICombiner();
public virtual IEnumerable<IRow> Combine(List<IRowset> inputs,
       IUpdatableRow output);
public abstract IEnumerable<IRow> Combine(IRowset left, IRowset right,
       IUpdatableRow output);
}
```

The custom implementation of an `ICombiner` interface should contain the definition for an `IEnumerable<IRow>` Combine override.

```
[SqlUserDefinedCombiner]
public class MyCombiner : ICombiner
{

public override IEnumerable<IRow> Combine(IRowset left, IRowset right,
	IUpdatableRow output)
{
    …
}
}
```

The **SqlUserDefinedCombiner** attribute indicates that the type should be registered as a user-defined combiner. This class cannot be inherited.

**SqlUserDefinedCombiner** is used to define the Combiner mode property. It is an optional attribute for a user-defined combiner definition.

CombinerMode     Mode

CombinerMode enum can take the following values:

* Full  (0)	Every output row potentially depends on all the input rows from left and right
		with the same key value.

* Left  (1)	Every output row depends on a single input row from the left (and potentially all rows
		from the right with the same key value).

* Right (2)     Every output row depends on a single input row from the right (and potentially all rows
		from the left with the same key value).

* Inner (3)	Every output row depends on a single input row from left and right with the same value.

Example: 	[`SqlUserDefinedCombiner(Mode=CombinerMode.Left)`]


The main programmability objects are:

```c#
	public override IEnumerable<IRow> Combine(IRowset left, IRowset right,
		IUpdatableRow output
```

Input rowsets are passed as **left** and **right** `IRowset` type of interface. Both rowsets must be enumerated for processing. You can only enumerate each interface once, so we have to enumerate and cache it if necessary.

For caching purposes, we can create a List\<T\> type of memory structure as a result of a LINQ query execution, specifically List<`IRow`>. The anonymous data type can be used during enumeration as well.

See [Introduction to LINQ Queries (C#)](https://msdn.microsoft.com/library/bb397906.aspx) for more information about LINQ queries, and [IEnumerable\<T\> Interface](https://msdn.microsoft.com/library/9eekhta0(v=vs.110).aspx) for more information about IEnumerable\<T\> interface.

To get the actual data values from the incoming `IRowset`, we use the Get() method of `IRow` interface.

```
mycolumn = row.Get<int>("mycolumn")
```

Individual column names can be determined by calling the `IRow` Schema method.

```
ISchema schema = row.Schema;
var col = schema[i];
string val = row.Get<string>(col.Name)
```

Or by using the schema column name:

```
c# row.Get<int>(row.Schema[0].Name)
```

The general enumeration with LINQ looks like the following:

```
var myRowset =
            (from row in left.Rows
                          select new
                          {
                              Mycolumn = row.Get<int>("mycolumn"),
                          }).ToList();
```

After enumerating both rowsets, we are going to loop through all rows. For each row in the left rowset, we are going to find all rows that satisfy the condition of our combiner.

The output values must be set with `IUpdatableRow` output.

```
output.Set<int>("mycolumn", mycolumn)
```

The actual output is triggered by calling to `yield return output.AsReadOnly();`.

Following is a combiner example:

```
[SqlUserDefinedCombiner]
public class CombineSales : ICombiner
{

public override IEnumerable<IRow> Combine(IRowset left, IRowset right,
	IUpdatableRow output)
{
    var internetSales =
    (from row in left.Rows
		  select new
		  {
		      ProductKey = row.Get<int>("ProductKey"),
		      OrderDateKey = row.Get<int>("OrderDateKey"),
		      SalesAmount = row.Get<decimal>("SalesAmount"),
		      TaxAmt = row.Get<decimal>("TaxAmt")
		  }).ToList();

    var resellerSales =
    (from row in right.Rows
     select new
     {
	 ProductKey = row.Get<int>("ProductKey"),
	 OrderDateKey = row.Get<int>("OrderDateKey"),
	 SalesAmount = row.Get<decimal>("SalesAmount"),
	 TaxAmt = row.Get<decimal>("TaxAmt")
     }).ToList();

    foreach (var row_i in internetSales)
    {
	foreach (var row_r in resellerSales)
	{

	    if (
		row_i.OrderDateKey > 0
		&& row_i.OrderDateKey < row_r.OrderDateKey
		&& row_i.OrderDateKey == 20010701
		&& (row_r.SalesAmount + row_r.TaxAmt) > 20000)
	    {
		output.Set<int>("OrderDateKey", row_i.OrderDateKey);
		output.Set<int>("ProductKey", row_i.ProductKey);
		output.Set<decimal>("Internet_Sales_Amount", row_i.SalesAmount + row_i.TaxAmt);
		output.Set<decimal>("Reseller_Sales_Amount", row_r.SalesAmount + row_r.TaxAmt);
	    }

	}
    }
    yield return output.AsReadOnly();
}
}
```

In this use-case scenario, we are building an analytics report for the retailer. The goal is to find all products that cost more than $20,000 and that sell through the website faster than through the regular retailer within a certain time frame.

Here is the base U-SQL script. You can compare the logic between a regular JOIN and a combiner:

```sql
DECLARE @LocalURI string = @"\usql-programmability\";

DECLARE @input_file_internet_sales string = @LocalURI+"FactInternetSales.txt";
DECLARE @input_file_reseller_sales string = @LocalURI+"FactResellerSales.txt";
DECLARE @output_file1 string = @LocalURI+"output_file1.tsv";
DECLARE @output_file2 string = @LocalURI+"output_file2.tsv";

@fact_internet_sales =
EXTRACT
	ProductKey int ,
	OrderDateKey int ,
	DueDateKey int ,
	ShipDateKey int ,
	CustomerKey int ,
	PromotionKey int ,
	CurrencyKey int ,
	SalesTerritoryKey int ,
	SalesOrderNumber String ,
	SalesOrderLineNumber  int ,
	RevisionNumber int ,
	OrderQuantity int ,
	UnitPrice decimal ,
	ExtendedAmount decimal,
	UnitPriceDiscountPct float ,
	DiscountAmount float ,
	ProductStandardCost decimal ,
	TotalProductCost decimal ,
	SalesAmount decimal ,
	TaxAmt decimal ,
	Freight decimal ,
	CarrierTrackingNumber String,
	CustomerPONumber String
FROM @input_file_internet_sales
USING Extractors.Text(delimiter:'|', encoding: Encoding.Unicode);

@fact_reseller_sales =
EXTRACT
	ProductKey int ,
	OrderDateKey int ,
	DueDateKey int ,
	ShipDateKey int ,
	ResellerKey int ,
    EmployeeKey int ,
	PromotionKey int ,
	CurrencyKey int ,
	SalesTerritoryKey int ,
	SalesOrderNumber String ,
	SalesOrderLineNumber  int ,
	RevisionNumber int ,
	OrderQuantity int ,
	UnitPrice decimal ,
	ExtendedAmount decimal,
	UnitPriceDiscountPct float ,
	DiscountAmount float ,
	ProductStandardCost decimal ,
	TotalProductCost decimal ,
	SalesAmount decimal ,
	TaxAmt decimal ,
	Freight decimal ,
	CarrierTrackingNumber String,
	CustomerPONumber String
FROM @input_file_reseller_sales
USING Extractors.Text(delimiter:'|', encoding: Encoding.Unicode);

@rs1 =
SELECT
    fis.OrderDateKey,
    fis.ProductKey,
    fis.SalesAmount+fis.TaxAmt AS Internet_Sales_Amount,
    frs.SalesAmount+frs.TaxAmt AS Reseller_Sales_Amount
FROM @fact_internet_sales AS fis
     INNER JOIN @fact_reseller_sales AS frs
     ON fis.ProductKey == frs.ProductKey
WHERE
    fis.OrderDateKey < frs.OrderDateKey
    AND fis.OrderDateKey == 20010701
    AND frs.SalesAmount+frs.TaxAmt > 20000;

@rs2 =
COMBINE @fact_internet_sales AS fis
WITH @fact_reseller_sales AS frs
ON fis.ProductKey == frs.ProductKey
PRODUCE OrderDateKey int,
        ProductKey int,
        Internet_Sales_Amount decimal,
        Reseller_Sales_Amount decimal
USING new USQL_Programmability.CombineSales();

OUTPUT @rs1 TO @output_file1 USING Outputters.Tsv();
OUTPUT @rs2 TO @output_file2 USING Outputters.Tsv();
```

A user-defined combiner can be called as a new instance of the applier object:

```
USING new MyNameSpace.MyCombiner();
```


Or with the invocation of a wrapper factory method:

```
USING MyNameSpace.MyCombiner();
```

## Use user-defined reducers
U-SQL enables you to write custom rowset reducers in C# by using the user-defined operator extensibility framework and implementing an IReducer interface.

User-defined reducer, or UDR, can be used to eliminate unnecessary rows during data extraction (import). It also can be used to manipulate and evaluate rows and columns. Based on programmability logic, it can also define which rows need to be extracted.

To define a UDR class, we need to create an `IReducer` interface with an optional `SqlUserDefinedReducer` attribute.

This class interface should contain a definition for the `IEnumerable` interface rowset override.

```
[SqlUserDefinedReducer]
public class EmptyUserReducer : IReducer
{

    public override IEnumerable<IRow> Reduce(IRowset input, IUpdatableRow output)
    {
	    …
    }

}
```

The **SqlUserDefinedReducer** attribute indicates that the type should be registered as a user-defined reducer. This class cannot be inherited.
**SqlUserDefinedReducer** is an optional attribute for a user-defined reducer definition. It's used to define IsRecursive property.

* bool     IsRecursive    
* **true**  = Indicates whether this Reducer is idempotent

The main programmability objects are **input** and **output**. The input object is used to enumerate input rows. Output is used to set output rows as a result of reducing activity.

For input rows enumeration, we use the `Row.Get` method.

```
foreach (IRow row in input.Rows)
{
	row.Get<string>("mycolumn");
}
```

The parameter for the `Row.Get` method is a column that's passed as part of the `PRODUCE` class of the `REDUCE` statement of the U-SQL base script. We need to use the correct data type here as well.

For output, use the `output.Set` method.

It is important to understand that custom reducer only outputs values that are defined with the `output.Set` method call.

```
output.Set<string>("mycolumn", guid);
```

The actual reducer output is triggered by calling `yield return output.AsReadOnly();`.

Following is a reducer example:

```
[SqlUserDefinedReducer]
public class EmptyUserReducer : IReducer
{

    public override IEnumerable<IRow> Reduce(IRowset input, IUpdatableRow output)
    {
	string guid;
	DateTime dt;
	string user;
	string des;

	foreach (IRow row in input.Rows)
	{
	    guid = row.Get<string>("guid");
	    dt = row.Get<DateTime>("dt");
	    user = row.Get<string>("user");
	    des = row.Get<string>("des");

	    if (user.Length > 0)
	    {
		output.Set<string>("guid", guid);
		output.Set<DateTime>("dt", dt);
		output.Set<string>("user", user);
		output.Set<string>("des", des);

		yield return output.AsReadOnly();
	    }
	}
    }

}
```

In this use-case scenario, the reducer is skipping rows with an empty user name. For each row in rowset, it reads each required column, then evaluates the length of the user name. It outputs the actual row only if user name value length is more than 0.

Following is base U-SQL script that uses a custom reducer:

```
DECLARE @input_file string = @"\usql-programmability\input_file_reducer.tsv";
DECLARE @output_file string = @"\usql-programmability\output_file.tsv";

@rs0 =
	EXTRACT
        guid string,
	    dt DateTime,
        user String,
        des String
	FROM @input_file USING Extractors.Tsv();

@rs1 =
    REDUCE @rs0 PRESORT guid
    ON guid  
    PRODUCE guid string, dt DateTime, user String, des String
    USING new USQL_Programmability.EmptyUserReducer();

@rs2 =
    SELECT guid AS start_id,
           dt AS start_time,
           DateTime.Now.ToString("M/d/yyyy") AS Nowdate,
           USQL_Programmability.CustomFunctions.GetFiscalPeriodWithCustomType(dt).ToString() AS start_fiscalperiod,
           user,
           des
    FROM @rs1;

OUTPUT @rs2 TO @output_file USING Outputters.Text();
```
