
https://qiita.com/hiroyuki_onodera/items/aef6194b5da425c198d8

# 課題:
Moleculeを使ってplaybookのテストなどを行う際に、クラウドにテスト環境を立ち上げる場合、terraformを使用すると以下のメリットがある。

- 様々なクラウドに対応可能
- 冪等性はterraformにて対応

ここでは、AWS EC2環境を例に、Molecule + Terraformで設定を試みる。(molecule-ec2は使用しない。)

# 環境:

- macOS 10.15.7
- Python(pyenv) 3.8.2
- ansible 2.10.2
- molecule 3.1.5
- Terraform v0.13.4
- yq 2.10.1 (オプション)
- vm接続に使用するssh-key ~/.ssh/id_rsa, ~/.ssh/id_rsa.pub
- 環境変数 AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY

# 主要ファイル:

```
.
├── molecule
│   └── default
│       ├── tf_common.tf.yml          # terraform用ec2設定ファイル1
│       ├── tf_ins01.tf.yml           # terraform用ec2設定ファイル2
│       │
│       ├── (tf_common.tf.json)       # tf_common.tf.ymlから生成
│       ├── (tf_ins01.tf.json)        # tf_ins01.tf.ymlから生成
│       │
│       ├── (instance_conf-ins01.yml) # terraform作成
│       │                             # moleculeに連携するインスタンスへの接続情報
│       │
├── (terraform.tfstate)       # terraform作成。ステータス情報
│       │
│       ├── molecule.yml              # molecule定義情報
│       ├── create.yml                # molecule vm作成playbook
│       ├── prepare.yml               # molecule vm事前準備playbook
│       ├── converge.yml              # molecule vmへの変更反映playbook
│       ├── verify.yml                # molecule vm検証playbook
│       └── destroy.yml               # molecule vm削除playbook
└── tasks
    └── main.yml
```

サンプルコードは以下
https://github.com/hiroyuki-onodera/molecule-delegated-terraform-ec2

# CH1: Terraform による AWS EC2管理

## 事前準備: IAMにてAmazonEC2FullAccess権限を持つユーザー追加
AWSコンソールにてIdentity and Access Management (IAM) > ユーザー からユーザーを追加

ここでは以下の様にしている

- AWS アクセスの種類: プログラムによるアクセス - アクセスキーを使用
- アクセス権限の境界: アクセス権限の境界が設定されていません
- アクセス許可の設定
    - 既存のポリシーを直接アタッチ
        - AWS 管理ポリシー AmazonEC2FullAccess

入手したアクセスキー ID, シークレットアクセスキーを環境変数 AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEYに設定

bashの場合の例

```
$ cat <<EOC >> ~/.bash_profile
export AWS_ACCESS_KEY_ID="...................."
export AWS_SECRET_ACCESS_KEY="........................................"
EOC
$ . ~/.bash_profile
```


## terraform による providerやネットワーク定義



- .tfフォーマットは他のユーティリティとの連携などが困難
- .tf.jsonフォーマットは人が直接使用するには難しい
- ここでは、.tf.jsonをYAMLで表記したファイルにてec2設定を行い、terraform使用前にjson化
- 最初から.tfフォーマットや.tf.jsonフォーマットで記述、管理するならばYAML管理は不要

- tf_common.tf.yml は、providerやネットワーク定義など個別のインスタンスに1:1で対応しない設定をまとめたもの。

- VPC EC2定義の大部分は以下から引用させていただいています。ありがとうございます。
    - TerraformでVPC・EC2インスタンスを構築してssh接続する
    - https://qiita.com/kou_pg_0131/items/45cdde3d27bd75f1bfd5

