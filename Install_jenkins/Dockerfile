FROM jenkins/jenkins:2.555.3-jdk21
USER root
RUN apt-get update && apt-get install -y lsb-release ca-certificates curl && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
    chmod a+r /etc/apt/keyrings/docker.asc && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
    https://download.docker.com/linux/debian $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" \
    | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && apt-get install -y docker-ce-cli && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
USER jenkins

ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"

ENV CASC_JENKINS_CONFIG="/var/jenkins_home/jenkins-casc.yaml"

COPY jenkins-casc.yaml /usr/share/jenkins/ref/jenkins-casc.yaml

RUN jenkins-plugin-cli --plugins "ldap blueocean docker-workflow json-path-api configuration-as-code git workflow-aggregator pipeline-stage-view pipeline-utility-steps workflow-scm-step workflow-cps credentials-binding timestamper ws-cleanup job-dsl matrix-auth matrix-project antisamy-markup-formatter script-security scm-api ssh-credentials ssh-slaves git-client git-server docker-commons docker-java-api docker-plugin docker-build-step docker-workflow role-strategy "