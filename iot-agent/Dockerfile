ARG  NODE_VERSION=12
FROM node:${NODE_VERSION}-slim

# IMPORTANT: For production environments use Docker Secrets to protect values of the sensitive ENV 
# variables defined below, by adding _FILE to the name of the relevant variable.
#
# - IOTA_AUTH_USER, IOTA_AUTH_PASSWORD - when using Keystone Security 
# - IOTA_AUTH_CLIENT_ID, IOTA_AUTH_CLIENT_SECRET - when using OAuth2 Security


COPY . /opt/iotXML/

WORKDIR /opt/iotXML


# hadolint ignore=DL3008
RUN \
	# Ensure that Git is installed prior to running npm install
	apt-get update && \
	apt-get install -y --no-install-recommends git && \
	echo "INFO: npm install --production..." && \
	npm install --only=prod --no-package-lock --no-optional && \
	# Remove Git and clean apt cache
	apt-get clean && \
	apt-get remove -y git && \
	apt-get -y autoremove

LABEL "maintainer"="FIWARE Foundation"
LABEL "org.opencontainers.image.authors"="https://www.fiware.org/contact-us/"
LABEL "org.opencontainers.image.documentation"="https://fiware-tutorials.readthedocs.io/en/latest/custom-iot-agent/index.html"
LABEL "org.opencontainers.image.vendor"="FIWARE Foundation e.V."
LABEL "org.opencontainers.image.licenses"="AGPL-3.0-only"
LABEL "org.opencontainers.image.title"="IoT Agent for a custom XML protocol"
LABEL "org.opencontainers.image.description"="An Internet of Things Agent for a XML protocol (with AMQP, HTTP and MQTT transports). This IoT Agent is designed to be a bridge between Ultralight and the NGSI interface of a context broker."
LABEL "org.opencontainers.image.source"="https://github.com/FIWARE/tutorials.Custom-IoT-Agent"
LABEL "org.nodejs.version"="${NODE_VERSION}"

USER node
ENV NODE_ENV=production

# Expose 4041 for NORTH PORT, 7896 for HTTP PORT
ARG IOTA_NORTH_PORT
ARG IOTA_HTTP_PORT
EXPOSE ${IOTA_NORTH_PORT:-4041} ${IOTA_HTTP_PORT:-7896}

CMD ["node", "/opt/iotXML/bin/iotagent-xml", "-- ", "config.js"]
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
   CMD ["npm", "run", "healthcheck"]
