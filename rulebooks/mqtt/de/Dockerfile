FROM quay.io/ansible/ansible-rulebook:v1.0.1
RUN pip install paho-mqtt asyncio-mqtt && \
    curl -Lo ./.ansible/collections/ansible_collections/ansible/eda/extensions/eda/plugins/event_source/mqtt.py https://raw.githubusercontent.com/ansible/event-driven-ansible/725d244cd3a764a6c9a4540a6526eb3bbe0cb452/extensions/eda/plugins/event_sources/mqtt.py && \
    sed -i 's/from typing import Any, dict/from typing import Any/g' ./.ansible/collections/ansible_collections/ansible/eda/extensions/eda/plugins/event_source/mqtt.py
