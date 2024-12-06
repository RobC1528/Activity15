# Neutron
- name: Install Neutron packages
  apt:
    name:
      - neutron-server
      - neutron-plugin-ml2
      - neutron-linuxbridge-agent
      - neutron-dhcp-agent
      - neutron-metadata-agent
    state: present

- name: Configure Neutron
  lineinfile:
    path: /etc/neutron/neutron.conf
    regexp: "^#?connection=.*"
    line: "connection = mysql+pymysql://neutron:{{ db_password }}@{{ controller_ip }}/neutron"

- name: Populate Neutron database
  shell: |
    su -s /bin/bash neutron -c "neutron-db-manage upgrade head"

- name: Restart Neutron services
  service:
    name: "{{ item }}"
    state: restarted
  with_items:
    - neutron-server
    - neutron-linuxbridge-agent
    - neutron-dhcp-agent
    - neutron-metadata-agent

# Horizon
- name: Install Horizon
  apt:
    name: openstack-dashboard
    state: present

- name: Restart Apache2
  service:
    name: apache2
    state: restarted

# Cinder
- name: Install Cinder packages
  apt:
    name:
      - cinder-api
      - cinder-scheduler
      - python3-cinderclient
    state: present

- name: Configure Cinder
  lineinfile:
    path: /etc/cinder/cinder.conf
    regexp: "^#?connection=.*"
    line: "connection = mysql+pymysql://cinder:{{ db_password }}@{{ controller_ip }}/cinder"

- name: Populate Cinder database
  shell: |
    su -s /bin/bash cinder -c "cinder-manage db sync"

- name: Restart Cinder services
  service:
    name: "{{ item }}"
    state: restarted
  with_items:
    - cinder-api
    - cinder-scheduler
