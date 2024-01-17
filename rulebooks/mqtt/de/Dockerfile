FROM quay.io/ansible/ansible-rulebook:v1.0.4
RUN pip install --no-cache-dir asyncio-mqtt==0.16.2 && \
    ansible-galaxy collection install kubealex.eda==1.0.7
