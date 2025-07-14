# Request Properties

*Thanks to Stephan Hausmann for the information on this page.*

Request properties are a way to add additional parameters to an IT Shop request via configuration. Values of request properties can be queried during an approval process or process chain. 

In addition to reading this document, please also consult [the official documentation](
https://docs.oneidentity.com/bundle/one-identity-manager_it-shop-administration_9.3/page/sources/itshop/itsrequestpropertyedit.htm).

This document will not discuss all available options, but present you a decent amount of options as a solid foundation to start with.

Note: The information on this page is based on Identity Manager 9.2.1. (Some screenshots still show previous versions.)

# "Old" and "new" request properties

When you create a Request Property there is a flag where you can choose whether you plan to use the “old” or the “new” ones. The flag is called "obsolete definition".

![](<images/01.png>)

We are not going to discuss the “old” request properties except to mention that they should work in the Web Designer Portal and the Angular portal at the same time, but is also limited compared to the “new” request properties. 

The fundamental change for the “new” request properties is that the values are not stored  in the `PersonWantsOrg` table, but in a dedicated parameter table. No schema extensions are required. However, the “new” request properties only work in the Angular Portal only. 

# A simple request property

Let's start with a simple example, where we define a request property with one parameter, assign it to a service item and request the service item with a value for the parameter. Then we are going to have a look where the parameter value is stored and how to query it. 

We are going to add a parameter that will ask for a user input. 

![](<images/02.png>)

*Parameter type set to "User prompt" for getting an input from the user.*

The request property needs to be referenced in the service item as shown below.

![](<images/03.png>)

The following 3 screenshots show how the request property looks within the Angular portal when you request the Service Item. 

![](<images/04.png>)

*How does the request property look during the request?*

![](<images/05.png>)

*The request property after the request has been submitted*

![](<images/06.png>)

*View for an approver*

The definition of the request property can be found in the tables `AccProductParamCategory`, `DialogParameterSet` and `DialogParameter`. `DialogParameter` has a reference to `DialogParameterSet` and `DialogParameterSet` has a reference to `AccProductParamCategory`. The following three screenshots show the details.

![](<images/07.png>)

*Definition of the request property in table `AccProductParamCategory`* 

![](<images/08.png>)

*DialogParameterSet - Reference to table AccProductParamCategory via XObjectKey*

![](<images/09.png>)

*Definition of Parameter1 in table DialogParameter. Reference to table DialogParameterSet via UID*

Once a request is submitted, you will find the values of the parameter(s) of the request property in the tables `DialogParameterSet` and `DialogParameter`. `DialogParameterSet` has a reference to `PersonWantsOrg` and `DialogParameter` has a reference to `DialogParameterSet`. The two next screenshots show the details. 

![](<images/10.png>)

You find the request property after the request has been submitted in table `DialogParameterSet` via the XObjectKey of the `PersonWantsOrg` table. 

![](<images/11.png>)

The values of the parameters can be found using a query for the `DialogParameterSet` (queried via the XObjectKey of the `PersonWantsOrg` table).

There is an example for how to query the new request properties in the default installation. The script `QER_Get_ParameterValue_Of_ParameterSet_Of_PWO` queries the new request properties in the context of requests with dynamic parameters – if you haven’t seen that OOTB feature yet, it is worth having a look. 

# Mandatory properties and display names

The next example is how to make a parameter mandatory. Just set the flag “Mandatory parameter” to `true`. In addition, we are adding a display name to the parameter which is going to be used in the UI. The next three screenshots show how that is configured and what that looks like in the UI.

![](<images/12.png>)

*Setting a display name, the sort order and making Parameter1 mandatory. Sort order will become relevant in the next chapter.*

![](<images/13.png>)

*In the Angular Portal the display name is shown and it is a mandatory parameter marked by an asterisk* 

![](<images/14.png>)

*Error message for mandatory parameter when the value remains empty*

# Default values

You can also set default values for a parameter which is shown in the next two screenshots. 

![](<images/15.png>)

![](<images/16.png>)

# Validation scripts

When the logic becomes is a bit more complex, you can use scripts. We start with the validation of a parameter value. We create a simple script that tests if “1” is part of the value, and if that is the case, we show an error with a message containing the parameter value.  

``` vb
Dim str = Convert.ToString(value)

If str.Contains("1") Then
	Throw New ViException(#LD("Parameter1 must not contain {0}",str)#, ExceptionRelevance.EndUser)
End If
```

The next screenshot shows where to configure the script.

![](<images/17.png>)

# Sort order

For the next example we are going to add a second parameter. To control the display order of parameters, we use the “Sort order” configuration. Parameter1 has “Sort order” 10 and Parameter2 gets 20 and therefore Parameter1 gets displayed first. 

# Filtering

Instead of typing a value, Parameter2 will allow to select a value from an existing table – which is going to be the `Department` table in our case. Instead of displaying the department name (which would be the default), we are going to show the full path of the department. 

We are not going to display all departments, but filter the departments based on the value of Parameter 1. We are using the `like` example to remind you that adding two strings requires SQL logic. 

``` sql
FullPath like CONCAT('%',$PC(Parameter1)$,'%')
```

The next five screenshots show how the configuration looks in details and what the resulting UI looks like. 

![](<images/19.png>)

*Parameter2 with sort order 20 (Parameter1 has 10), therefore Parameter2 will appear second.*

![](<images/20.png>)

*We will select Table as Data source that will use a table as source for the list. We could also hard code a list of permitted values.*

![](<images/21.png>)

*Table column (query) will be the UID of the table, we use the FullPath as Display value and make the Condition query dependent on Parameter1*

# Script for changing values

As the last example we will be calculating values of a parameter. When Parameter1 changes, we are going to calculate the value for Parameter3. For that we are going to use “Script for changing values”.

Our example script is a very simple one: 

``` vb
ParameterSet("Parameter3").Value = Value
```

The next three screenshots show the details of the configuration and how the UI will look like. 

![](<images/24.png>)

*Script to run when the value of Parameter1 changes*

![](<images/25.png>)

*Default values do not trigger the script*

![](<images/26.png>)

*When the value of Parameter1 has changed the script, calculate the value for Parameter3*

The script will run multiple times, and you may only want to run the script when e.g. the old value is different to the new value. In that case, you can add a simple string comparison:

``` vb
If String.Compare( Convert.ToString(ParameterSet("Parameter1").Value) , Convert.ToString(Value) ) <> 0 Then
	ParameterSet("Parameter2").Value = Convert.ToString(value)
	ParameterSet("Parameter3").Value = ParameterSet("Parameter1").Value
End If
```

The code completion feature will show more helpful methods that you can use within your scripts.

## Making a parameter mandatory

``` vb
ParameterSet("Parameter3").IsMandatory = True
```

## Storing date values in a request property

When storing a `DateTime` value in a request property, it is stored as `string` value. Care must be taken to store the value in a culture-invariant format so that it can always be deserialized back into a `DateTime` value.

To store the date in a culture-invariant format, use the `ToString` overload shown in the following example.

``` vb
' assuming person is a Person entity, get its ExitDate value
If person.GetValue(Of Date)("ExitDate") > DbVal.MinDate Then
    ' set the NewExitDate parameter
		ParameterSet("NewExitDate").Value = (person.GetValue(Of Date)("ExitDate")).ToString("o", System.Globalization.CultureInfo.InvariantCulture)
End If
```

## Referencing the current user

To reference the UID of the logged-in user, use the `Connection.User.Uid` property.

To embed the UID of the logged-in user in a WHERE clause, you can use the `%useruid%` variable.

## Referencing the request recipient

The following script shows how to obtain the request recipient (`UID_PersonOrdered`).

``` vb
Dim objKey = TryCast(ParameterSet, IDialogParameterSet).UsedBy
Dim f As ISqlFormatter = Connection.SqlFormatter

Dim request As IEntity = Nothing

If objKey.Tablename = "ShoppingCartItem" AndAlso Connection.Session.Source().TryGet(objKey, request) Then
  Dim recipient As String = request.GetValue("UID_PersonOrdered").String

  ' set the query where clause with something that depends on the recipient, for example:
  TryCast(ParameterSet("Role"), IDialogParameter).QueryWhereClause = String.Format("UID_Org in (select UID_Org from PersonInOrg where {0})", f.UidComparison("UID_Person", recipient))
End If
```

## When does the script for changing values run?

It is important to keep in mind that this script is executed more often than you expect.

- The script for a parameter runs when the parameter set as a whole is *loaded*. This happens whenever some component (the UI, or a script) loads the parameter set in order to evaluate its values.
- The script runs whenever the value of the parameter *changes*.

You may want some code to run only in one of the two cases. You can use the `IsInit` variable to detect the specific case:

``` vb
If DbVal.ConvertTo(Of Boolean)(Variables("IsInit")) Then
  ' this code runs only on initialization
Else
  ' this code runs only when the parameter value changes
End If
```
