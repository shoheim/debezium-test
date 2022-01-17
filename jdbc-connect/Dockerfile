FROM redhat/ubi8

ARG KAFKA_VERSION=3.0.0
ARG SCALA_VERSION=2.12
ARG SHA512HASH="CB82E685A76FA6041DCB39A8519B4F3C1A16066E9D5D8EAB11A825B517D91F690ED9AF40492A11471265AE9C486017FD128492F867E3BE63EC7770D44F7E54D2"

ENV KAFKA_VERSION=$KAFKA_VERSION \
    SCALA_VERSION=$SCALA_VERSION \
    KAFKA_HOME=/kafka \
    SHA512HASH=$SHA512HASH \
    KAFKA_URL_PATH=kafka/$KAFKA_VERSION/kafka_$SCALA_VERSION-$KAFKA_VERSION.tgz

ENV KAFKA_DATA=$KAFKA_HOME/data

USER root

RUN set -x && \
    yum install -y java-1.8.0-openjdk && \
    yum install -y zip && \
    yum install -y iputils

RUN groupadd -r kafka -g 1001 && useradd -u 1001 -r -g kafka -m -d $KAFKA_HOME -s /sbin/nologin -c "Kafka user" kafka && \
    chmod 755 $KAFKA_HOME
RUN mkdir $KAFKA_DATA && \
    mkdir $KAFKA_HOME/logs

RUN chown -R kafka $KAFKA_HOME && \
    chgrp -R kafka $KAFKA_HOME

RUN curl -fSL -o /tmp/kafka.tgz $(curl --stderr /dev/null https://www.apache.org/dyn/closer.cgi\?as_json\=1 | sed -rn 's/.*"preferred":.*"(.*)"/\1/p')$KAFKA_URL_PATH || curl -fSL -o /tmp/kafka.tgz https://archive.apache.org/dist/$KAFKA_URL_PATH

RUN echo "$SHA512HASH /tmp/kafka.tgz" | sha512sum -c - &&\
    tar -xzf /tmp/kafka.tgz -C $KAFKA_HOME --strip-components 1 &&\
    rm -f /tmp/kafka.tgz

RUN zip -d /kafka/libs/log4j-1.2.17.jar org/apache/log4j/net/JMSAppender.class org/apache/log4j/net/SocketServer.class

COPY ./log4j.properties $KAFKA_HOME/config/log4j.properties

# RUN mkdir $KAFKA_HOME/config.orig &&\
#    mv $KAFKA_HOME/config/* $KAFKA_HOME/config.orig &&\
#    chown -R kafka:kafka $KAFKA_HOME/config.orig

RUN rm -f $KAFKA_HOME/libs/*-{sources,javadoc,scaladoc}.jar* &&\
    rm -r $KAFKA_HOME/site-docs

RUN sed -i 's/base_dir=\$(dirname \$0)\/../base_dir=\$(realpath \$(dirname \$0)\/..)/' $KAFKA_HOME/bin/kafka-run-class.sh

RUN chmod -R g+w,o+w $KAFKA_HOME

RUN mkdir /usr/local/share/java

COPY ./kafka-connect-jdbc-sink-0.0.1-jar-with-dependencies.jar /usr/local/share/java
COPY ./db2jcc4.jar /usr/local/share/java

ENV CLASSPATH=/usr/local/share/java

COPY ./config/connect-distributed.properties $KAFKA_HOME/config/connect-distributed.properties

COPY ./docker-entrypoint.sh $KAFKA_HOME

RUN chmod a+x $KAFKA_HOME/docker-entrypoint.sh

USER kafka

WORKDIR $KAFKA_HOME

EXPOSE 8083

ENTRYPOINT ["/kafka/docker-entrypoint.sh"]
