- name: Wait for apt lock to be released
  shell: |
    while fuser /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock >/dev/null 2>&1; do
      echo "Waiting for apt lock to be released..."
      sleep 5
    done
  changed_when: false


# keystone
- name: Install Keystone and dependencies
  apt:
    name:
      - keystone
      - python3-openstackclient
      - apache2
      - libapache2-mod-wsgi-py3
    state: present
    update_cache: true

- name: Configure Keystone service
  template:
    src: keystone.conf.j2
    dest: /etc/keystone/keystone.conf
    owner: root
    group: root
    mode: '0644'

- name: Synchronize Keystone database
  command: keystone-manage db_sync

- name: Bootstrap Keystone
  command: >
    keystone-manage bootstrap
    --bootstrap-password {{ keystone_admin_password }}
    --bootstrap-admin-url http://{{ ansible_fqdn }}:5000/v3/
    --bootstrap-internal-url http://{{ ansible_fqdn }}:5000/v3/
    --bootstrap-public-url http://{{ ansible_fqdn }}:5000/v3/
    --bootstrap-region-id RegionOne

- name: Configure Apache for Keystone
  block:
    - name: Enable Apache modules
      command: a2enmod wsgi
      notify: Restart Apache

    - name: Create Keystone WSGI file
      copy:
        content: |
          WSGIDaemonProcess keystone processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
          WSGIProcessGroup keystone
          WSGIScriptAlias / /usr/bin/keystone-wsgi-public

          <VirtualHost *:5000>
              WSGIProcessGroup keystone
              WSGIScriptAlias / /usr/bin/keystone-wsgi-public
              WSGIPassAuthorization On
              ErrorLog /var/log/apache2/keystone-error.log
              CustomLog /var/log/apache2/keystone-access.log combined
          </VirtualHost>

          <VirtualHost *:35357>
              WSGIProcessGroup keystone
              WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
              WSGIPassAuthorization On
              ErrorLog /var/log/apache2/keystone-error.log
              CustomLog /var/log/apache2/keystone-access.log combined
          </VirtualHost>
        dest: /etc/apache2/sites-available/keystone.conf
        owner: root
        group: root
        mode: '0644'

    - name: Enable Keystone site
      command: a2ensite keystone
      notify: Restart Apache

- name: Restart Apache to apply changes
  systemd:
    name: apache2
    state: restarted
    enabled: true

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
