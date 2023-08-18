#FROM containers.intersystems.com/intersystems/irishealth-community:2023.1.1.380.0
FROM containers.intersystems.com/intersystems/iris:2023.1.1.380.0

USER ${ISC_PACKAGE_MGRUSER}
COPY  iris.script /tmp/iris.script

RUN iris start IRIS \
    && iris session IRIS < /tmp/iris.script \
    && iris stop IRIS quietly