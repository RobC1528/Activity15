# keystone
- name: Install Keystone packages
  apt:
    name:
      - keystone
      - python3-openstackclient
    state: present

- name: Configure Keystone
  template:
    src: keystone.conf.j2
    dest: /etc/keystone/keystone.conf

- name: Initialize Keystone Database
  command: keystone-manage db_sync

- name: Bootstrap Keystone
  command: >
    keystone-manage bootstrap --bootstrap-password {{ keystone_admin_password }}
                              --bootstrap-admin-url http://{{ ansible_host }}:5000/v3/
                              --bootstrap-internal-url http://{{ ansible_host }}:5000/v3/
                              --bootstrap-public-url http://{{ ansible_host }}:5000/v3/
                              --bootstrap-region-id RegionOne

# Glance
- name: Install Glance packages
  apt:
    name:
      - glance
    state: present

- name: Configure Glance
  template:
    src: glance-api.conf.j2
    dest: /etc/glance/glance-api.conf

- name: Initialize Glance Database
  command: glance-manage db_sync

# Nova
- name: Install Nova packages
  apt:
    name:
      - nova-api
      - nova-scheduler
      - nova-conductor
      - nova-compute
    state: present

- name: Configure Nova
  template:
    src: nova.conf.j2
    dest: /etc/nova/nova.conf

- name: Initialize Nova Database
  command: nova-manage db_sync

# Groupvar
keystone_admin_password: "secure_password"
