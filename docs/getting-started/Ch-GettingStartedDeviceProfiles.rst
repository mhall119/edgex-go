#############################
Understanding Device Profiles
#############################

=========================
What is a Device Profile?
=========================

A Device Profile describes the capabilities of a model or class of devices in a way that EdgeX services understand. A typical profile will define the datatypes that the device uses, the values that the device can povide, and commands that can be called against the device for reading values or triggering actions. Every Device that EdgeX is made aware of must be associated with a Device Profile. It's possible, even likely, that you will have more than one Device deployed that uses the same Profile.

A Device Profile can be added to EdgeX using the Metadata APIs directly, but they are also commonly added automatically by a Device Service if that service is designed to handle only a specific set of devices. For example, the `GrovePi Device Service <https://github.com/edgexfoundry/device-grove-c/>`_ used by the `Community Devkit <https://www.edgexfoundry.org/devkits/community-devkit/>`_ is designed to work only with the GrovePi sensor devices, and so it provides a `Device Profile <https://github.com/edgexfoundry/device-grove-c/blob/master/res/Grove_Device.yaml>`_ that will be automatically installed when that device service is started. Conversely, the `MQTT Device Service <https://github.com/edgexfoundry/device-mqtt-go>`_ can handle any type of device that uses MQTT, and so you would need to add your device profiles yourself when using this device service.

===========================
Anatomy of a Device Profile
===========================

The Device Profile is composed of four primary sections: The summary of the device, the datatype definitions, the resources available to be ready or written to, and the commands that can be called. Below we will take a look at each of these sections individually, using the :download:`sample device profile from the SDK guide <random-generator-device.yaml>` as our example.

--------------
Device Summary
--------------

Every profile begins with information about the type of device that the profile describes. With the exception of the `name` property, which is used as the identifier for this profile within EdgeX, everything else is here to assist human interactions.

::

    name: "RandNum-Device"
    manufacturer: "EdgeX Foundry"
    model: "abc123"
    labels:
        - "random"
        - "test"
    description: "random number generator to simulate a device"

Because our example profile describes a virtual device, not a real one, we use dummy data for `manufacturer` and `model`.

You can use as many `labels` as you like, and they can be any string you like. These can be used when calling the Metadata APIs to lookup multiple device profiles which share the same label.

----------------
Device Resources
----------------

The `deviceResources` section is where you will define the types of data that will be used when reading or writing to these devices. These define the format, restrictions, and optionally transformations that need to be applied to data of that type.

This section actually is doing more than just defining datatypes, each `deviceResource` entry in this section actually defines a specific device resource which be read and/or written. Each entry in this section must define its datatype, which are sometimes referred to as `PropertyValue`, as their definition is via a `value` field in the the `deviceResource`'s `properties` map.

Keep in mind that these datatypes can be used for different meanings, and so should be kept generic. For example, one `Temperature` type definition could be used for both `Water Temperature` and `Air Temperature` readings, as well as `Thermostat Setting` values.

This is incorrect, please see my previous comment about `deviceResource` entries...

::

    deviceResources:
        - name: "randomnumber"
          description: "generated random number"
          attributes:
            { type: "randdom" }
          properties:
            value:
                { type: "INT32", readWrite: "R", defaultValue: "0.00", minimum: "0.00", maximum: "100.00"  }
            units:
                { type: "String", readWrite: "R", defaultValue: "" }

The device services requirements specifies that the types are the same as native Go type names, which are all lower-case. I'm not sure if the code is case-insensitive, so the examples should probably use lower-case too. We should make sure this is tested so that we can definitively document the correct usage.

In our example Device Profile we deal with only one data type, the `randomnumber`. It is defined as a 32bit signed integer with a minimum values of 0.00 and a maximum of 100.00, defaulting to 0.00. We also must specify a unit property for this data, which would be `Degrees Celcius` for a Temperature, but in our case the `randomnumber` has no unit, so we can just let it default to an empty string.

**NOTE** I'm pretty sure default, minimum, and maximum are no-ops for the Delhi release.

`Question` - why do we need a readWrite field for a UnitProperty? This doesn't really make any sense...

Like the `name` property in the profile summary, the `name` of the deviceResource is also used as an identifier both within EdgeX and elsewhere in the Device Profile.

**TODO**: What are `attributes` and when do they get used?

`attributes` are protocol specific properties (e.g. a BLE attribute UUID) that are used by the protocol layer of a device service to access the actual device resource (aka reading, device object, ...). When a device service handles a command for a specific device resource, the attribute map is passed to the protocol layer of the device service as part of the command.

**TODO**: Why do these have `readWrite` properties? What does that do?

