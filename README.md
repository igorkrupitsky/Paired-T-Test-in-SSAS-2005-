This article explains how to do Paired T-Test in SSAS 2005 (Microsoft SQL Analysis Services).

Introduction
This article assumes some prior knowledge of SSAS and MDX.

Suppose you have invented a pill that would let you loose weight and want to test it. You managed to convince eight people to measure their weight, take the pill for a month, and measure their weight again. After a month, they come back with the results. Some lost weight while others did not. Although on average every person lost weight, how do you calculate the statistical significance of the improvement? Perhaps, you were just lucky and the pill did not work. This article explains how to calculate the probability that the weight change was due to chance (or the P-Value) using SSAS.

You might ask: Why not just perform this calculation in Excel? After all, Excel provides the TTEST() function to create such a test. Here you can see this calculation done in Excel. Here is an article that explains in detail how this can be done in Excel.

Well, you should probably use Excel for a single survey. However, if you have multiple surveys taken at different subjects, different locations, and different times, SSAS will provide a nice way of analyzing your data while providing you with statistical significance. Even with a single survey, you might want to slice the survey data by some demographic attribute such as age or gender. For example, as you slice down to weight improvements for 20-30 year old females, you might have too few surveys to draw any statistically significant conclusions and the P-Value will tell you this.

This article uses a SQL Server database called "PairedT-Test". Please use this SQL script file: PairedT-Test.sql to create it. The database has the following Entity Relationship Diagram:

Image 1

This article also uses the [Paired T- Test] SSAS Cube with a Person and Trial dimensions.

Image 2

You can restore the OLAP database from this backup file: PairedT-Test.abf. Make sure to update the connection and password to the PairedT-Test database. Alternatively, you can restore the OLAP database from the XMLA script file: PairedT-Test.xmla. Make sure to update the connection and password to the PairedT-Test database and to process the database.

Also, make sure that the SSAS_Stat_Func.dll assembly is registered in the cube’s Assemblies folder.

Image 3

To see all of the survey data in the "PairedT-Test" OLAP database, please run this MDX statement in SQL Server Management Studio:

SQL
  WITH
  MEMBER [Measures].[Weight Before] AS ([Measures].[Avg Weight], [Trial].[Trial].&[1])
  MEMBER [Measures].[Weight After]  AS ([Measures].[Avg Weight], [Trial].[Trial].&[2])
  SELECT {
  [Measures].[Weight Before],
  [Measures].[Weight After],
  [Measures].[Weight Diff]
  } on COLUMNS, {
  [Person].[Person].[Person].Members,
  [Person].[Person].[All]} on ROWS
  FROM [Paired T- Test]

You will see the following results:

Image 4

You can see the weight for each person before and after they have taken your weight loss pill and the difference between before and after.

To see the summary including the Statistical Significance information including the P-Value, run this MDX statement:

  SQL
  SELECT {
  [Measures].[Weight Diff],
  [Measures].[Weight Diff Std Dev],
  [Measures].[Standard Error],
  [Measures].[T-Value],
  [Measures].[P-Value]
  } on COLUMNS,
  {[Person].[Person].[All]} on ROWS
  FROM [Paired T- Test]

You will see the following results:

Image 5

P-Value (of 0.76 %) gives you the probability that the difference in weight (19.38) can be contributed to chance alone.

The [Paired T- Test] cube has the following calculations:

[Avg Weight]	[Measures].[Weight]/[Measures].[Count]
[Weight Diff]	([Measures].[Avg Weight], [Trial].[Trial].&[1]) - ([Measures].[Avg Weight], [Trial].[Trial].&[2])
[Weight Diff Std Dev]	STDDEV([Person].[Person].[Person].Members, [Measures].[Weight Diff])
[Standard Error]	([Measures].[Weight Diff Std Dev] / (([Measures].[Count]/2)^0.5))
[T-Value]	[Measures].[Weight Diff] / [Measures].[Standard Error]
[P-Value]	SSAS_Stat_Func.GetPValue([Measures].[T-Value], ([Measures].[Count]/2)-1)

The P-Value gives you the probability that the difference in weight can be contributed to chance alone. P-Value is calculated using the following steps:

Calculate the difference in weight for each individual.
Calculate the Average for difference in weight for each individual.
Calculate the Standard Deviation for difference in weight for each individual.
Calculate the Standard Error by dividing the Standard Deviation by the square root of the number of people.
Calculate the T-Value by dividing the Average for difference in weight by Standard Error.
Calculate the P-Value by using a very complex formula that is contained in the assembly described below.
In my statistics class, we were told to use a table at the end of the textbook to lookup the P-Value for a given T-Value and a degree of freedom (number of individuals minus one). Because I could not tell SSAS to use my statistics notebook, I had to figure out the formula used to generate the table at the end of my textbook. A Google search pointed me to this page that provided me with the needed formula in JavaScript.

I used the JavaScript formula to write the function GetPValue() in VB.NET. The SSAS_Stat_Func.dll assembly exposes a single function GetPValue(). The GetPValue function returns the P-Value and takes in two parameters: the T-Value and the degree of freedom.

VB

Public Function GetPValue(ByVal t As Double, ByVal n As Long) As Double
    Dim PiD2 As Double = Math.PI / 2
    t = Math.Abs(t)

    Dim w As Double = t / Math.Sqrt(n)
    Dim th As Double = Math.Atan(w)

    If (n = 1) Then
        Return 1 - th / PiD2
    End If

    Dim sth As Double = Math.Sin(th)
    Dim cth As Double = Math.Cos(th)

    If ((n Mod 2) = 1) Then
        Return 1 - (th + sth * cth * StatCom(cth * cth, 2, n)) / PiD2
    Else
        Return 1 - sth * StatCom(cth * cth, 1, n)
    End If
End Function

Function StatCom(ByVal q As Double, ByVal i As Long, ByVal j As Long) As Double
    Dim zz As Double = 1
    Dim z As Double = zz
    Dim k As Long = i

    While (k <= (j - 3))
        zz = zz * q * k / (k + 1)
        z = z + zz
        k = k + 2
    End While

    Return z
End Function
