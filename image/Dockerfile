FROM jenkins/jenkins:lts
USER root
RUN apt-get update && apt-get upgrade -y && apt-get install -y python3 python3-pip && apt-get install -y rsync
RUN pip install pytest allure-pytest 'selene>=2.0.b4'
USER jenkins