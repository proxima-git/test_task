# Тестовое задание
**Задача**: Написaть Ansible роль, устанавливающую пароль пользователю (в нашем случае test) и флаг необходимости смены пароля при первом входе в систему.
___
**Используемые инструменты**: Ansible 9.2.0
___
##  Шаг 1
Создадим дериктории для нашей роли, в которых будут храниться наши исполняемые задачи и переменные:
```shell
proxima@proximaserver:~/ansible/roles$ tree
.
└── set_user_password
    ├── tasks
    │   └── main.yml
    └── vars
        └── user1.yml

```
В файле `user1.yml` находятся перемнные нашего юзера (логин/пароль):
```yml
---
username: test
user_password: qwerty123

```
В файле `tasks/main.yml` прописана задача, которая будет устанавливать пароль нашему пользователю. Если в управляемой системе не будет существовать пользователя, параметр `state: present` создаст его.\
Команда `chage -d 0` установит флаг указаному пользователю о необходимости сменить пароль:
```yml
---
# Укажем переменные
- name: include vars
  include_vars:
    file: "../vars/user1.yml"


# Установим пароль пользователю.
# Если пользователя не существует в системе, модуль его создаст
- name: Set password for user
  user:
    name: "{{ username }}"
    state: present
    shell: /bin/bash
    password: "{{ user_password | password_hash('sha512') }}"

# Установим пользователю флаг необходимости смены пароля при первом входе в систему.
- name: Force user to change password
  shell: chage -d 0 "{{ username }}"
```
## Шаг 2
Так как в нашей задаче мы используем конфиденциальную информацию, я посчитал что хорошим тоном будет зашифровать данные. Для этого воспользуемся методом `ansible-vault`.

Создадим файл `secret.txt`, в котором будет храниться пароль для шифрования/расшифрования:
```shell
proxima@proximaserver:~/ansible$ cat secret.txt
mypassword
```
Используя файл, зашифруем `user1.yml`:
```shell
proxima@proximaserver:~/ansible$ ansible-vault encrypt roles/set_user_password/vars/user1.yml --vault-pass-file secret.txt
Encryption successful
proxima@proximaserver:~/ansible$ cat roles/set_user_password/vars/user1.yml
$ANSIBLE_VAULT;1.1;AES256
30643361393539366261303035323234373864343165366137633434643939323635316139316230
3638396664623634343964396665636434373066396166340a396364303735336539343866633830
64653730616139396366663762643839653830383138306131333165316563623333346237666530
6637343132366338630a356564316363316131633566346432333032316333383734303035626139
30646137353139386336643865356235666337623031323434616633313730643864386636646263
3738383066646335383366363534633163333863326462373962
```
## Шаг 3
Напишем плейбук для испольнения нашей роли:
```yml
proxima@proximaserver:~/ansible$ cat play.yml
---
# Плейбук, запускающий роль "set_user_password"
- name: Run role - set_user_password
  hosts: localhost
  become: yes

  roles:
    - set_user_password

```
Так как в задании не указано про управляемые хосты, я решил протестировать данный плейбук на localhost.\
При запуске плейбука укажите параметр `--vault-pass-file secret.txt` для расшифровки наших данных и `-K` для исполнения команды от суперпользователя:
```shell
proxima@proximaserver:~/ansible$ ansible-playbook play.yml --vault-pass-file secret.txt -K
BECOME password:

PLAY [Run role - set_user_password] *********************************************************************************************************************

TASK [Gathering Facts] *********************************************************************************************************************
ok: [localhost]

TASK [set_user_password : include vars] *********************************************************************************************************************
ok: [localhost]

TASK [set_user_password : Set password for user] *********************************************************************************************************************
changed: [localhost]

TASK [set_user_password : Force user to change password] *********************************************************************************************************************
changed: [localhost]

PLAY RECAP *********************************************************************************************************************
localhost                  : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```
Проверим результат, ввойдя в систему под юзером `test`:
```shell
proxima@proximaserver:~/ansible$ su test
Password:
You are required to change your password immediately (administrator enforced).
Changing password for test.
Current password:
New password:
Retype new password:
test@proximaserver:/home/proxima/ansible$

```
## Дополнительно
Итоговое решение:
```shell
proxima@proximaserver:~/ansible$ tree
.
├── play.yml
├── roles
│   └── set_user_password
│       ├── tasks
│       │   └── main.yml
│       └── vars
│           └── user1.yml
└── secret.txt

5 directories, 4 files

```
Среды, на которых проводил тесты:
- Ubuntu 24.04 LTS
- AlmaLinux 9.4