```yaml:molecule/default/tf_common.tf.yml
---
terraform:
  required_providers:
    aws:
      source: hashicorp/aws
      version: "~> 3.0"
provider:
  aws:
    region: ap-northeast-1
data:
  aws_ami:
    ami01:
      most_recent: true
      owners:
      - amazon
      filter:
      - name: architecture
        values:
        - x86_64
      - name: root-device-type
        values:
        - ebs
      - name: name
        values:
        - amzn2-ami-hvm-*
      - name: virtualization-type
        values:
        - hvm
      - name: block-device-mapping.volume-type
        values:
        - gp2
      - name: state
        values:
        - available
resource:
  aws_vpc:
    vpc01:
      cidr_block: 10.0.0.0/16
      enable_dns_support: true
      enable_dns_hostnames: true
  aws_subnet:
    subnet01:
      cidr_block: 10.0.1.0/24
      availability_zone: ap-northeast-1a
      vpc_id: "${aws_vpc.vpc01.id}"
      map_public_ip_on_launch: true
  aws_internet_gateway:
    internet_gateway01:
      vpc_id: "${aws_vpc.vpc01.id}"
  aws_route_table:
    route_table01:
      vpc_id: "${aws_vpc.vpc01.id}"
  aws_route:
    route01:
      gateway_id: "${aws_internet_gateway.internet_gateway01.id}"
      route_table_id: "${aws_route_table.route_table01.id}"
      destination_cidr_block: 0.0.0.0/0
  aws_route_table_association:
    route_table_association01:
      subnet_id: "${aws_subnet.subnet01.id}"
      route_table_id: "${aws_route_table.route_table01.id}"
  aws_security_group:
    security_group01:
      name: security_group01
      vpc_id: "${aws_vpc.vpc01.id}"
  aws_security_group_rule:
    in_ssh:
      security_group_id: "${aws_security_group.security_group01.id}"
      type: ingress
      cidr_blocks:
      - 0.0.0.0/0
      from_port: 22
      to_port: 22
      protocol: tcp
    in_icmp:
      security_group_id: "${aws_security_group.security_group01.id}"
      type: ingress
      cidr_blocks:
      - 0.0.0.0/0
      from_port: -1
      to_port: -1
      protocol: icmp
    out_all:
      security_group_id: "${aws_security_group.security_group01.id}"
      type: egress
      cidr_blocks:
      - 0.0.0.0/0
      from_port: 0
      to_port: 0
      protocol: "-1"
  aws_key_pair:
    key_pair01:
      key_name: key_pair01
      public_key: ${file("~/.ssh/id_rsa.pub")}
```

## terraform による インスタンス固有の定義

- インスタンス名.tf.ymlは、インスタンスに1:1で対応する資源をまとめたもの
- インスタンス毎に1ファイルの想定
- j2などテンプレートエンジンを用いての作成は、どの範囲までユーザー指定とするかなど環境、プロジェクト依存の項目が多く、不必要に複雑化する為に断念。シンプルにこのファイル内での直接指定による設定としている。
- 作成したインスタンスへの接続情報はlocal_file:リソースを用いてMoleculeに連携する
- ユーザー利便を考慮してsshログイン方法をoutputにて表示

```yaml:molecule/default/tf_ins01.tf.yml
---
resource:
  aws_instance:
    ins01:
      ami: "${data.aws_ami.ami01.image_id}"
      vpc_security_group_ids:
      - "${aws_security_group.security_group01.id}"
      subnet_id: "${aws_subnet.subnet01.id}"
      key_name: "${aws_key_pair.key_pair01.id}"
      instance_type: t2.micro
  aws_eip:
    eip-ins01:
      instance: "${aws_instance.ins01.id}"
      vpc: true
  local_file:
    instance_conf-ins01:
      filename: "./instance_conf-ins01.yml"
      content: |
        # this file is maintaind by terraform
        ---
        instance: ins01
        address: ${aws_eip.eip-ins01.public_ip}
        user: ec2-user
        port: 22
        identity_file: ~/.ssh/id_rsa
output:
  ssh_login-ins01:
    value: ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa
      ec2-user@${aws_eip.eip-ins01.public_ip}
```

local_fileを使用して、インスタンスへの接続方法をmoleculeに対して連携する

## terraformによるデプロイ、デストロイ確認

### tf.jsonファイル用意

最初から.tf.jsonや.tfファイルを使用しているならば不要
yqユーティリティによりYAMLファイルからterraform用の.tf.jsonファイルに変換

```
$ cd molecule/default/
$ yq . tf_common.tf.yml > tf_common.tf.json
$ yq . tf_ins01.tf.yml > tf_ins01.tf.json
```

