---
order: 1
---

# Creating a general service in C++

Broadly speaking, development of a new service in any language requires considering:
- What types of messages it subscribes to
- What computations it should do when it receives a subscribed message
- What messages (if any) it should publish in response
- Where it should save files (if any)

Since OpenUxAS is written in C++, you'll generally want to compile a new service written in C++ as part of the main executable. When you run the OpenUxAS executable, you specify in an XML configuration file 1) which services you want to load and 2) values for any service-specific parameters. This XML configuration file is passed to the UxAS executable as a command line argument. This means that for services written in C++ and compiled as part of the main executable, you also need to consider:
- What configurable parameters the service has
- What the format of its XML configuration element


## Basic steps

OpenUxAS includes a template header file `00_ServiceTemplate.h` and source file `00_ServiceTemplate.cpp` for general services in C++. If `{UxAS_ROOT}` is your main OpenUxAS directory, then these files are located in directory `{UXAS_ROOT}/src/cpp/Services`.

In general, creating a new non-task service consists of the following steps:

1. Choose a name `{ServiceName}` for your service
2. In directory `{UxAS_ROOT}/src/cpp/Services`, copy `00_ServiceTemplate.h` to `{ServiceName}.h` and `00_ServiceTemplate.cpp` to `{ServiceName}.cpp`
3. In both `{ServiceName}.h` and `{ServiceName}.cpp`, 
   1. Change all instances of the string `00_ServiceTemplate` to `{ServiceName}`
   2. Refine the `namespace` (if desired)
4. In header file `{ServiceName}.h`:
   1. Change the header guard string `UXAS_00_SERVICE_TEMPLATE_H` to a unique string appropriate for the new service, e.g. `UXAS_{SERVICE_NAME}_H`
   2. If you expect the service to generate output files, update function `s_directoryName()` to return an appropriate directory name
   3. Add `#include` directives for any necessary header files
   4. Declare any necessary private member data and functions
   5. Update the doxygen comments
5. In source file `{ServiceName}.cpp`:
   1. Update the logic of the member function `configure`
   2. Add `#include` directives for any published or subscribed messages
   3. Add `#include` directives for any other necessary header files
   4. Update the logic for member function `processLmcpMessage` to handle different types of subscribed message types
   5. Define any necessary private member functions 
6. Modify the build process to include any 3rd party dependencies
7. Create and run an example to exercise the service

## An example C++ service

Suppose we want to create a service called `JsonAirVehicleLoggerService` that receives `AirVehicleState` messages, converts portions of them to JSON format, and saves them to a file. More specifically, we'd like to specify 1) the name of the output file, 2) the IDs of air vehicles whose messages should be converted and saved, 3) the level of message content per vehicle to be converted and saved. We'd also like the service to publish a `KeyValuePair` message that reports the value of the ID of each vehicle the first time it is seen.

We decide that the XML configuration element for our service should look something like this:
```xml
<Service Type="JsonAirVehicleLoggerService" Filename="myOutputFile.txt">
   <AirVehicle VehicleID="10" Level="0"/>
   <AirVehicle VehicleID="11" Level="1"/>
   <AirVehicle VehicleID="12" Level="2"/>
 </Service>
```

We also decide that the different reporting levels should save the following message fields:

| Level   | Fields   |
| ------- | -------- |
| 0	      | ID, Time, Latitude, Longitude |
| 1 	  | ID, Time, Latitude, Longitude, Heading, Groundspeed, Airspeed, VerticalSpeed |
| 2+      | ID, Time, Latitude, Longitude, Heading, Groundspeed, Airspeed, VerticalSpeed, EnergyAvailable, ActualEnergyRate |

## Creating the example C++ service

After copying the template file and going through the basic steps above, we end up with the following header file

