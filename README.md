# cpp-pcp-client

cpp-pcp-client is a C++ client library for the [PCP protocol][specs]. It includes a
collection of abstractions which can be used to initiate connections to a PCP
broker, wrapping the PCP message format and performing schema validation for
message bodies.

cpp-pcp-client implements PCP by layering it upon WebSocket; it uses
[WebSocket++][websocket++] for that.

A tutorial on how to create a PCP agent / controller pair with cpp-pcp-client is
[here][tutorial].

Dependencies
------------

 - a C++11 compiler (clang/gcc 4.7)
 - CMake (3.2 or newer)
 - Boost (1.54 or newer)
 - OpenSSL
 - [leatherman][leatherman], installed as a standalone library (0.3.5 or newer)

Initial Setup
-------------

#### Setup on Fedora 23

The following will install all required tools and libraries:

    yum install boost-devel openssl-devel gcc-c++ make wget tar cmake

#### Setup on Mac OSX El Capitan (homebrew)

This assumes Clang is installed and the system OpenSSL libraries will be used.

The following will install all required libraries:

    brew install cmake boost

#### Setup on Ubuntu 15.10 (Trusty)

The following will install most required tools and libraries:

    apt-get install build-essential libboost-all-dev libssl-dev wget tar cmake

#### Setup on Windows

[MinGW-w64][MinGW-w64] is used for full C++11 support, and
[Chocolatey][Chocolatey] can be used to install. You should have at least 2GB of
memory for compilation.

* install [CMake][CMake-choco] & [7zip][7zip-choco]

        choco install cmake 7zip.commandline

* install [MinGW-w64][MinGW-w64-choco]

        choco install mingw --params "/threads:win32"

For the remaining tasks, build commands can be executed in the shell from

    Start > MinGW-w64 project > Run Terminal

* select an install location for dependencies, such as C:\\tools or cmake\\release\\ext;
we'll refer to it as $install

* build [Boost][Boost-download]

        .\bootstrap mingw
        .\b2 toolset=gcc --build-type=minimal install --prefix=$install --with-program_options --with-system --with-filesystem --with-date_time --with-thread --with-regex --with-log --with-locale --with-chrono boost.locale.iconv=off

In Powershell:

    choco install cmake 7zip.commandline -y
    choco install mingw --params "/threads:win32" -y
    $env:PATH = "C:\tools\mingw64\bin;$env:PATH"
    $install = "C:\tools"

    (New-Object Net.WebClient).DownloadFile("https://downloads.sourceforge.net/boost/boost_1_54_0.7z", "$pwd/boost_1_54_0.7z")
    7za x boost_1_54_0.7z
    pushd boost_1_54_0
    .\bootstrap mingw
    .\b2 toolset=gcc --build-type=minimal install --prefix=$install --with-program_options --with-system --with-filesystem --with-date_time --with-thread --with-regex --with-log --with-locale --with-chrono boost.locale.iconv=off
    popd

Build and install
-----------------

* build & install [leatherman][leatherman]

* build and install cpp-pcp-client

  Thanks to the CMake, the project can be built out-of-source tree, which allows
  for multiple independent builds.

  example release build:

      mkdir release
      cd release
      cmake ..
      make
      sudo make install

  example debug/test build:

      mkdir debug
      cd debug
      cmake -DCMAKE_BUILD_TYPE=Debug ..
      make
      sudo make install

Usage
-----

