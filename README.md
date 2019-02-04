# ansible-cortex

## Description

Deploys and configures [Cortex](https://github.com/TheHive-Project/Cortex), an open source
observable analyzer that integrates closely with [TheHive](https://thehive-project.org)
open source incident response tool. It installs based off the RPM and can
optionally pre-seed the ElasticSearch index, eliminating some of the annoying
manual steps in getting Cortex running.

You will need to install ElasticSearch separately. The role is tested with the
[Elastic-provided ansible role](https://github.com/elastic/ansible-elasticsearch).
Sample configuration is included in the documentation.

This should cover most use cases of Cortex, but PRs and suggested improvements
are welcome.

## Requirements

* Ansible 2.0 or higher
* CentOS 7
* ElasticSearch 5.x

## Usage

This is recommended to be installed on a dedicated server, though both ElasticSearch
and TheHive can safely be installed together with Cortex. An optional Nginx proxy
is enabled by default, and support is available for Vouch and LDAP authentication.
If using delegated authentication, it is important to correctly set a seed user
that you can log in as.

ElasticSearch must be installed and running already. This role is tested using the
ansible-elasticsearch role, which can be imported from Ansible Galaxy.

The following vars are recommended:

```yaml
es_instance_name: "thehive"
es_version: 5.6.14
es_major_version: 5.x
es_data_dirs:
  - "/data/es"
es_config:
  node.name: "thehive"
  cluster.name: "thehive"
  node.data: true
  node.master: true
  script.inline: on
  thread_pool.index.queue_size: 100000
  thread_pool.search.queue_size: 100000
  thread_pool.bulk.queue_size: 100000
es_scripts: true
es_templates: false
es_version_lock: false
es_heap_size: 1g
es_xpack_features: ["alerting","monitoring"]
```

Note that ElasticSearch 6.x is not supported by Cortex. Currently the master
branch of the ansible-elasticsearch module supports 5.x.

The following vars must be set at a minimum:

* cortex_url (fqdn where cortex will be accessible)
* cortex_crypto_secret (see `defaults/main.yml` for instructions on how to generate this)

A sample common configuration that automatically seeds TheHive and uses LDAP authentication
and Cortex is included below:

```yaml
cortex_url: "cortex.corp"
cortex_seed_initial_username: "admin"

cortex_http_addr: "127.0.0.1"

cortex_crypto_secret: "..."

cortex_auth_ldap:
  enabled: true
  servers: ["ldapserver.corp:636"]
  use_ssl: true
  bind_dn: "bind_dn"
  bind_pw: "bind_pw"
  search_base: "dc=corp"
  username_attribute: "sAMAccountName"
}


```

## Vouch Authentication
This role supports authentication through a Vouch (formerly known as Lasso) proxy.
This allows you to do OAUTH authentication through providers such as Okta.

When using Vouch, it is critical to set ```cortex_http_addr``` to 127.0.0.1.
Because Vouch uses cookies to communicate authentication information back to the
application, you must place both your Vouch proxy and Cortex site under a common
domain name (e.g., vouch.corp and cortex.corp).

## Role Variables

```yaml
# Whether or not the TheHive RPM repo should be installed.
# This is generally what you want, unless you are using your own RPM repo.
cortex_install_repo: true

# Cortex version to lock and install
cortex_version: 2.1.3

# Note that the mappings and seed data are dependent on the schema version.
# If you are installing a version of Cortex that uses a different index name,
# the mappings and data files need to be updated.
cortex_index: cortex_2

# Cortex URL
cortex_url: localhost

# Wheteher or not an nginx instance should be installed as a proxy
cortex_install_nginx: true

# Whether or not to configure nginx proxy
cortex_configure_nginx: true

# Referenced files will be included in each nginx server config
cortex_nginx_includes: []

# Optionally use SSL with Nginx
cortex_nginx_ssl:
  enabled: false
  certificate: ""
  key: ""
  #cabundle: provide if using a bundle

# Defines where analyzers will be found.
cortex_analyzer_paths: ["/opt/cortex_analyzers/analyzers"]

# Whether or not the Cortex-Analyzers project will be checked out
cortex_install_public_analyzers: true

# Where public repo will be cloned
cortex_public_repo_path: "/opt/cortex_analyzers"

# All analyzers are available to Cortex but the lists below will cause
# ansible to process the requirements. Nearly all analyzers have requirements,
# so this is essential for them to work. Unfortuantely, some analyzers use
# python3 and some use python2 (explicitly), so identify which list to add
# your analyzer to.
# Only Cortex-Analyzer modules will be configured with these vars.
cortex_python2_analyzers: []
cortex_python3_analyzers: []

# The port Cortex will listen to. This var can be changed even when using
# the nginx proxy.
cortex_http_port: 9001

# IP address Cortex should bind to. In general, this can be left as is. However,
# this must be set to 127.0.0.1 when authenticating through a proxy
cortex_http_addr: "0.0.0.0"


# Set this. Generate a key like this:
# cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 64 | head -n 1
cortex_secret: ""

# By default, Cortex requires manual steps to configure.
# You can optionally load a pre-configured mapping and seed data, which makes
# Cortex immediately usable out of the box.
cortex_load_seed_data: true

# Name of the initial user to create. Note if you are using vouch or LDAP for
# authentication, you must set this to a valid username in your directory.
# Cortex does not create users on first logon.
cortex_seed_initial_username: "admin"

# Set this to automatically create an API key for TheHive to use. This results
# in Cortex being ready out of the box for TheHive integration.
# Generate a key like this:
# cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1
cortex_api_key: ""

# Optionally use Vouch authentication (e.g., for Google Authentication, Okta, etc)
cortex_auth_vouch:
  enabled: false
  url: ""
  logon_header: THEHIVE_USER

# Optionally use LDAP authentication.
cortex_auth_ldap:
  enabled: false
  servers: []
  use_ssl: ""
  bind_dn: ""
  bind_pw: ""
  search_base: ""
  username_attribute: "cn"

# ElasticSearch configuration. If using recommended ES configuration, this
# does not need to be changed.
cortex_es:
  index: cortex
  cluster: thehive
  endpoint: 127.0.0.1:9300

# Packages that will be installed with Cortex
cortex_packages:
  - java-1.8.0-openjdk
  - python-pip
  - python-setuptools
  - unzip
  - git
  - python-devel
  - python36-pip
  - python36-devel
  - python36-setuptools
  - ssdeep
  - cortex-{{ cortex_version }}

# Packages that will be installed if the nginx proxy is used.
# libsemanage-python is necessary for selinux.
cortex_nginx_packages:
  - nginx
  - libsemanage-python
```
