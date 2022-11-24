FROM jenkins/jenkins:lts

#stopped working, need to install via UI or investigate plugin management CLI tool
#COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
#RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt


USER root
RUN apt-get update && apt-get upgrade -y && apt-get install -y python3 python3-pip && apt-get install -y rsync
RUN pip install pytest allure-pytest 'selene>=2.0.b5' 'python-dotenv==0.20.0'
USER jenkins
