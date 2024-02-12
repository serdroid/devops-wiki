# How to change a user's password with ansible

## Generate a password hash
There are a few different methods to generate a password hash. We can use this FAQ document https://docs.ansible.com/ansible/latest/reference_appendices/faq.html#how-do-i-generate-encrypted-passwords-for-the-user-module for reference.

I prefer to use python passlib library.

## Generate a new password hash with passlib
It is assumed that python and pip installed on the host. A temporary directory is created and a virtual environment has been setup.

```
cd /tmp
mkdir newpass
cd newpass
python3 -m venv updpass
source updpass/bin/activate

pip install passlib
```

The command below prompts for the password, input the new password and then press enter.

```
python -c "from passlib.hash import sha512_crypt; import getpass; print(sha512_crypt.using(rounds=5000).hash(getpass.getpass()))" > newpasswd

chmod 600 newpasswd
```
Generated password hash for the given password is written into newpasswd file. Now we got password hash, so we can deactivate the virtual environment.

```
deactivate
```

## Ansible playbook
The ansible playbook below can be used to change password of a user. The playbook uses two variables: 'chuser' for the user, 'newwpasswd' for the password hash.

```
cat > change-user-passwd.yml << EOF
---
- name: Change user password playbook
  hosts: all
  remote_user: ansible
  become: yes
  become_user: root
  tasks:
  - name: Change user password
    ansible.builtin.user:
      name: "{{ chuser }}"
      update_password: always
      password: "{{ newpasswd }}"
EOF
```

The playbook can be run like below. Do not forget to remove password hash file.

```
ansible-playbook -i hosts change-user-passwd.yml --extra-vars chuser=root --extra-vars newpasswd=$(cat /tmp/newpass/newpasswd)

rm -rf /tmp/newpass
```