yqはMACにデフォルトでは導入されていない。
以下の様にpythonなどでも代替可能。

```
$ cd molecule/default/
$ cat tf_common.tf.yml | python -c "import yaml; import json; import sys; print(json.dumps(yaml.load(sys.stdin, Loader=yaml.FullLoader), indent=2))" > tf_common.tf.json
$ cat tf_ins01.tf.yml | python -c "import yaml; import json; import sys; print(json.dumps(yaml.load(sys.stdin, Loader=yaml.FullLoader), indent=2))" > tf_ins01.tf.json
```


### terraform初期化 (terraform init)

```
$ terraform init

Initializing the backend...

Initializing provider plugins...
- Using previously-installed hashicorp/aws v3.13.0
- Using previously-installed hashicorp/local v1.4.0
...
Terraform has been successfully initialized!
...
```

プロバイダーファイルは.terraform/plugins下に置かれる。
もし~/.terraform.d/pluginsに事前に配置しておくと、.terraform/plugins下はシンボリックリンクとなる。

### AWS資源作成 (terraform apply)

```
$ terraform apply --auto-approve
...

Apply complete! Resources: 14 added, 0 changed, 0 destroyed.

Outputs:

ssh_login-ins01 = ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa ec2-user@52.68.125.189
```

### EC2インスタンスにsshログイン,ログアウト

```
$ ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa ec2-user@52.68.125.189
Warning: Permanently added '52.68.125.189' (ECDSA) to the list of known hosts.

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
25 package(s) needed for security, out of 39 available
Run "sudo yum update" to apply all updates.
[ec2-user@ip-10-0-1-74 ~]$ ^D
ログアウト
Connection to 52.68.125.189 closed.
```

### AWS資源削除 (terraform destroy)

```
$ terraform destroy --auto-approve
...
aws_vpc.vpc01: Destruction complete after 1s

Destroy complete! Resources: 14 destroyed.
```

# CH2. Moleculeとの連携

## Molecule設定ファイル (molecule.yml)

$ molecule init role ROLENAME --driver-name delegated
などにて作成されるmolecule.ymlファイルを編集
platformsのnameをterraform設定ファイルと合わせておく

```yaml:molecule/default/molecule.yml
---
dependency:
  name: galaxy
driver:
  name: delegated
platforms: # terraformにて作成するec2インスタンスと名前が合致する必要がある
  - name: ins01
provisioner:
  name: ansible
verifier:
  name: ansible
```

## Molecule用テスト環境作成playbook (create.yml)

$ molecule init role ROLENAME --driver-name delegated
などにて作成されるcreate.ymlファイルを編集
Dump taskは無変更

```yaml:molecule/default/create.yml
---
- name: Create
  hosts: localhost
  connection: local
  gather_facts: false
  #no_log: "{{ molecule_no_log }}"
  tasks:

  # molecule.ymlとterraform設定の整合性検証用1(オプション)
  - name: set_fact molecule_platforms
    set_fact:
      molecule_platforms: "{{ molecule_yml.platforms|map(attribute='name')|list }}"
  - debug:
      var: molecule_platforms

  # インスタンス作成処理本体(必須)
  - name: terraform apply -auto-approve
    shell: |-
      set -ex
      exec 2>&1
      which yq
      which terraform
      ! ls *.tf.json || rm *.tf.json
      ! ls instance_conf-*.yml || rm instance_conf-*.yml
      ls *.tf.yml | awk '{TO=$1; sub(/.tf.yml$/,".tf.json",TO); print "yq . "$1" > "TO}' | bash -x
      terraform init -no-color
      terraform apply -auto-approve -no-color || terraform apply -auto-approve -no-color
    register: r
  - debug:
      var: r.stdout_lines

  # Moleculeに連携する各インスタンスへの接続情報をlistにまとめる(必須)
  - name: Make instance_conf from terraform localfile
    vars:
      instance_conf: []
    with_fileglob:
    - instance_conf-*.yml
    set_fact:
      instance_conf: "{{ instance_conf + [lookup('file',item)|from_yaml] }}"

  # molecule.ymlとterraform設定の整合性検証用2(オプション)
  - name: set_fact terraform_platforms
    set_fact:
      terraform_platforms: "{{ instance_conf|map(attribute='instance')|list }}"
  - debug:
      var: terraform_platforms
  - name: Check molecule_platforms is included in terraform_platforms
    assert:
      that:
      - "{{ (molecule_platforms|difference(terraform_platforms)) == [] }}"

  # Moleculeに連携する各インスタンスへの接続情報をファイル出力(必須)
  - name: Dump instance config
    copy:
      content: |
        # Molecule managed

        {{ instance_conf | to_json | from_json | to_yaml }}
      dest: "{{ molecule_instance_config }}"
```
## Molecule用テスト環境準備playbook (prepare.yml)