##### Table of Contents
- [Important Data Structures](#data_structures)
- [Creating Connections](#connections)
- [Message Schemas and Callbacks](#receiving_messages)
- [Sending Messages](#sending_messages)
- [Data Validation](#validation)

<a name="data_structures"/>
###Important Data Structures

Before we start to look at creating connections and sending/receiving messages, it
is important to look at some of the data structures used by cpp-pcp-client.

__leatherman::json_container::JsonContainer__

The [JsonContainer][json_container] class is used frequently by the
cpp-pcp-client library as a simplified abstraction around complex JSON
c++ libraries.

__ParsedChunks__

The _Parsed_Chunks_ struct is a simplification of a parsed PCP message. It allows
for direct access of a message's Envelope, Data and Debug chunks as JsonContainer
or string objects.

The ParsedChunks struct is defined as:

```
    struct ParsedChunks {
        // Envelope
        JsonContainer envelope;

        // Data
        bool got_data;
        ContentType data_type;
        JsonContainer data;
        std::string binary_data;

        // Debug
        std::vector<JsonContainer> debug;
    }
```

<a name="connections"/>
###Creating Connections

The first step to interacting with a PCP broker is creating a connection. To
achieve this we must first create an instance of the Connector object.

The constructor of the Connector class is defined as:

```
    Connector(const std::string& broker_ws_uri,
              const std::string& client_type,
              const std::string& ca_crt_path,
              const std::string& client_crt_path,
              const std::string& client_key_path,
              const long& connection_timeout = 5000)
```

The parameters are described as:

 - client_type - A free form value used to group applications connected to a PCP broker.
 For example, an applications connected with the type _potato_ will all be addressable
 as the PCP endpoint pcp://*/potato. The only value that you cannot use is "server",
 which is reserved for PCP brokers (please refer to the URI section in the
 [PCP specifications][specs]).
 - broker_ws_uri - The WebSocket URL of the PCP broker. For example, _wss://localhost:8142/pcp/_.
 - ca_crt_path - The path to your CA certificate file.
 - client_crt_path - The path to a client certificate file generated by your CA.
 - client_key_path - The path to a client public key file generated by you CA.
 - connection_timeout - The maximum amount of time (in miliseconds) it can take
   for a connection to be established. Defaults to 5000 ms.

This means that you can instantiate a Connector object as follows:

```
    Connector connector { "wss://localhost:8142/pcp/", "controller",
                          "/etc/puppet/ssl/ca/ca_crt.pem",
                          "/etc/puppet/ssl/certs/client_crt.pem",
                          "/etc/puppet/ssl/public_keys/client_key.pem",
                          5000 };
```

When you have created a Connector object you are ready to connect. The Connector's
connect method is defined as:

```
    void connect(int max_connect_attempts = 0) throws (connection_config_error,
                                                       connection_fatal_error)
```

The parameters are described as:

 - max_connect_attempts - The amount of times the Connector will try and establish
 a connection to the PCP broker if a problem occurs. It will try to connect
 indefinately when set to 0. Defaults to 0.

The connect method can throw the following exceptions:

```
    class connection_config_error : public connection_error
```

This exception will be thrown if a Connector is misconfigured. Misconfiguration
includes specifying an invalid broker WebSocket URL or a path to a file that
doesn't exist. Note that if this exception has been thrown no attempt at
creating a network connection has yet been made.

```
    class connection_fatal_error : public connection_error
```

This exception wil be thrown if the connection cannot be established after the
Connector has tried _max_connect_attempts_ times.

A connection can be established as follows:

```
    try {
        connector.connect(5);
    } catch (connection_config_error e) {
        ...
    } catch (connection_fatal_error) {
        ...
    }
```

If no exceptions are thrown it means that a connection has been sucessfuly
established. You can check on the status of a connection with the Connector's
isConnected method.

The isConnected method is defined as:

```
    bool isConnected()
```

And it can be used as follows:

```
    if (connector.isConnected()) {
        ...
    } else {
        ...
    }
```

By default a connection is non persistent. For instance, in case WebSocket is
used as the underlying transport layer, ping messages must be sent periodically
to keep the connection alive. Also, the connection may drop due to communication
errors.  You can enable connection persistence by calling the _monitorConnection_
method that will periodically check the state of the underlying connection. It
will send keepalive messages to the broker and attempt to re-establish the
connection in case it has been dropped.

_monitorConnection_ is defined as:

```
    void monitorConnection(int max_connect_attempts = 0)
```

The parameters are described as:

- max_connect_attemps -The number of times the Connector will try to reconnect
 a connection to the PCP broker if a problem occurs. It will try to connect
 indefinately when set to 0. Defaults to 0.

Note that if the Connector fails to re-establish the connection after the
specified number of attempts, a _connection_fatal_error_ will be thrown.
Also, calling _monitorConnection_ will block the execution thread as the
monitoring task will not be executed on a separate thread. On the other hand,
the caller can safely execute _monitorConnection_ on a separate thread since
the function returns once the _Connector_ destructor is invoked.

```
    connector.monitorConnection(5);
```

<a name="receiving_messages"/>
### Message Schemas and Callbacks

Every message sent over the PCP broker has to specify a value for the _data_schema_
field in the message envelope. These data schema's determine how a message's data
section is validated. To process messages received from a PCP broker you must
first create a schema object for a specific _data_schema_ value.

The constructor for the Schema class is defined as:

```
    Schema(const std::string& name, ContentType content_type)
```

The parameters are described as:

 - name - The name of the schema. This should be the same as the value found in
 a message's data_schema field.
 - content_type - Defines the content type of the schema. Valid options are
 ContentType::Binary and ContentType::Json

A Schema object can be created as follows:

```
    Schema cnc_request_schema { "cnc_request", ContentType::Json};
```

You can now start to add constraints to the Schema. Consider the following JSON-schema:

```
  {
    "title": "cnc_request",
    "type": "object",
    "properties": {
      "module": {
        "type": "string"
      },
      "action": {
        "type": "string"
      },
    },
    "required": ["module"]
  }
```
You can reproduce its constraints by using the addConstraint method which
is defined as follows:

```
    void addConstraint(std::string field, TypeConstraint type, bool required)
```

The parameters are described as follows:

 - field - The name of the field you wish to add the constraint to.
 - type - The type constraint to put on the field. Valid types are TypeConstraint::Bool,
 TypeConstraint::Int, TypeConstraint::Bool, TypeConstraint::Double, TypeConstraint::Array,
 TypeConstraint::Object, TypeConstraint::Null and TypeConstraint::Any
 - required - Specify whether the field is required to be present or not. If not specified
 it will default to false.

```
    cnc_request_schema.addConstraint("module", TypeConstraint::String, true);
    cnc_request_schema.addConstraint("action", TypeConstraint::String);
```

With a schema defined we can now start to process messages of that type by registering
message specific callbacks. This is done with the Connector's registerMessageCallback
method which is defined as follows:

```
    void registerMessageCallback(const Schema schema, MessageCallback callback)
```

The parameters are described as follows:

 - schema - A previously created schema object
 - callback - A callback function with the signature void(const ParsedChunks& msg_content)

For example:

```
    void cnc_requestCallback(const ParsedChunks& msg_content) {
      std::cout << "Message envelope: " << msg_content.envelope.toString() << std::endl;

      if (msg_content.has_data()) {
        if (msg_content.data_type == ContentType::Json) {
          std::cout << "Content Type: JSON" << std::endl;
          std::cout << msg_content.data.toString() << std::endl;
        } else {
          std::cout << "Content Type: Binary" << std::endl;
          std::cout << msg_content.binary_data << std::endl;
        }
      }

      for (const auto& debug_chunk : msg_content) {
        std::cout << "Data Chunk: " << debug_chunk << std::endl;
      }
    }

    ...
    connector.registerMessageCallback(cnc_request_schema, cnc_requestCallback);
```

Now that the callback has been regsitered, every time a message is received where
the data_schema field is _cnc_request_, the content of the Data chunk will be
validated against the schema and if it passes, the above defined function will be called.
If a message is received which doesn't have a registered data_schema the message
will be ignored.

Using this method of registering schema/callback pairs we can handle each message
in a unique manner,

```
    connector.registerMessageCallback(cnc_request_schema, cnc_requestCallback);
    connector.registerMessageCallback(puppet_request_schema, puppet_requestCallback);
    connector.registerMessageCallback(puppet_db_request_schema, puppet_db_requestCallback);
```

or you can assign one callback to a lot of different schemas,

```
    connector.registerMessageCallback(schema_1, genericCallback);
    ...
    connector.registerMessageCallback(schema_n, genericCallback);
    connector.registerMessageCallback(schema_n1, genericCallback);
```

<a name="sending_messages"/>
### Sending Messages

Once you have established a connection to the PCP broker you can send messages
using the _send_ method. There are two main overloads for this function that
are defined as (please check connector.hpp for the other overloads):

```
    void send(std::vector<std::string> targets,
              std::string data_schema,
              unsigned int timeout,
              JsonContainer data_json,
              std::vector<JsonContainer> debug = std::vector<JsonContainer> {})
                        throws (connection_processing_error, connection_not_init_error)
```

With the parameters are described as follows:

 - targets - A vector of the destinations the message will be sent to
 - data_schema - The Schema that identifies the message type
 - timeout - Duration the message will be valid on the fabric, in seconds
 - data_json - A JsonContainer representing the data chunk of the message
 - debug - A vector of strings representing the debug chunks of the message (defaults to empty)


```
    std::string send(std::vector<std::string> targets,
                     std::string data_schema,
                     unsigned int timeout,
                     bool destination_report,
                     std::string data_binary,
                     std::vector<JsonContainer> debug = std::vector<JsonContainer> {})
                              throws (connection_processing_error, connection_not_init_error)

```

The above overload returns the ID of the sent message and accepts the following
parameters:

 - targets - A vector of the destinations the message will be sent to
 - data_schema - The Schema that identifies the message type
 - timeout - Duration the message will be valid on the fabric, in seconds
 - destination_report - A boolean indicating whether or not requesting a destination report
 - data_binary - A string representing the data chunk of the message
 - debug - A vector of strings representing the debug chunks of the message (defaults to empty)

The _send_ methods can throw the following exceptions:

```
    class connection_processing_error : public connection_error
```

This exception is thrown when an error occurs during at the underlying WebSocket
layer.

```
    class connection_not_init_error : public connection_error
```

This exception is thrown when trying to send a message when there is no active
connection to the broker.

Example usage:

```
    JsonContainer data {};
    data.set<std::string>("foo", "bar");
    try {
      auto id = connector.send({"pcp://*/potato"}, "potato_schema", 42, data);
      std::cout << "Sent potato message to all potato clients, with ID=" << id << std::endl;
    } catch (connection_not_init_error e) {
      std::cout << "Cannot send message without being connected to the broker" << std::endl;
    } catch (connection_processing_error e) {
      std::cout << "An error occured at the WebSocket layer: " << e.what() << std::endl;
   }
```

<a name="validation"/>
### Data Validation

As mentioned in the [Message Schemas and Callbacks](#receiving_messages), messages
received from the PCP broker will be matched against a Schema that you defined.
The Connector object achieves this functionality by using an instance of the Validator
class. It is possible to instantiate your own instance of the Validator class and
use schema's to validate other, non message, data structures.

The Validator is limited to a no-args constructor:

```
    Validator()
```

You can register a Schema by using the _registerSchema_ method, defined as:

```
    void registerSchema(const Schema& schema) throws (schema_redefinition_error)
```

The parameters are described as follows:

 - schema - A schema object that desribes a set of constraints.

When a Schema has been registered you can use the _validate_ method to validate
a JsonContainer object. The _validate_ method is defined as follows:

```
    void validate(JsonContainer& data, std::string schema_name) const throws (validation_error)
```

The parameters are described as follows:

 - data - A JsonContainer you want to validate.
 - schema_name - The name of the schema you want to validate against.

Example usage:

```
    Validator validator {};
    Schema s {"test-schema", ContentType::Json };
    s.addConstraint("foo", TypeConstraint::Int);
    validator.registerSchema(s);

    JsonContainer d {};
    d.set<int>("foo", 42);

    try {
      Validator.validate(d, "test-schema");
    } catch (validation_error) {
      std::cout << "Validation failed" << std::endl;
    }
```

[tutorial]: https://github.com/puppetlabs/cpp-pcp-client/tree/master/tutorial
[specs]: https://github.com/puppetlabs/pcp-specifications
[json_container]: https://github.com/puppetlabs/leatherman/tree/master/json_container
[websocket++]: http://www.zaphoyd.com/websocketpp/
[leatherman]: https://github.com/puppetlabs/leatherman
[MinGW-w64]: http://mingw-w64.sourceforge.net/
[Chocolatey]: https://chocolatey.org
[CMake-choco]: https://chocolatey.org/packages/cmake
[7zip-choco]: https://chocolatey.org/packages/7zip.commandline
[MinGW-w64-choco]: https://chocolatey.org/packages/mingw
[Boost-download]: http://sourceforge.net/projects/boost/files/latest/download
