# launchapp.yml
```yml
---
- hosts: localhost
  connection: local
  gather_facts: yes
  vars_files:
  - /etc/ansible/group_vars/all/pass.yml
  vars:
    key_name: eng103a_zilamo_app
    region: eu-west-1
    image: ami-07d8796a2b0f8d29c
    id: "zilamo-cloud-ansible-app"
    sec_group: "{{ id }}-sec2"
    ansible_python_interpreter: /usr/bin/python3
  tasks:
    - name: Provisioning EC2 instances
      block:
      - name: Upload public key to AWS
        ec2_key:
          name: "{{ key_name }}"
          key_material: "{{ lookup('file', '~/.ssh/eng103a_zilamo_app.pub') }}"
          region: "{{ region }}"
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
      - name: Create security group
        ec2_group:
          name: "{{ sec_group }}"
          vpc_id: vpc-0c1b307a57b01f06f
          description: "Sec group for app {{ id }}"
          region: "{{ region }}"
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          rules:
            - proto: tcp
              ports:
                - 22
              cidr_ip: 0.0.0.0/0
              rule_desc: allow all on ssh port
            - proto: tcp
              ports:
                - 80
              cidr_ip: 0.0.0.0/0
              rule_desc: allow all on http port
            - proto: tcp
              ports:
                - 3000
              cidr_ip: 0.0.0.0/0
              rule_desc: allow all on 3000 port
        register: result_sec_group
      - name: Provision instance(s)
        ec2:
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          key_name: "{{ key_name }}"
          group_id: "{{ result_sec_group.group_id }}"
          image: "{{ image }}"
          instance_type: t2.micro
          region: "{{ region }}"
          vpc_subnet_id: subnet-0128d81162fe6f84b
          assign_public_ip: yes
          wait: true
          count: 1
          instance_tags:
            Name: eng103a_zilamo_cloud_ansible_app
      tags: ['never', 'create_ec2']
```
# launchdb.yml
```yml
---
- hosts: localhost
  connection: local
  gather_facts: yes
  vars_files:
  - /etc/ansible/group_vars/all/pass.yml
  vars:
    key_name: eng103a_zilamo_db
    region: eu-west-1
    image: ami-07d8796a2b0f8d29c
    id: "zilamo-ansible-vpc-db"
    sec_group: "{{ id }}-sec-db"
    ansible_python_interpreter: /usr/bin/python3
  tasks:
    - name: Provisioning EC2 instances
      block:
      - name: Upload public key to AWS
        ec2_key:
          name: "{{ key_name }}"
          key_material: "{{ lookup('file', '~/.ssh/eng103a_zilamo_db.pub') }}"
          region: "{{ region }}"
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
      - name: Create security group
        ec2_group:
          name: "{{ sec_group }}"
          description: "Sec group for app {{ id }}"
          region: "{{ region }}"
          vpc_id: vpc-0c1b307a57b01f06f
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          rules:
            - proto: tcp
              ports:
                - 22
              cidr_ip: 0.0.0.0/0
              rule_desc: allow all on ssh port
            - proto: tcp
              ports:
                - 27017
              cidr_ip: 0.0.0.0/0
              rule_desc: allow all on 27017 port
        register: result_sec_group
      - name: Provision instance(s)
        ec2:
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          key_name: "{{ key_name }}"
          group_id: "{{ result_sec_group.group_id }}"
          image: "{{ image }}"
          instance_type: t2.micro
          region: "{{ region }}"
          vpc_subnet_id: subnet-0f7cec3b6ed1b5c30
          assign_public_ip: no
          wait: true
          count: 1
          instance_tags:
            Name: eng103a_zilamo_cloud_ansible_db
      tags: ['never', 'create_ec2']
```
# provisionapp.yml
```yml
---
-  hosts: awsapp
   gather_facts: yes
   become: yes
   tasks:
   -  name: moving app folder in
      synchronize:
        src: ~/app/
        dest: ~/
   -  name: load a specific version of nodejs
      shell: curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -

   -  name: install the required packages
      apt:
        pkg:
          - nginx
          - nodejs
          - npm
        update_cache: yes
   -  name: nginx configuration for reverse proxy
      synchronize:
        src: ~/app/default
        dest: /etc/nginx/sites-available/default
   -  name: nginx restart
      service: name=nginx state=restarted
   -  name: nginx enable
      service: name=nginx enabled=yes
   -  name: initialise DB_HOST and seed the DB
      become_user: ubuntu
      shell:
         cd ~/app; echo "mongodb://privateip:27017/posts" > mongoip; node seeds/seed.js;
   -  name: install and run the app
      become_user: ubuntu
      shell:
         cd ~/app; npm install; screen -d -m npm start;
```
# provisiondb.yml
```yml
---
-  hosts: awsdb
   gather_facts: yes
   become: true
   tasks:
   -  name: installing mongo
      apt:
        name: mongodb
        state: present
        update_cache: yes
   - name: allow 0.0.0.0
     ansible.builtin.lineinfile:
      path: /etc/mongodb.conf
      regexp: '^bind_ip = '
      line: bind_ip = 0.0.0.0
   -  name: restart mongodb
      service: name=mongodb state=restarted
   -  name: mongod enable
      service: name=mongodb enabled=yes
```

# hosts
```yaml
[local]
localhost ansible_python_interpreter=/usr/bin/python3

[awsapp]
18.203.233.161 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/eng103a_zilamo_app

[awsdb]
52.215.206.137 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/eng103a_zilamo_db
```


# launch commands
```bash
sudo ansible-playbook launchapp.yml --ask-vault-pass --tags create_ec2 --tags=ec2-create -e "ansible_python_interpreter=/usr/bin/python3"
sudo ansible-playbook launchdb.yml --ask-vault-pass --tags create_ec2 --tags=ec2-create -e "ansible_python_interpreter=/usr/bin/python3"
sudo ansible-playbook provisiondb.yml --ask-vault-pass
sudo ansible-playbook provisionapp.yml --ask-vault-pass 
sudo ansible aws -m ping --ask-vault-pass
```
My launchapp, launchdb, provisionapp, provisiondb YML files are located in [yml files](launch%20and%20provision%20yml%20files.md")