ターゲットシステムにpythonが入っていなければ、rawモジュールを使用してpythonの導入をおこなったりすることに使用可能。
ここでは、接続ができるまで待たせる。
(terraform終了時点でインスタンスは上がっているので、ほとんど待たされないが。)

```yaml:molecule/default/prepare.yml
---
- name: Prepare
  hosts: all
  gather_facts: no
  tasks:
  - wait_for_connection:
```

## Molecule用テスト環境削除playbook (destroy.yml)

$ molecule init role ROLENAME --driver-name delegated
などにて作成されるdestroy.ymlファイルを編集。
Dump taskなどは無変更

```yaml:molecule/default/destroy.yml
---
- name: Destroy
  hosts: localhost
  connection: local
  gather_facts: false
  no_log: "{{ molecule_no_log }}"
  tasks:

  - name: Populate instance config
    shell: |-
      set -ex
      exec 2>&1
      terraform destroy --auto-approve -no-color || terraform destroy --auto-approve -no-color
    register: r
  - debug:
      var: r.stdout_lines

  - name: Populate instance config
    set_fact:
      instance_conf: {}

  - name: Dump instance config
    copy:
      content: |
        # Molecule managed

        {{ instance_conf | to_json | from_json | to_yaml }}
      dest: "{{ molecule_instance_config }}"
```


## テスト対象roleを呼び出すplaybook (converge.yml)

このroleのディレクトリー名変更に追随できる様に修正している。

```yaml:molecule/default/converge.yml
---
- name: Converge
  hosts: all
  tasks:
    - name: "Include {{ playbook_dir|dirname|dirname|basename }}"
      include_role:
        name: "{{ playbook_dir|dirname|dirname|basename }}"
```

## テスト対象roleのtasks/main.yml

ここでは対象インスタンスのOS種類情報を取得表示させる

```
---
- debug:
    var: ansible_distribution
```

## molecule test 実行例

molecule testにて、インスタンス作成、反映、削除まで通しで実行させてみる

