FROM jenkins/jenkins:lts

USER root
RUN apt-get update && apt-get upgrade -y && apt-get install -y python3 python3-pip && apt-get install -y rsync
RUN pip install pytest allure-pytest 


USER jenkins
