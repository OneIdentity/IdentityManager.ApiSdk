# Request Properties

*Thanks to Stephan Hausmann for the information on this page.*

Request properties are a way to add additional parameters to an IT Shop request via configuration. Request properties can be queried during an approval process or process chain. 

In addition to reading this document, please also consult [the official documentation](
https://docs.oneidentity.com/bundle/one-identity-manager_it-shop-administration_9.3/page/sources/itshop/itsrequestpropertyedit.htm).

This document will not discuss all available options, but present you a decent amount of options as a solid foundation to start with.

# "Old" and "new" request properties

When you create a Request Property there is a flag where you can choose whether you plan to use the “old” or the “new” ones. The flag is called "obsolete definition".

<image>

We are not going to discuss the “old” request properties except to mention that they should work in the Web Designer Portal and the Angular portal at the same time, but is also limited compared to the “new” request properties. 

The fundamental change for the “new” Request Properties is that the values are not stored any more in the PersonWantsOrg table, but in dedicated table and no schema extensions are required. But, the “new” Request Properties work in the Angular Portal only. 

# A simple request property

Let’s start with a simple example, where we define a Request Property with one parameter, assign it to a Service Item and request the Service Item with a value for the parameter. Then we are going to have a look where the parameter value is stored and how to query it. 

We have seen in Figure 1 already the basic definition of the request property and we are going to add a parameter first. This parameter will ask for a user input. 

<image>

The request property needs to be referenced in the Service Item as shown in Figure 3. We assume, that you already know how to get the Service Item into an IT Shop etc. 

<image>

The following 3 screenshots show how the request property looks within the Angular portal when you request the Service Item. 

<images 4,5,6>

Figure 4: How does the request property look during the request? 

Figure 5: The request property after the request has been submitted 

Figure 6: View for an approver 

The definition of the request property can be found in the tables `AccProductParamCategory`, `DialogParameterSet` and `DialogParameter`. `DialogParameter` has a reference to `DialogParameterSet` and `DialogParameterSet` has a reference to `AccProductParamCategory`. The following three screenshots show the details.

<images 7,8,9>
Figure 7: Definition of the request property in table AccProductParamCategory 
Figure 8: DialogParameterSet - Reference to table AccProductParamCategory via XObjectKey 

Figure 9: Definition of Parameter1 in table DialogParameter. Reference to table DialogParameterSet via UID 

Once a request is submitted you find the values of the parameter(s) of the request property with the tables `DialogParameterSet` and `DialogParameter`. `DialogParameterSet` has a reference to `PersonWantsOrg` and `DialogParameter` has a reference to `DialogParameterSet`. The two next screenshots show the details. 

<images 10,11>

Figure 10: You find the request property after the request has been submitted in table DialogParameterSet via the XObjectKey of the PersonWantsOrg table. 

Figure 11: The values of the parameters can be found using a query for the DialogParameterSet (Figure 8 – queried via the XObjectKey of the PersonWantsOrg table) 

f you look for an example how to query the new request properties, there is an example available as part of the default installation. The script `TSB_PersonWantsOrg_HandleRequestWithParameters` does query the new request properties in the context of requests with dynamic parameters – if you haven’t seen that OOTB feature yet, it is worth having a look. 

# Mandatory properties and display names

The next example is how to make a parameter mandatory. Well, very simple just set the flag “Mandatory parameter” to true and in addition we are adding a display name to the parameter which is going to be used in the UI. The next three screenshots show how that is configured and how that looks like. 

<image 12, 13, 14>

Figure 12: Setting a display name, the sort order and making Parameter1 mandatory. Sort order will become relevant in the next chapter. 

Figure 13: In the Angular Portal the display name is shown and it is a mandatory parameter marked by a * 

Figure 14: Error message for mandatory parameter when the value remains empty 

# Default values

You can also set default values for a parameter which is shown in the next two screenshots. 

<images 15, 16>

# Validation scripts

When the logic you have to apply is a bit more complex, you can use scripts. We start with the validation of a parameter value. For that we create a simple script that tests if “1” is part of the value and if that is the case an error will be shown with a message containing the parameter value.  

``` vb
Dim str = Convert.ToString(value)

If str.Contains("1") Then
	Throw New ViException(#LD("Parameter1 must not contain {0}",str)#, ExceptionRelevance.EndUser)
End If
```

The next two screenshots show where to configure the script and how the error message looks like. 

<17,18>

# Sort order

For the next example we are going to add a second parameter. To control the display order of parameters we use the “Sort order” configuration. Parameter1 has “Sort order” 10 and Parameter2 gets 20 and therefore Parameter1 gets displayed first. 

Instead of typing a value, Parameter2 will allow to select a value from an existing table – which is going to be the Department table in our case. Instead of displaying the department name (which would be the default) we are going to show the full path of the department 

We are not going to display all departments, but filter the departments based on the value of Parameter 1. We are using the “like” example to remind you that adding two strings requires SQL logic. 

``` sql
FullPath like CONCAT('%',$PC(Parameter1)$,'%')
```

The next five screenshots show how the configuration looks in details and how the resulting UI looks like. 

<>

Figure 19: Parameter2 with sort order 20 (Parameter1 has 10), therefore Parameter2 will appear second. 

<>

Figure 20: We will select Table as Data source that will use a table as source for the list. We could also hard code a list of permitted values. 

<>

Figure 21: Table column (query) will be the UID of the table, we use the FullPath as Display value and make the Condition query dependent on Parameter1 

# Script for changing values

As the last example we will be calculating values of a parameter. When Parameter1 changes we are going to calculate the value for Parameter3. For that we are going to use “Script for changing values” 

Our example script is a very simple one: 

``` vb
ParameterSet("Parameter3").Value = Value
```

The next three screenshots show the details of the configuration and how the UI will look like. 

<img 24>

Figure 24: Script to run when the value of Parameter1 changes 

<img 25>
Figure 25: Default values do not trigger the script 

<img 26>

Figure 26: When the value of Parameter1 has been change the script  calculated the value for Parameter3 

The script may be triggered unexpected and you may only want to run your script when e.g. the String value of the old value is different from the String value of the new value. In that case you can add a simple String compare within your script


``` vb
If String.Compare( Convert.ToString(ParameterSet("Parameter1").Value) , Convert.ToString(Value) ) <> 0 Then
	ParameterSet("Parameter2").Value = Convert.ToString(value)
	ParameterSet("Parameter3").Value = ParameterSet("Parameter1").Value
End If
```

Thanks to the code completion feature you will find more methods you can use within your scripts. As a starting point I would like to mention three snippets to get a starting point for looking into details of the API. 

``` vb
ParameterSet("Parameter3").IsMandatory = True
Value = Connection.User.Uid
Value = Provider.GetValue(Of String)("UID_PersonInserted")
```

You can even think about selecting (or typing) a parameter value, load (=calculate) values from the database, modify them and submit the request for saving them after approval. 

As you can see, there are a lot of options using the new request properties for the Angular portal and much more as shown here is possible.

