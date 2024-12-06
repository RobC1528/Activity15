# keystone
- name: Install Keystone (Identity Service)
  hosts: controller
  become: true
  tasks:
    - name: Install required packages
      apt:
        name: 
          - software-properties-common
          - mariadb-server
          - python3-pymysql
        state: present
        update_cache: yes

    - name: Configure MariaDB for OpenStack
      copy:
        content: |
          [mysqld]
          bind-address = {{ controller_ip }}
          default-storage-engine = innodb
          innodb_file_per_table = on
          max_connections = 4096
          collation-server = utf8_general_ci
          character-set-server = utf8
        dest: /etc/mysql/mariadb.conf.d/99-openstack.cnf
      notify: Restart MariaDB

    - name: Restart MariaDB
      service:
        name: mariadb
        state: restarted

    - name: Create Keystone database
      mysql_db:
        name: keystone
        state: present

    - name: Create Keystone database user
      mysql_user:
        name: keystone
        password: "{{ db_password }}"
        host: "%"
        priv: "keystone.*:ALL"
        state: present

    - name: Install Keystone packages
      apt:
        name: 
          - keystone
          - apache2
          - libapache2-mod-wsgi-py3
        state: present

    - name: Configure Keystone
      lineinfile:
        path: /etc/keystone/keystone.conf
        regexp: "^#?connection=.*"
        line: "connection = mysql+pymysql://keystone:{{ db_password }}@{{ controller_ip }}/keystone"

    - name: Populate Keystone database
      shell: |
        su -s /bin/bash keystone -c "keystone-manage db_sync"


# glance
- name: Install Glance (Image Service)
  hosts: controller
  become: true
  tasks:
    - name: Create Glance database
      mysql_db:
        name: glance
        state: present

    - name: Create Glance database user
      mysql_user:
        name: glance
        password: "{{ db_password }}"
        host: "%"
        priv: "glance.*:ALL"
        state: present

    - name: Install Glance packages
      apt:
        name: 
          - glance
        state: present

    - name: Configure Glance API
      lineinfile:
        path: /etc/glance/glance-api.conf
        regexp: "^#?connection=.*"
        line: "connection = mysql+pymysql://glance:{{ db_password }}@{{ controller_ip }}/glance"

    - name: Populate Glance database
      shell: |
        su -s /bin/bash glance -c "glance-manage db_sync"

# nova
- name: Install Nova (Compute Service)
  hosts: controller
  become: true
  tasks:
    - name: Create Nova databases
      mysql_db:
        name: nova
        state: present
      with_items:
        - nova
        - nova_api
        - nova_cell0

    - name: Create Nova database user
      mysql_user:
        name: nova
        password: "{{ db_password }}"
        host: "%"
        priv: "nova.*:ALL"
        state: present

    - name: Install Nova packages
      apt:
        name: 
          - nova-api
          - nova-conductor
          - nova-novncproxy
        state: present

    - name: Configure Nova
      lineinfile:
        path: /etc/nova/nova.conf
        regexp: "^#?connection=.*"
        line: "connection = mysql+pymysql://nova:{{ db_password }}@{{ controller_ip }}/nova"

    - name: Populate Nova databases
      shell: |
        su -s /bin/bash nova -c "nova-manage api_db sync && nova-manage cell_v2 map_cell0"
