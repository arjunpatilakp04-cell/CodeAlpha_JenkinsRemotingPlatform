FROM jenkins/jenkins:lts-jdk17

USER root

# Install plugins needed for JCasC, security, and remoting
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

# JCasC config will be mounted at runtime
ENV CASC_JENKINS_CONFIG=/var/jenkins_home/casc_configs
ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"

USER jenkins