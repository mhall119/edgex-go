#############################
Understanding Device Profiles
#############################

=========================
What is a Device Profile?
=========================

A Device Profile describes the capabilities of a model or class of devices in a way that EdgeX services understand. A typical profile will define the datatypes that the device uses, the values that the device can povide, and commands that can be called against the device for reading values or triggering actions. Every Device that EdgeX is made aware of must be associated with a Device Profile. It's possible, even likely, that you will have more than one Device deployed that uses the same Profile.

A Device Profile can be added to EdgeX using the Metadata APIs directly, but they are also commonly added automatically by a Device Service if that service is designed to handle only a specific set of devices. For example, the GrovePi Device Service used by the Community Devkit is designed to work only with the GrovePi sensor devices, and so it provides a Device Profile that will be automatically installed when that device service is started. Conversely, the MQTT Device Service can handle any type of device that uses MQTT, and so you would need to add your device profiles yourself when using this device service.

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

Keep in mind that these datatypes can be used for different meanings, and so should be kept generic. For example, one `Temperature` type definition could be used for both `Water Temperature` and `Air Temperature` readings, as well as `Thermostat Setting` values.

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

In our example Device Profile we deal with only one data type, the `randomnumber`. It is defined as a 32bit integer with a minimum values of 0 and a maximum of 100, defaulting to 0. We also must specify a unit for this data, which would be `Degrees Celcius` for a Temperature, but in our case the `randomnumber` has no unit, so we can just let it default to an empty string.

Like the `name` property in the profile summary, the `name` of the deviceResource is also used as an identifier both within EdgeX and elsewhere in the Device Profile.

**TODO**: What are `attributes` and when do they get used?

**TODO**: When/where are units used?

**TODO**: Why do these have `readWrite` properties? What does that do?

---------
Resources
---------

A Resource is a data value with context. It combines the datatypes defined in the `deviceResources` section above with a meaning behind it. Here is where you would define things like `Water Temperature` and `Air Temperature` readings, as well as writable values.

::

    resources:
        - name: "Random"
          get:
            - { operation: "get", object: "randomnumber", property: "value", parameter: "Random" }

Once again our example profile is quite simple, defining only one resource named `Random` which is the random number that our simulated device will produce. You can see how it is defined as being of the `randomnumber` datatype that we previously defined.

A Resource can have either a `get` or a `set` operation (or both) depending on your device capabilities.

**TODO**: When to use `object` and when to use `resource`

**TODO**: What does `property` do and what values can it take?

**TODO**: What does 'parameter' do?

--------
Commands
--------

Commands are how EdgeX calls your device (via the Device Service) to retrieve data, change settings or trigger actions. You can define multiple commands, and each command can have either a `get` or a `put` function (or both). Note that here we use `put` instead of `set` as above, because a `put` operation doesn't necessarily result in setting a Resource value.

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

Additionally, a **PUT** command must declare a `parameters` property of what values can be passed into the command from the caller. This is a comma-separated list of `deviceResource` definitions from the previous sections.

Every command must declare what `responses` might be returned in response to it being called. Usually this means a success response (code 200 in our example) and one or more error responses (code 503 in our example). Here we're using HTTP response codes, but you are not required to follow that convention.

Every response must declare its `expectedValues`, meaning what data the response will contain in addition to the response code itself. This is a comma-separated list of `deviceResource` definitions from the previous sections.

**TODO**: What is {deviceId} in the `path` property? What other variables are available?