I'm going to assume you're asking about the `readWrite` keys in the PropertyValue and PropertyUnit maps? The former controls whether the property can be read, written, or both. As mentioned previously, I don't know `readWrite` is ever used for a PropertyUnit...

**TODO**: What other keys are available for property value? (size, word, offset, etc defined in PropertyValue.go)

Please see Appendix D, PropertyValue. Note that `size` and `word` are both deprecated:

https://wiki.edgexfoundry.org/download/attachments/7602423/edgex-cali-delhi-device-requirements-v8.pdf?version=1&modificationDate=1533761857000&api=v2

**TODO**: When/where are units used?

As I understand it, the PropertyUnit is meant for clients of EdgeX when displaying the changing values of a `deviceResource`.

---------
Resources
---------

A Resource is a data value with context. It combines the datatypes defined in the `deviceResources` section above with a meaning behind it. Here is where you would define things like `Water Temperature` and `Air Temperature` readings, as well as writable values.

Please review my description of resources in the device requirements document (v8 or the google doc version which has been updated recenty). From my understanding, device resources are meant for aggregation. They allow a single command to trigger multiple operations (i.e. get/read, put/write) to device resources. For instance, the device-virtual device service allows for the `enableRandomization` and `collectionFrequency` device resources to be set together in a single transaction.

::

    resources:
        - name: "Random"
          get:
            - { operation: "get", object: "randomnumber", property: "value", parameter: "Random" }

Once again our example profile is quite simple, defining only one resource named `Random` which is the random number that our simulated device will produce. You can see how it is defined as being of the `randomnumber` device resource that we previously defined.

A Resource can have lists of `get` or a `set` operations (or both) depending on your device capabilities.

**TODO**: When to use `object` and when to use `resource`

`object` refers to a device resource (aka a device object), whereas `resource` refers to another resource, used for aggregation of resources, in addition to aggregation of device resources.

**TODO**: What does `property` do and what values can it take?

The original design allowed for additional properties to be added to device resource, however the code (see device-sdk ProtocolHandler class) only checks for the existence of a `value` property, throwing an error if not found:

::
    //TODO Add property flexibility
    if (!operation.getProperty().equals("value")
        throw new ServiceException(new UnsupportedOperationException("Only property of value is implemented for this service!"));

**TODO**: What does 'parameter' do?

`Parameter` appears to be used to set the name of the `reading` generated by the resource operation.

**Note** - we should check this against the behavior of the Java device SDK, which appears to use the associated deviceObject name for readings. If a resource defines multiple get/read operations, and the resulting readings for each is set to the value of `Parameter` which is typically set to the resource name (e.g. "All"), then all readings would have the same name!

--------
Commands
--------

Commands are how EdgeX calls your device (via the Device Service) to retrieve data, change settings or trigger actions. You can define multiple commands, and each command can have either a `get` or a `put` function (or both). Note that here we use `put` instead of `set` as above, because a `put` operation doesn't necessarily result in setting a Resource value.

Again, please review the device services requirements document, although this is one area which I didn't get quite right...  As I understand it, the commands section describes commands which can be used by an external client of EdgeX to interact with a device managed by one of EdgeX's device services. The Core Command service uses this section to provide command support for all active of device services of an EdgeX instance.

* **GET** commands are issued to a device or sensor to get a current value for a particular attribute on the device, such as the current temperature provided by a thermostat sensor, or the on/off status of a light.
* **PUT** commands are issued to a device or sensor to change the current state or status of a device or one of its attributes, such as setting the speed in RPMs of a motor, or setting the brightness of a dimmer light.

::

    commands:
        - name: "Random"
          get:
            path: "/api/v1/device/{deviceId}/Random"
            responses:
                - code: "200"
                  description: ""
                  expectedValues: ["randomnumber"]
                - code: "503"
                  description: "service unavailable"
                  expectedValues: []

A command is represented by a `name` and the `path` which can be used to call it.

Note, a command `name` needs to match a device resource or resource. It's also unclear why `path` is required (especially with an endpoint version included). I think this could be automatically generated?

Additionally, a **PUT** command must declare a `parameters` property of what values can be passed into the command from the caller. This is a comma-separated list of `deviceResource` definitions from the previous sections.

Every command must declare what `responses` might be returned in response to it being called. Usually this means a success response (code 200 in our example) and one or more error responses (code 503 in our example). Here we're using HTTP response codes, but you are not required to follow that convention.

Every response must declare its `expectedValues`, meaning what data the response will contain in addition to the response code itself. This is a comma-separated list of `deviceResource` definitions from the previous sections.

**TODO**: What is {deviceId} in the `path` property? What other variables are available?

This is the actual REST endpoint path that Core Command would call. {deviceId} is parameter which is replaced by an actual device `Id` in the actual REST call.
