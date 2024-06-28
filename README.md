[![FIWARE Banner](https://fiware.github.io/tutorials.IoT-over-MQTT/img/fiware.png)](https://www.fiware.org/developers)

[![FIWARE IoT Agents](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/iot-agents.svg)](https://github.com/FIWARE/catalogue/blob/master/iot-agents/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Iot-Agent.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://nexus.lab.fiware.org/repository/raw/public/badges/stackoverflow/fiware.svg)](https://stackoverflow.com/questions/tagged/fiware)

This tutorial a wires up the dummy IoT devices which are responding using a custom [XML](https://www.w3.org/TR/xml11/)
message format. A **custom IoT Agent** is created based on the IoT Agent Node.js
[library](https://iotagent-node-lib.readthedocs.io/en/latest/) and the framework found in the
[IoT Agent for Ultralight](https://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual)
devices so that measurements can be read and commands can be sent using
[NGSI-v2](https://fiware.github.io/specifications/OpenAPI/ngsiv2) requests sent to the
[Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/).

The tutorial uses [cUrl](https://ec.haxx.se/) commands throughout, but is also available as
[Postman documentation](https://www.postman.com/downloads/).

# Start-Up

## NGSI-v2 Smart Supermarket

**NGSI-v2** offers JSON based interoperability used in individual Smart Systems. To run this tutorial with **NGSI-v2**, use the `NGSI-v2` branch.

```console
git clone https://github.com/FIWARE/tutorials.Custom-IoT-Agent.git
cd tutorials.Custom-IoT-Agent
git checkout NGSI-v2

./services create
./services start
```

| [![NGSI v2](https://img.shields.io/badge/NGSI-v2-5dc0cf.svg)](https://fiware-ges.github.io/orion/api/v2/stable/) | :books: [Documentation](https://github.com/FIWARE/tutorials.Custom-IoT-Agent/tree/NGSI-v2) |  <img src="https://cdn.jsdelivr.net/npm/simple-icons@v3/icons/postman.svg" height="15" width="15"> [Postman Collection](https://fiware.github.io/tutorials.Custom-IoT-Agent/) |  ![](https://img.shields.io/github/last-commit/fiware/tutorials.Custom-IoT-Agent/NGSI-v2)
| --- | --- | --- | --- |

---

## License

[MIT](LICENSE) © 2020-2024 FIWARE Foundation e.V.
