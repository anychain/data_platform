# Creates an Ambari Server base on centos
FROM centos:7
MAINTAINER gengjh<david@esse.io>

ENV ARTIFACTS_REPO http://repo.esse.io:33080/ARTIFACTS

ADD ambari.repo /etc/yum.repos.d/

# increase timeouts to avoid "No more mirrors to try" if yum repos are busy for a few minutes
RUN echo "retries=0" >> /etc/yum.conf
RUN echo "timeout=60" >> /etc/yum.conf

RUN yum install -y tar git curl bind-utils unzip net-tools

# install Ambari specified 1.8 jdk
ADD $ARTIFACTS_REPO/jdk-8u40-linux-x64.tar.gz /usr/jdk64/
RUN cd /usr/jdk64 && tar -xf jdk-8u40-linux-x64.tar.gz && rm -f jdk-8u40-linux-x64.tar.gz
ENV JAVA_HOME /usr/jdk64/jdk1.8.0_40
ENV PATH $PATH:$JAVA_HOME/bin

# jce
ADD $ARTIFACTS_REPO/jce_policy-8.zip $JAVA_HOME/jre/lib/security/
RUN cd $JAVA_HOME/jre/lib/security && unzip jce_policy-8.zip && rm -f jce_policy-8.zip && mv UnlimitedJCEPolicyJDK8/*jar . && rm -rf UnlimitedJCEPolicyJDK8

RUN yum install -y ambari-server ambari-agent
# RUN ambari-server setup --silent --java-home $JAVA_HOME

# increase PermGen Space for Ambari views
ENV AMBARI_JVM_ARGS -XX:MaxPermSize=512m

# fix annoying PAM error 'couldnt open session'
RUN sed -i "/pam_limits/ s/^/#/" /etc/pam.d/*

ADD install-ssh.sh /tmp/
RUN sh /tmp/install-ssh.sh

# kerberos client
RUN yum install -y krb5-workstation

# add ambari shell to the image so new users don't need the 1GB java image
RUN curl -o /tmp/ambari-shell.jar http://repo.esse.io:33080/ARTIFACTS/ambari-shell-0.1.31.jar
ADD install-cluster.sh /tmp/
ADD wait-for-host-number.sh /tmp/
ADD ambari-shell.sh /tmp/
ENV PLUGIN_PATH /plugins
WORKDIR /tmp

# fixing pgsql issue
RUN rm -rf /tmp/.s.PGSQL.5432.*

# add a custom folder to the hadoop classpath
RUN mkdir -p /usr/lib/hadoop/lib
ENV HADOOP_CLASSPATH /usr/lib/hadoop/lib/*

# add mysql and psql connectors to ambari-server so it can be downloaded by services (e.g.: Ranger)
ADD $ARTIFACTS_REPO/mysql-connector-java-5.1.17.jar /tmp/
ADD $ARTIFACTS_REPO/postgresql-8.4-703.jdbc4.jar /tmp/

ADD internal-hostname.sh /etc/ambari-agent/conf/internal-hostname.sh
RUN sed -i "/\[agent\]/ a hostname_script=\/etc\/ambari-agent\/conf\/internal-hostname.sh" /etc/ambari-agent/conf/ambari-agent.ini

# RUN mkdir /var/log/hadoop-metrics && chmod 777 /var/log/hadoop-metrics
# ADD hadoop-metrics2.properties.j2 /var/lib/ambari-server/resources/stacks/HDP/2.0.6/hooks/before-START/templates/hadoop-metrics2.properties.j2

COPY start-agent /start-agent
COPY start-server /start-server

# download jq binaries
RUN curl -o /usr/bin/jq $ARTIFACTS_REPO/jq && chmod +x /usr/bin/jq

# increase agent timeout
RUN sed -i "s/agent.package.install.task.timeout=1800/agent.package.install.task.timeout=2400/" /etc/ambari-server/conf/ambari.properties
RUN sed -i "s/agent.task.timeout=900/agent.task.timeout=2000/" /etc/ambari-server/conf/ambari.properties

# do not use the docker0 interface
RUN sed -i "s/\"ifconfig\"/\"ifconfig eth0\"/" /usr/lib/python2.6/site-packages/ambari_agent/Facter.py

# install Zeppelin
RUN yum install -y wget
RUN wget $ARTIFACTS_REPO/apache-maven-3.3.3-bin.tar.gz && tar xzf apache-maven-3.3.3-bin.tar.gz -C /usr/local && cd /usr/local && ln -s apache-maven-3.3.3 maven
ENV M2_HOME /usr/local/maven
ENV PATH $PATH:$M2_HOME/bin
RUN cd /var/lib/ambari-server/resources/stacks/HDP/2.3/services && git clone https://github.com/hortonworks-gallery/ambari-zeppelin-service.git

# fix Ambari views with WASB
# remove after Ambari 2.1.3
# ADD ambari-views-utils-fixed-wasb.jar /tmp/wasb/WEB-INF/lib/
# ADD azure-storage-2.2.0.jar /tmp/wasb/WEB-INF/lib/
# ADD hadoop-azure-2.7.1.jar /tmp/wasb/WEB-INF/lib/
# RUN cd /tmp/wasb/WEB-INF/lib/ && mv ambari-views-utils-fixed-wasb.jar $(jar tf /var/lib/ambari-server/resources/views/files-*.jar | grep ambari-view | cut -d "/" -f 3)
# RUN cd /tmp/wasb && jar -uf /var/lib/ambari-server/resources/views/files*.jar WEB-INF/lib/*.jar
# RUN cd /tmp/wasb && jar -uf /var/lib/ambari-server/resources/views/hive*.jar WEB-INF/lib/*.jar
# RUN cd /tmp/wasb && jar -uf /var/lib/ambari-server/resources/views/pig*.jar WEB-INF/lib/*.jar
# RUN cd /tmp/wasb && jar -uf /var/lib/ambari-server/resources/views/slider*.jar WEB-INF/lib/*.jar

# Add customize files
ADD logo.png /usr/lib/ambari-server/web/img/
ADD logo.png /usr/lib/ambari-server/web/img/logo-white.png
ADD index.html /usr/lib/ambari-server/web/
ADD app.js.gz /usr/lib/ambari-server/web/javascripts/
ADD edp1.1-repoinfo.xml /var/lib/ambari-server/resources/stacks/HDP/2.3/repos/repoinfo.xml
ADD $ARTIFACTS_REPO/ambari-admin-2.1.2.377.jar /var/lib/ambari-server/resources/views/
ENV PS1 "[\u@docker-ambari \W]# "

VOLUME /var/log

EXPOSE 8080 8440 8441
CMD ["/start-server"]