```
$ molecule test
--> Test matrix

└── default
    ├── dependency
    ├── lint
    ├── cleanup
    ├── destroy
    ├── syntax
    ├── create
    ├── prepare
    ├── converge
    ├── idempotence
    ├── side_effect
    ├── verify
    ├── cleanup
    └── destroy

--> Scenario: 'default'
--> Action: 'dependency'
Skipping, missing the requirements file.
Skipping, missing the requirements file.
--> Scenario: 'default'
--> Action: 'lint'
--> Lint is disabled.
--> Scenario: 'default'
--> Action: 'cleanup'
Skipping, cleanup playbook not configured.
--> Scenario: 'default'
--> Action: 'destroy'

    PLAY [Destroy] *****************************************************************

    TASK [Populate instance config] ************************************************
    changed: [localhost]

    TASK [debug] *******************************************************************
    ok: [localhost]

    TASK [Populate instance config] ************************************************
    ok: [localhost]

    TASK [Dump instance config] ****************************************************
    changed: [localhost]

    PLAY RECAP *********************************************************************
    localhost                  : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

--> Scenario: 'default'
--> Action: 'syntax'

    playbook: /Users/aa220269/repo/repo-test/cookbooks/molecule-delegated-terraform-ec2/molecule/default/converge.yml
--> Scenario: 'default'
--> Action: 'create'

    PLAY [Create] ******************************************************************

    TASK [set_fact molecule_platforms] *********************************************
    ok: [localhost]

    TASK [debug] *******************************************************************
    ok: [localhost] => {
        "molecule_platforms": [
            "ins01"
        ]
    }

    TASK [terraform apply -auto-approve] *******************************************
    changed: [localhost]

    TASK [debug] *******************************************************************
    ok: [localhost] => {
        "r.stdout_lines": [
            "+ which yq",
            "/Users/aa220269/.pyenv/shims/yq",
            "+ which terraform",
            "/usr/local/bin/terraform",
            "+ ls tf_common.tf.json tf_ins01.tf.json",
            "tf_common.tf.json",
            "tf_ins01.tf.json",
            "+ rm tf_common.tf.json tf_ins01.tf.json",
            "+ ls 'instance_conf-*.yml'",
            "ls: instance_conf-*.yml: No such file or directory",
            "+ ls tf_common.tf.yml tf_ins01.tf.yml",
            "+ awk '{TO=$1; sub(/.tf.yml$/,\".tf.json\",TO); print \"yq . \"$1\" > \"TO}'",
            "+ bash -x",
            "+ yq . tf_common.tf.yml",
            "+ yq . tf_ins01.tf.yml",
            "+ terraform init -no-color",
            "",
            "Initializing the backend...",
            "",
            "Initializing provider plugins...",
            "- Using previously-installed hashicorp/aws v3.13.0",
            "- Using previously-installed hashicorp/local v1.4.0",
            "",
            "The following providers do not have any version constraints in configuration,",
            "so the latest version was installed.",
            "",
            "To prevent automatic upgrades to new major versions that may contain breaking",
            "changes, we recommend adding version constraints in a required_providers block",
            "in your configuration, with the constraint strings suggested below.",
            "",
            "* hashicorp/local: version = \"~> 1.4.0\"",
            "",
            "Terraform has been successfully initialized!",
            "",
            "You may now begin working with Terraform. Try running \"terraform plan\" to see",
            "any changes that are required for your infrastructure. All Terraform commands",
            "should now work.",
            "",
            "If you ever set or change modules or backend configuration for Terraform,",
            "rerun this command to reinitialize your working directory. If you forget, other",
            "commands will detect it and remind you to do so if necessary.",
            "+ terraform apply -auto-approve -no-color",
            "data.aws_ami.ami01: Refreshing state...",
            "aws_vpc.vpc01: Creating...",
            "aws_key_pair.key_pair01: Creating...",
            "aws_key_pair.key_pair01: Creation complete after 1s [id=key_pair01]",
            "aws_vpc.vpc01: Creation complete after 3s [id=vpc-004bac783f79ba162]",
            "aws_route_table.route_table01: Creating...",
            "aws_internet_gateway.internet_gateway01: Creating...",
            "aws_security_group.security_group01: Creating...",
            "aws_subnet.subnet01: Creating...",
            "aws_route_table.route_table01: Creation complete after 0s [id=rtb-01a167c14ee021fe3]",
            "aws_subnet.subnet01: Creation complete after 1s [id=subnet-0ffa370a28910b65a]",
            "aws_route_table_association.route_table_association01: Creating...",
            "aws_internet_gateway.internet_gateway01: Creation complete after 1s [id=igw-0cfeaa28a6412a523]",
            "aws_route.route01: Creating...",
            "aws_route_table_association.route_table_association01: Creation complete after 0s [id=rtbassoc-0695a2e2b04fb4728]",
            "aws_security_group.security_group01: Creation complete after 1s [id=sg-05db55c4377f19c4f]",
            "aws_security_group_rule.out_all: Creating...",
            "aws_security_group_rule.in_icmp: Creating...",
            "aws_security_group_rule.in_ssh: Creating...",
            "aws_instance.ins01: Creating...",
            "aws_route.route01: Creation complete after 1s [id=r-rtb-01a167c14ee021fe31080289494]",
            "aws_security_group_rule.out_all: Creation complete after 1s [id=sgrule-3399627109]",
            "aws_security_group_rule.in_ssh: Creation complete after 2s [id=sgrule-740776232]",
            "aws_security_group_rule.in_icmp: Creation complete after 2s [id=sgrule-1963657664]",
            "aws_instance.ins01: Still creating... [10s elapsed]",
            "aws_instance.ins01: Still creating... [20s elapsed]",
            "aws_instance.ins01: Still creating... [30s elapsed]",
            "aws_instance.ins01: Creation complete after 33s [id=i-0f6d44308394d6b5c]",
            "aws_eip.eip-ins01: Creating...",
            "aws_eip.eip-ins01: Creation complete after 1s [id=eipalloc-0c4b8efe50c27636c]",
            "local_file.instance_conf-ins01: Creating...",
            "local_file.instance_conf-ins01: Creation complete after 0s [id=7d42af9527d38119349dc1f1c15e81e7a9c7e02e]",
            "",
            "Apply complete! Resources: 14 added, 0 changed, 0 destroyed.",
            "",
            "Outputs:",
            "",
            "ssh_login-ins01 = ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa ec2-user@54.249.181.45"
        ]
    }

    TASK [Make instance_conf from terraform localfile] *****************************
    ok: [localhost] => (item=/Users/aa220269/repo/repo-test/cookbooks/molecule-delegated-terraform-ec2/molecule/default/instance_conf-ins01.yml)

    TASK [set_fact terraform_platforms] ********************************************
    ok: [localhost]

    TASK [debug] *******************************************************************
    ok: [localhost] => {
        "terraform_platforms": [
            "ins01"
        ]
    }

    TASK [Check molecule_platforms is included in terraform_platforms] *************
    ok: [localhost] => {
        "changed": false,
        "msg": "All assertions passed"
    }

    TASK [Dump instance config] ****************************************************
    changed: [localhost]

    PLAY RECAP *********************************************************************
    localhost                  : ok=9    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

--> Scenario: 'default'
--> Action: 'prepare'

    PLAY [Prepare] *****************************************************************

    TASK [wait_for_connection] *****************************************************
    ok: [ins01]

    PLAY RECAP *********************************************************************
    ins01                      : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

--> Scenario: 'default'
--> Action: 'converge'

    PLAY [Converge] ****************************************************************

    TASK [Gathering Facts] *********************************************************
[WARNING]: Platform linux on host ins01 is using the discovered Python
interpreter at /usr/bin/python, but future installation of another Python
interpreter could change the meaning of that path. See https://docs.ansible.com
/ansible/2.10/reference_appendices/interpreter_discovery.html for more
information.
    ok: [ins01]

    TASK [Include molecule-delegated-terraform-ec2] ********************************

    TASK [molecule-delegated-terraform-ec2 : debug] ********************************
    ok: [ins01] => {
        "ansible_distribution": "Amazon"
    }

    PLAY RECAP *********************************************************************
    ins01                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

--> Scenario: 'default'
--> Action: 'idempotence'
Idempotence completed successfully.
--> Scenario: 'default'
--> Action: 'side_effect'
Skipping, side effect playbook not configured.
--> Scenario: 'default'
--> Action: 'verify'
--> Running Ansible Verifier

    PLAY [Verify] ******************************************************************

    TASK [Example assertion] *******************************************************
    ok: [ins01] => {
        "changed": false,
        "msg": "All assertions passed"
    }

    PLAY RECAP *********************************************************************
    ins01                      : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

Verifier completed successfully.
--> Scenario: 'default'
--> Action: 'cleanup'
Skipping, cleanup playbook not configured.
--> Scenario: 'default'
--> Action: 'destroy'

    PLAY [Destroy] *****************************************************************

    TASK [Populate instance config] ************************************************
    changed: [localhost]

    TASK [debug] *******************************************************************
    ok: [localhost]

    TASK [Populate instance config] ************************************************
    ok: [localhost]

    TASK [Dump instance config] ****************************************************
    changed: [localhost]

    PLAY RECAP *********************************************************************
    localhost                  : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

--> Pruning extra files from scenario ephemeral directory

```

# 参考

TerraformでVPC・EC2インスタンスを構築してssh接続する
https://qiita.com/kou_pg_0131/items/45cdde3d27bd75f1bfd5