```cpp
/* 
 * File:   JsonAirVehicleLoggerService.h
 * Author: lhumphrey
 *
 * Created on Mar 12, 2021
 */

#ifndef UXAS_JSON_AIR_VEHICLE_LOGGER_SERVICE_H
#define UXAS_JSON_AIR_VEHICLE_LOGGER_SERVICE_H

#include <nlohmann/json.hpp>

#include "ServiceBase.h"
#include "CallbackTimer.h"
#include "TypeDefs/UxAS_TypeDefs_Timer.h"

#include <cstdint>
#include <fstream>

namespace uxas
{
namespace service
{
namespace data
{

/*! \class JsonAirVehicleLoggerService
    \brief This service saves fields of AirVehicleState messages in JSON format.

 * For each ID-specified air vehicle, save certain fields of corresponding
 * AirVehicleState messages depending on reporting level: 0-, 1, or 2+.
 * 
 * Example Configuration Element: 
 * <Service Type="JsonAirVehicleLoggerService" Filename="myOutput.txt">
 *   <AirVehicle ID="3" Level="0"/>
 *   <AirVehicle ID="4" Level="0"/>
 *   <AirVehicle ID="5" Level="1"/>
 *   <AirVehicle ID="6" Level="2"/>
 * </Service>
 *   
 * Options:
 *  - Filename - Name of the file in which to log results
 *  - - ID - The ID number of the air vehicle
 *  - - Level - The level of information to report: "0", "1", or "2"
 * 
 * Subscribed Messages:
 *  - afrl::cmasi::AirVehicleState
 * 
 * Sent Messages:
 *  - afrl::cmasi::KeyValuePair
 * 
 */

class JsonAirVehicleLoggerService : public ServiceBase
{
public:

    static const std::string&
    s_typeName()
    {
        static std::string s_string("JsonAirVehicleLoggerService");
        return (s_string);
    };

    static const std::vector<std::string>
    s_registryServiceTypeNames()
    {
        std::vector<std::string> registryServiceTypeNames = {s_typeName()};
        return (registryServiceTypeNames);
    };

    // Define name of directory in which output file will be stored
    static const std::string&
    s_directoryName() { static std::string s_string("JsonAirVehicleLogs"); return (s_string); };

    static ServiceBase*
    create()
    {
        return new JsonAirVehicleLoggerService;
    };

    JsonAirVehicleLoggerService();

    virtual
    ~JsonAirVehicleLoggerService();

private:

    static
    ServiceBase::CreationRegistrar<JsonAirVehicleLoggerService> s_registrar;

    /** brief Copy construction not permitted */
    JsonAirVehicleLoggerService(JsonAirVehicleLoggerService const&) = delete;

    /** brief Copy assignment operation not permitted */
    void operator=(JsonAirVehicleLoggerService const&) = delete;

    bool
    configure(const pugi::xml_node& serviceXmlNode) override;

    bool
    initialize() override;

    bool
    start() override;

    bool
    terminate() override;

    bool
    processReceivedLmcpMessage(std::unique_ptr<uxas::communications::data::LmcpMessage> receivedLmcpMessage) override;


private:

    // Member variables to store output filename, a map of ID numbers to report Level based
    // on sevice configuration element, and the set of ID numbers seen during execution
    std::string m_outputFilename = std::string("output.txt");
    std::unordered_map<int64_t, int64_t> m_idToLevel;
    std::unordered_set<int64_t> m_seenIds;
};

}; //namespace data
}; //namespace service
}; //namespace uxas

#endif /* UXAS_JSON_AIR_VEHICLE_LOGGER_SERVICE_H */

```

