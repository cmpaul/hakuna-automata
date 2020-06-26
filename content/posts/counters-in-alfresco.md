---
title: "Counters in Alfresco"
date: 2012-06-09T06:03:56-07:00
draft: false
---
I recently built a simple sequence generator in Alfresco as part of a project for a client who needed a way to configure and retrieve a unique integer value to be used both in Alfresco and outside of the system.

<!--more-->

The sequence needed to be global and maintain its current value across sessions and server restarts. While there were a few project-specific requirements, such as requiring multiple counters and a way to distinguish them, I've seen other requests for ways to do something along these lines. 

The approach I took was suggested by Mike Rogers, which is simply to create a custom type that will store the count as an attribute. My client needed a way to distinguish between multiple counters for various sequences so each is stored as a separate node. I then built a webscript to access and maintain it. These two steps are outlined below.

## 1. **Create the counter type**
Define a content model to include the following type. The type provides an integer attribute with a value to be between 0 and 99,999.

```xml
<type name="dm:counter">
    <title>Counter</title>
    <parent>cm:content</parent>
    <properties>
        <property name="dm:count">
            <title>Value</title>
            <type>d:int</type>
            <default>0</default>
            <constraints>
                <constraint name="dm:positiveInteger" type="MINMAX">
                    <parameter name="minValue">
                        <value>0</value>
                    </parameter>
                    <parameter name="maxValue">
                        <value>999999</value>
                    </parameter>
                </constraint>
            </constraints>
        </property>
    </properties>
</type>
```

The custom counter type allows us to instantiate unique, named counters in the repository. Another benefit to building this simple extended type is that Share configuration is not required as the default form config will automatically display all of the node's attributes.

## 2. **Build the counter webscripts**

Two webscripts provide an interface to retrieve an auto-incremented value and delete the counter. They are accessible at the URL: `/alfresco/service/counter/{name}[?value={value}]`

The `{name}` parameter is used to find the counter node in a default location: a folder called, "Counters," in the "Company Home" folder. 

If the node doesn't exist, it is created in this default location. The counter value is then retrieved from the node, incremented by one, stored back to the node, and returned in a JSON object. The optional `{value}` parameter is used to set the value of the counter. 

The creation, incrementing, and setting of the counter in the GET request means this isn't a 100% RESTful solution, but it wouldn't be hard to break these operations up into the appropriate endpoints.

First, the GET webscript. All webscript files for this example are in the repo project under `/config/extension/templates/webscripts/com/tribloom/counter`. The counter.get.desc.xml descriptor initializes the GET webscript with Alfresco:

```xml
<webscript>
    <shortname>counter</shortname>
    <description>Retrieves a counter value with the given name. The value will be auto-incremented. An optional value parameter can be used to set the counter.</description>
    <url>/counter/{name}?value={value}</url>
    <family>Counter Demo</family>
    <format default="json" />
    <authentication>user</authentication>
    <transaction>none</transaction>
</webscript>
```

Next, build the JavaScript controller, `counter.get.js`:

```javascript
<import resource="classpath:alfresco/extension/templates/webscripts/com/tribloom/counter/counter.js">

function main() {
	var value = args.value;
	var name = url.templateArgs.name;
	var counterNode = getCounter(name);
	if (counterNode == null) {
		counterNode = countersFolder.createNode(name, "dm:counter");
	}
	if (counterNode.islocked || !counterNode.hasPermission("Write")) {
		status.setCode(
			status.STATUS_NOT_AUTHORIZED,
			"Node is locked or you do not have permission to access it"
		);
		return;
	}
	if (value != null) {
		value = parseInt(value);
		if (isNaN(value)) {
			status.setCode(
				status.STATUS_BAD_REQUEST,
				"Invalid value: " + url.templateArgs.value
			);
			return;
		}
	} else {
		value = 1 + counterNode.properties["dm:count"];
	}
	counterNode.properties["dm:count"] = value;
	counterNode.save();
	model.name = name;
	model.value = value;
}
main();
```

Next, the FreeMarker template formats the response into a JSON object:

```javascript
{
  "name" : <#if name?exists>"${name}"<#else>null</#if>,
  "value" : <#if value?exists>${value}<#else>0</#if>
}
```

The `DELETE` webscript is simpler, taking only a descriptor and controller:

```xml
<webscript>
	<shortname>counter</shortname>
	<description>Deletes a counter value with the given name.</description>
	<url>/counter/{name}</url>
	<family>Counter Demo</family>
	<format default="json" />
	<authentication>user</authentication>
	<transaction>none</transaction>
</webscript>
<import resource="classpath:alfresco/extension/templates/webscripts/com/tribloom/counter/counter.js">

function main() {
	var name = url.templateArgs.name;
	var counterNode = getCounter(name);
	if (counterNode == null) {
		status.setCode(
			status.STATUS_NOT_FOUND, // 404
			"No counter found with the name " + name
		);
		return;
	}
	if (counterNode.hasPermission("Write")) {
		countersFolder.removeNode(counterNode);
		status.setCode(
			status.STATUS_OK, // 200
			"Counter " + name + " deleted."
		);
	}
}
main();
```

Once the model and webscripts are deployed, you can access the webscript URL from a field or form controller. 

In my projects, I've used this webscript to pre-populate read-only fields with an ID. It's important to note that this webscript is not synchronized against concurrent access. The result is that each call produces an ID that may be unique but is not sequential -- in other words, two subsequent requests may not produce sequential numbers. If multiple users will be accessing the webscript simultaneously it might be necessary to back the webscript with a Java class that can ensure synchronization. Something else to consider when implementing your counter is whether to expose the optional value parameter on the GET request or the ability to delete the counter. These will open the door to situations where the value returned is not unique.