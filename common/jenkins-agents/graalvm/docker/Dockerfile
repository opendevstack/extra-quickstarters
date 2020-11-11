FROM opendevstackorg/ods-jenkins-slave-base-centos7:latest

LABEL maintainer="Rene Dupont <rene.dupont@boehringer-ingelheim.com>"

# Labels consumed by Red Hat build service
LABEL com.redhat.component="jenkins-agent-graalvm-centos7-docker" \
      name="openshift3/jenkins-agent-graalvm-centos7" \
      version="3.6" \
      architecture="x86_64" \
      release="4" \
      io.k8s.display-name="Jenkins Agent GraalVM" \
      io.k8s.description="The jenkins agent graalvm image uses GraalVM and has maven aswell as gradle on top of the jenkins slave base image." \
      io.openshift.tags="openshift,jenkins,agent,graalvm,maven,gradle"

ARG nexusUrl
ARG nexusUsername
ARG nexusPassword

ENV MAVEN_VERSION=3.6.3 \
    BASH_ENV=/usr/local/bin/scl_enable \
    ENV=/usr/local/bin/scl_enable \
    PROMPT_COMMAND=". /usr/local/bin/scl_enable" \
    HOME=/home/jenkins \
    GRADLE_USER_HOME=/home/jenkins/.gradle

# Install GraalVM
ARG GRAAL_VERSION=20.1.0
ARG GRAALVM_PKG=https://github.com/graalvm/graalvm-ce-builds/releases/download/vm-$GRAAL_VERSION/graalvm-ce-java11-linux-amd64-$GRAAL_VERSION.tar.gz
ENV GRAALVM_HOME=/opt/graalvm \
    JAVA_HOME=/opt/graalvm \    
    PATH=/opt/graalvm/bin:$PATH
    # Download and extract
RUN set -eux && \
	mkdir -p ${GRAALVM_HOME} && \
	cd ${GRAALVM_HOME} && \
    curl --fail --silent --location --retry 3 ${GRAALVM_PKG} \
    | gunzip | tar -xC ${GRAALVM_HOME} --strip-components=1 
    # Set alternative links
RUN mkdir -p "/usr/java" \
    && ln -sfT "$JAVA_HOME" /usr/java/default \
    && ln -sfT "$JAVA_HOME" /usr/java/latest \
    && for bin in "${JAVA_HOME}/bin/"*; do \
        base="$(basename "$bin")"; \
        [ ! -e "/usr/bin/$base" ]; \
        alternatives --install "/usr/bin/$base" "$base" "$bin" 1; \
    done     
RUN alternatives --set java ${GRAALVM_HOME}/bin/java && \
    alternatives --set javac ${GRAALVM_HOME}/bin/javac && \
    java -version && \
    javac -version    
    # Install native-image plugin and required libs for native builds
RUN cd $JAVA_HOME/bin/ \
    && chmod +x gu \
    && ./gu install native-image \
    && yum install -y gcc glibc-devel zlib-devel libstdc++-devel
RUN chown -R 1001:0 $GRAALVM_HOME && \
    chmod -R g+rw $GRAALVM_HOME

# Install Maven
ARG BASE_URL=https://apache.osuosl.org/maven/maven-3/${MAVEN_VERSION}/binaries
RUN mkdir -p /usr/share/maven /usr/share/maven/ref \
  && curl -fsSL -o /tmp/apache-maven.tar.gz ${BASE_URL}/apache-maven-${MAVEN_VERSION}-bin.tar.gz \
  && tar -xzf /tmp/apache-maven.tar.gz -C /usr/share/maven --strip-components=1 \
  && rm -f /tmp/apache-maven.tar.gz \
  && ln -s /usr/share/maven/bin/mvn /usr/bin/mvn \
  && mvn --version
ENV MAVEN_HOME /usr/share/maven
ENV MAVEN_CONFIG $HOME/.m2

# Container support is now integrated in Java 11, the +UseCGroupMemoryLimitForHeap option has been pruned
ENV JAVA_TOOL_OPTIONS="-XX:+UnlockExperimentalVMOptions -Dsun.zip.disableMemoryMapping=true"

# When bash is started non-interactively, to run a shell script, for example it
# looks for this variable and source the content of this file. This will enable
# the SCL for all scripts without need to do 'scl enable'.
ADD contrib/bin/scl_enable /usr/local/bin/scl_enable
ADD contrib/bin/configure-slave /usr/local/bin/configure-slave

# set java proxy var
COPY set_maven_proxy.sh /tmp/set_maven_proxy.sh
RUN chmod 777 /tmp/set_maven_proxy.sh

ADD ./contrib/settings.xml $HOME/.m2/
RUN mv $HOME/.m2/settings.xml $HOME/.m2/settings.xml.orig && \
 /tmp/set_maven_proxy.sh && \
 xpr=$(cat /tmp/mvn_proxy) && \
 xpr="${xpr//\//\\/}" && \
 xpr="${xpr//|/\\|}" && \
 cat $HOME/.m2/settings.xml.orig | sed -e "s|<proxies>|<proxies>$xpr|g" > $HOME/.m2/settings.xml && \
 sed -i "s/__NEXUS_USER/$nexusUsername/gi" $HOME/.m2/settings.xml && \
 sed -i "s/__NEXUS_PW/$nexusPassword/gi" $HOME/.m2/settings.xml && \
 sed -i "s|__NEXUS_URL|$nexusUrl|gi" $HOME/.m2/settings.xml && \
 cat $HOME/.m2/settings.xml

# install gradle ..
ADD gradlew /tmp/gradlew
RUN mkdir -p /tmp/gradle/wrapper
ADD gradle/* /tmp/gradle/wrapper
RUN ls /tmp/gradle/wrapper

# set java proxy var
COPY set_gradle_proxy.sh /tmp/set_gradle_proxy.sh
RUN chmod 777 /tmp/set_gradle_proxy.sh

RUN mkdir $GRADLE_USER_HOME
RUN /tmp/set_gradle_proxy.sh 

#set java proxy via JAVA_OPTS as src
RUN bash -l -c 'echo export JAVA_OPTS="$(/tmp/set_java_proxy.sh && echo $JAVA_OPTS)" >> /etc/bash.bashrc'

RUN chown -R 1001:0 $HOME && \
    chmod -R g+rw $HOME
USER 1001