and the following source file
```cpp
/* 
 * File:   JsonAirVehicleLoggerService.cpp
 * Author: lhumphrey
 *
 */

#include "JsonAirVehicleLoggerService.h"

#include "afrl/cmasi/KeyValuePair.h"
#include "afrl/cmasi/AirVehicleState.h"

#include <iostream>
#include <fstream>
#include <string>

#define STRING_XML_FILENAME "Filename"
#define STRING_XML_VEHICLE "AirVehicle"
#define STRING_XML_ID "ID"
#define STRING_XML_LEVEL "Level"

namespace uxas
{
namespace service
{
namespace data
{

JsonAirVehicleLoggerService::ServiceBase::CreationRegistrar<JsonAirVehicleLoggerService>
JsonAirVehicleLoggerService::s_registrar(JsonAirVehicleLoggerService::s_registryServiceTypeNames());

JsonAirVehicleLoggerService::JsonAirVehicleLoggerService()
: ServiceBase(JsonAirVehicleLoggerService::s_typeName(), JsonAirVehicleLoggerService::s_directoryName()) { };

JsonAirVehicleLoggerService::~JsonAirVehicleLoggerService() { };

bool JsonAirVehicleLoggerService::configure(const pugi::xml_node& serviceXmlNode)
{
    bool isSuccess(true);

    // Get and save the Filename attribute (if specified)
    if (!serviceXmlNode.attribute(STRING_XML_FILENAME).empty())
    {
        m_outputFilename = serviceXmlNode.attribute(STRING_XML_FILENAME).value();
    }

    // Get and save the ID and Level attributes for each AirVehicle
    for (pugi::xml_node subXmlNode = serviceXmlNode.first_child(); subXmlNode; subXmlNode = subXmlNode.next_sibling())
    {
        if (std::string(STRING_XML_VEHICLE) == subXmlNode.name())
        {   
            if(!subXmlNode.attribute(STRING_XML_ID).empty() && !subXmlNode.attribute(STRING_XML_LEVEL).empty()) 
            {
                int64_t vId = subXmlNode.attribute(STRING_XML_ID).as_int();
                int64_t reportLevel = subXmlNode.attribute(STRING_XML_LEVEL).as_int();
                m_idToLevel[vId] = reportLevel;
            }
            else
            {
                UXAS_LOG_ERROR(s_typeName(), "::configure encountered ", STRING_XML_VEHICLE, 
                " element with missing attribute ", STRING_XML_ID, " or ", STRING_XML_LEVEL);
                isSuccess = false;
            }
        }
        else
        {
            UXAS_LOG_ERROR(s_typeName(), "::configure encountered unknown element ", subXmlNode.name());
            isSuccess = false;
        }
    }

    addSubscriptionAddress(afrl::cmasi::AirVehicleState::Subscription);

    return (isSuccess);
}

bool JsonAirVehicleLoggerService::initialize()
{
    std::cout << "*** INITIALIZING:: Service[" << s_typeName() << "] Service Id[" << m_serviceId 
              << "] with working directory [" << m_workDirectoryName << "] *** " << std::endl;
    
    return (true);
}

bool JsonAirVehicleLoggerService::start()
{
    std::cout << "*** STARTING:: Service[" << s_typeName() << "] Service Id[" << m_serviceId 
              << "] with working directory [" << m_workDirectoryName << "] *** " << std::endl;
    
    // Remove any existing contents of the output file on service start
    std::ofstream file(m_workDirectoryPath + m_outputFilename);
    file << "";
    file.close();

    return (true);
};

bool JsonAirVehicleLoggerService::terminate()
{
    std::cout << "*** TERMINATING:: Service[" << s_typeName() << "] Service Id[" << m_serviceId 
              << "] with working directory [" << m_workDirectoryName << "] *** " << std::endl;
    
    return (true);
}

bool JsonAirVehicleLoggerService::
processReceivedLmcpMessage(std::unique_ptr<uxas::communications::data::LmcpMessage> receivedLmcpMessage)
{
    if (afrl::cmasi::isAirVehicleState(receivedLmcpMessage->m_object))
    {
        auto avState = std::static_pointer_cast<afrl::cmasi::AirVehicleState> (receivedLmcpMessage->m_object);

        // If the ID of an AirVehicleState message corresponds to a configured ID and
        // has not been seen before, store the ID and publish a corresponding KeyValuePair
        if (m_idToLevel.count(avState->getID()) && !m_seenIds.count(avState->getID()))
        {
            m_seenIds.insert(avState->getID());
            auto kvp = std::make_shared<afrl::cmasi::KeyValuePair>();
            kvp->setKey(std::string("JsonAirVehicleLoggerVehicleID"));
            kvp->setValue(std::to_string(avState->getID()));
            sendSharedLmcpObjectBroadcastMessage(kvp);
        }

        // If the ID was listed in the configuration element, save a JSON message 
        // with content according to the configured Level for that ID
        if (m_idToLevel.count(avState->getID())) 
        {
            nlohmann::json j;
            j["ID"] = avState->getID();
            j["Time"] = avState->getTime();
            j["Latitude"] = avState->getLocation()->getLatitude();
            j["Longitude"] = avState->getLocation()->getLongitude();

            if (m_idToLevel[avState->getID()] > 0) {
                j["Heading"] = avState->getHeading();
                j["Groundspeed"] = avState->getGroundspeed();
                j["Airspeed"] = avState->getAirspeed();
                j["VerticalSpeed"] = avState->getVerticalSpeed();
            }

            if (m_idToLevel[avState->getID()] > 1) {
                j["EnergyAvailable"] = avState->getEnergyAvailable();
                j["ActualEnergyRate"] = avState->getActualEnergyRate();
            }

            std::ofstream file(m_workDirectoryPath + m_outputFilename, std::ios::app);
            file << j.dump() << std::endl;
            file.close();
        }
        
    }
    return false;
}

}; //namespace data
}; //namespace service
}; //namespace uxas
```

