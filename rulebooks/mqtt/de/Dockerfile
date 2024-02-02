FROM quay.io/ansible/ansible-rulebook:stable-1.0
RUN pip install --no-cache-dir aiomqtt==2.0.0 && \
    ansible-galaxy collection install kubealex.eda==1.0.8