To write our service, we follow the steps above. Some things to take note of:

- For steps 1-3 we:
   - Replace `00_ServiceTemplate` with `JsonAirVehicleLoggerService`
   - Refine the namespace to `uxas::service::data`
- For step 4 we do the following in the header file:
   - Change the header guard string `UXAS_00_SERVICE_TEMPLATE_H` to `UXAS_JSON_AIR_VEHICLE_LOGGER_SERVICE_H`
   - Update function `s_directoryName()` to return `"JsonAirVehicleLogs"`
   - Include a header `nlohmann/json.hpp` for a 3rd party dependency for working with JSON
   - Declare private member variables `m_outputFilename` (with default value `"output.txt"`), `m_idToLevelMap`, and `m_seenVehicleSet` that will be used in the source file
   - Update the main doxygen-compatible comment block at the top, paying particular attention to the example configuration element, its options, and the lists of subscribed and sent messages
- For step 5 we do the following in the source file:
   - Include headers for necessary messages: `"afrl/cmasi/KeyValuePair.h"` and `"afrl/cmasi/AirVehicleState.h"`
   - Include headers `<fstream>` and `<string>`
   - Update the logic of function `configure` to process an XML configuration element of the form given in the main comment block of the header
      - (Optional convention) Define macros for configuration element and attribute strings 
      - Save configuration parameters to `m_outputFilename` (if applicable) and `m_idToLevel`
      - Log errors and set `isSuccess = False` if the configuration element is invalid
   - Update the logic of function `processReceivedLmcpMessage` to process `AirVehicleState` messages
      - Output a `KeyValuePair` of the form (`JsonAirVehicleLoggerVehicleID`,`{ID}`) the first time an `ID` number corresponding to an `AirVehicle` configuration element is seen
      - Construct and save a JSON-formatted message to the configured output file with content depending on the `Level` of the `AirVehicle` configuration element with the corresponding `ID`


## Compiling the example C++ service
(TBD)


## Running the example C++ service

By convention, OpenUxAS examples are placed in subdirectories within `{UXAS_ROOT}/examples`. There is a template directory `{UXAS_ROOT}/examples/00_Template` that contains the files needed to bootstrap many standard examples. Once you have recompiled OpenUxAS with the `JsonAirVehicleLoggerService`, you can run this service as part of an example by performing the following steps within directory `{UXAS_ROOT}/examples`
1. Copy subdirectory `00_Template` to a new subdirectory `{My_Example}`
2. Edit OpenUxAS configuration file `{MyExample}/cfg_Template.xml` to include an XML configuration element for the service (the one in the header file's main comment block will work)
3. Run OpenUxAS with the modified configuration file and in conjunction with OpenAMASE in one of several ways
   - With the `run-example` script from directory `{UXAS_ROOT}` using command: `./run-examples {My_Example}`
   - With shell scripts from directory `{UXAS_ROOT}/examples/{My_Example}` using command: `./runAMASE_Template.sh & ./runUxAS_Template.sh`
4. Press the Play button in the OpenAMASE simulation pane
   - In the OpenAMASE simulation, vehicles should start to move
   - In the terminal, the OpenUxAS binary should output text, including "INITIALIZING" and "STARTING" messages from `JsonAirVehicleLoggerService`
5. After the simulation has run for at least a few seconds, 
   - If you used the `run-example` script, close the OpenAMASE GUI and the OpenUxAS process will terminate automatically
   - If you used the shell scripts, close the OpenAMASE GUI and also kill the OpenUxAS process by pressing Ctrl+C in the terminal window
6. Examine the contents of `{UXAS_ROOT}/examples/{My_Example}/RUNDIR_Template/datawork`
   - Subdirectory `JsonAirVehicleLogs` should contain an output file with JSON-formatted messages
   - Subdirectory `SavedMessages` should contain a SQL file `messageLog_1_0.db3` that contains the `KeyValuePair` messages published by `JsonAirVehicleLoggerService`    

  
