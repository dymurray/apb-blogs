# Introduction

In the "Up and running with the OpenShift Ansible Broker" blog (by Jesus Rodriguez), we saw how you can leverage the OpenShift Ansible Broker to easily provision services on OpenShift Container Platform. In this blog, we're going to explore developing an Ansible Playbook Bundle (APB) to manage services.

APBs leverage the power of Ansible to allow you to define your application in the same language you would define your infrastructure. Most commonly, APBs are used to orchestrate pre-certified containers while also using Ansible to provision on and off-platform services while also allowing you to package your application management logic in a container image. With APBs you can define how to install, and uninstall, your application with the flexibility provided by Ansible.

This tutorial will give a user a walk-through on developing APBs to deploy a MediaWiki and PostgreSQL service. Once provisioned, we To simplify this process and focus on just the APB development, we're going to assume that Rocket.Chat and MongoDB are already containerized and can run under the `restricted` Security Context Constraint (SCC) in OpenShift. These container images can be found on RHCC([MediaWiki 1.23](https://access.redhat.com/containers/?tab=overview#/registry.access.redhat.com/openshift3/mediawiki-123) and [PostgreSQL 9.5](https://access.redhat.com/containers/?tab=overview#/registry.access.redhat.com/rhscl/postgresql-95-rhel7)).

# Creating MediaWiki APB
To start we will [install the APB tooling](https://github.com/ansibleplaybookbundle/ansible-playbook-bundle/blob/master/docs/apb_cli.md#installing-the-apb-tool). Then run:
```
apb init mediawiki-apb
```

At this point you will see the following file structure:
```
mediawiki-apb/
├── apb.yml
├── Dockerfile
├── playbooks
│   ├── deprovision.yml
│   └── provision.yml
└── roles
    ├── deprovision-mediawiki-apb
    │   └── tasks
    │       └── main.yml
    └── provision-mediawiki-apb
        └── tasks
            └── main.yml
```

| file/directory                     | description                                                   |
|------------------------------------|---------------------------------------------------------------|
| `apb.yml`                          | The APB spec declaration                                      |
| `Dockerfile`                       | The APBs Dockerfile                                           |
| `playbooks/provision.yml`          | An Ansible Playbook defining the APBs `provision` action      |
| `playbooks/deprovision.yml`        | An Ansible Playbook defining the APBs `deprovision` action    |
| `roles/provision-mediawiki-apb`    | An Ansible Role defining which tasks are run on `provision`   |
| `roles/deprovision-mediawiki-apb`  | An Ansible Role defining which tasks are run on `deprovision` |

## APB Spec
`apb.yml` is the declaration of the APB spec. In here, we will list all relevant application specific information including: OpenShift Service Catalog metadata, all of the APBs `plans`, and input parameters to prompt the user when provisioning. Looking at apb.yml, we are free to edit the default plan that is created by the tooling. Go ahead and edit the file to match below.
```yaml
# apb.yml
version: 1.0
name: mediawiki-apb
description: This APB deploys Mediawiki123.
bindable: False
async: optional
metadata:
  displayName: Mediawiki (APB)
plans:
  - name: default
    description: This plan deploys a Mediawiki instance
    free: True
    metadata: {}
    parameters:
      - name: mediawiki_db_schema
        default: mediawiki
        type: string
        title: Mediawiki DB Schema
        required: True
      - name: mediawiki_site_name
        default: MediaWiki
        type: string
        title: Mediawiki Site Name
        required: True
        updatable: True
      - name: mediawiki_site_lang
        default: en
        type: string
        title: Mediawiki Site Language
        required: True
      - name: mediawiki_admin_user
        default: admin
        type: string
        title: Mediawiki Admin User
        required: True
      - name: mediawiki_admin_pass
        type: string
        title: Mediawiki Admin User Password
        required: True
```
Note the `parameters` section, our MediaWiki container image will expect these parameters as environment variables to be used and configured on startup. Now that we have defined our APBs parameters, we can reference them as environment variables to be injected into the container from the `deploymentConfig`.

## Provisioning
First, we need to edit the provision role for the APB. `apb init` makes this easy by leaving us commented blocks of code that we can work from.

We want three standard resources to be created by our APB: a `service`, a `deploymentConfig`, and a `route`. Start by opening up `roles/provision-mediawiki-apb/tasks/main.yml` and uncommenting the `deploymentConfig` resource so necessary information for attaching a `persistentVolume` to the MediaWiki deployment. Once you're finished editing, your `deploymentConfig` should look like this:

```yaml
- name: create deployment config
  openshift_v1_deployment_config:
    name: mediawiki
    namespace: '{{ namespace }}'
    labels:
      app: mediawiki
      service: mediawiki
    replicas: 1
    selector:
      app: mediawiki
      service: mediawiki
    spec_template_metadata_labels:
      app: mediawiki
      service: mediawiki
    containers:
    - env:
      - name: MEDIAWIKI_DB_SCHEMA
        value: "{{ mediawiki_db_schema }}"
      - name: MEDIAWIKI_SITE_NAME
        value: "{{ mediawiki_site_name }}"
      - name: MEDIAWIKI_SITE_LANG
        value: "{{ mediawiki_site_lang }}"
      - name: MEDIAWIKI_ADMIN_USER
        value: "{{ mediawiki_admin_user }}"
      - name: MEDIAWIKI_ADMIN_PASS
        value: "{{ mediawiki_admin_pass }}"
      - name: MEDIAWIKI_SITE_SERVER
      image: docker.io/ansibleplaybookbundle/mediawiki123:latest
      name: mediawiki
      ports:
      - container_port: 8080
        protocol: TCP
```

The only value we haven't set yet is the `MEDIAWIKI_SITE_SERVER` environment variable. To set this, we need to get the fully qualified route of where the application will exist on OpenShift. Let's uncomment the `route` and `service` resources that were generated and move the `route` before the `deploymentConfig` resource so we can store and use it. Your provision role should now look like this:

```yaml
- name: create mediawiki service
  k8s_v1_service:
    name: mediawiki
    namespace: '{{ namespace }}'
    labels:
      app: mediawiki
      service: mediawiki
    selector:
      app: mediawiki
      service: mediawiki
    ports:
      - name: web
        port: 8080
        target_port: 8080

- name: create mediawiki route
  openshift_v1_route:
    name: mediawiki
    namespace: '{{ namespace }}'
    spec_port_target_port: web
    labels:
      app: mediawiki
      service: mediawiki
    to_name: mediawiki
    state: present
  register: route

- name: create deployment config
  openshift_v1_deployment_config:
    name: mediawiki
    namespace: '{{ namespace }}'
    labels:
      app: mediawiki
      service: mediawiki
    replicas: 1
    selector:
      app: mediawiki
      service: mediawiki
    spec_template_metadata_labels:
      app: mediawiki
      service: mediawiki
    containers:
    - env:
      - name: MEDIAWIKI_DB_SCHEMA
        value: "{{ mediawiki_db_schema }}"
      - name: MEDIAWIKI_SITE_NAME
        value: "{{ mediawiki_site_name }}"
      - name: MEDIAWIKI_SITE_LANG
        value: "{{ mediawiki_site_lang }}"
      - name: MEDIAWIKI_ADMIN_USER
        value: "{{ mediawiki_admin_user }}"
      - name: MEDIAWIKI_ADMIN_PASS
        value: "{{ mediawiki_admin_pass }}"
      - name: MEDIAWIKI_SITE_SERVER
        value: '{{ route.route.spec.host }}'
      image: docker.io/ansibleplaybookbundle/mediawiki123:latest
      name: mediawiki
      ports:
      - container_port: 8080
        protocol: TCP
```

Note that we used `register` when we created the route to store the output of the `route` creation. We then use this output as the value for the `MEDIAWIKI_SITE_SERVER` environment variable as `{{ route.route.spec.host }}`.

## Deprovisioning
By default we recommend that an APB author provide a basic deprovision role. This allows a service consumer to delete the service from the OpenShift Service Catalog and the APBs corresponding resources to be properly deleted. `apb init` provides a skeleton deprovision role with commented resources just like the provision role. Uncomment the `service`, `route`, and `deploymentconfig` resources in the deprovision task like so:

```yaml
- openshift_v1_route:
    name: mediawiki
    namespace: '{{ namespace }}'
    state: absent

- k8s_v1_service:
    name: mediawiki
    namespace: '{{ namespace }}'
    state: absent

- openshift_v1_deployment_config:
    name: mediawiki
    namespace: '{{ namespace }}'
    state: absent
```
This will properly delete all created resources in the APBs namespace.

## Building and Testing
Now that our APB is ready to be tested, we can build and push the image to be deployed by the OpenShift Ansible Broker. For this tutorial, I am assuming your Broker is configured to source APBs from the internal OpenShift registry (Please [refer to this guide](https://github.com/openshift/ansible-service-broker/blob/master/docs/config.md#local-openshift-registry) to configure the OpenShift Ansible Broker with the `local_openshift` registry adapter).

To push your image onto the OpenShift registry, type:
```
apb push --openshift
```

This command will automatically tag & build your image and push it to the internal OpenShift registry. It will then bootstrap the broker and relist the service catalog. Refreshing the webUI will display your new APB in the console. You can now deploy your application by clicking on it and filling out the respective parameters to deploy MediaWiki.

# Binding to PostgreSQL
Now that we have a basic MediaWiki APB developed. We want to be able to bind it to a database. To do this lets create a basic PostgreSQL APB. I will not rehash all of the previous steps here but instead simply paste the APB spec and provision role.

## APB Spec
```yaml
version: 1.0
name: postgresql-apb
description: RHSCL PostgreSQL APB
bindable: true
async: optional
metadata:
  displayName: PostgreSQL (APB)
plans:
  - name: default
    description: A single DB server with no persistent storage
    free: true
    metadata:
      displayName: Default
    parameters:
      - name: postgresql_database
        default: admin
        type: string
        title: PostgreSQL Database Name
        required: true
      - name: postgresql_password
        type: string
        description: A random alphanumeric string if left blank
        title: PostgreSQL Password
      - name: postgresql_user
        default: admin
        title: PostgreSQL User
        type: string
        maxlength: 63
        required: true
```
Note how we set `bindable:` to `true`. This means that the Broker will know that our application can be bound to and we will need to provide the binding credentials at the end of our provision role.

## Provision Role
```yaml
- name: Create postgresql service
  k8s_v1_service:
    name: postgresql
    namespace: '{{ namespace }}'
    labels:
      app: postgresql
      service: postgresql
    selector:
      app: postgresql
      service: postgresql
    ports:
    - name: port-5432
      port: 5432
      protocol: TCP
      target_port: 5432
  register: postgres_service

- name: Create postgresql deployment config
  openshift_v1_deployment_config:
    name: postgresql
    namespace: '{{ namespace }}'
    labels:
      app: postgresql
      service: postgresql
    replicas: 1
    selector:
      app: postgresql
      service: postgresql
    spec_template_metadata_labels:
      app: postgresql
      service: postgresql
    containers:
    - env:
      - name: POSTGRESQL_PASSWORD
        value: '{{ postgresql_password }}'
      - name: POSTGRESQL_USER
        value: '{{ postgresql_user }}'
      - name: POSTGRESQL_DATABASE
        value: '{{ postgresql_database }}'
      image: registry.access.redhat.com/rhscl/posgresql-95-rhel7
      name: postgresql
      ports:
      - container_port: 5432
        protocol: TCP
      termination_message_path: /dev/termination-log
      working_dir: /
    triggers:
    - type: ConfigChange

- name: Wait for postgres to come up
  wait_for:
    port: 5432
    host: "{{ postgres_service.service.spec.cluster_ip }}"
    timeout: 300

- name: encode bind credentials
  asb_encode_binding:
    fields:
      DB_TYPE: postgres
      DB_HOST: postgresql
      DB_PORT: "5432"
      DB_USER: "{{ postgresql_user }}"
      DB_PASSWORD: "{{ postgresql_password }}"
      DB_NAME: "{{ postgresql_database }}"
```
The important things to note in the provision role is the final task `asb_encode_binding`. This is used on bindable applications to expose the secret fields to the broker. These variables will be injected as environment variables to the MediaWiki application container once bound. Also note that we have no need for a route in this instance, so we only create a service resource.

We will now push this image to the internal OpenShift registry:
```
apb push --openshift
```

You can now successfully bind PostgreSQL to MediaWiki in the web console.

# Troubleshooting

## APB not displaying in the web console
If you do not see your APB in the OpenShift web console after `apb push --openshift`, a good way to debug is to type:
```
apb list
```

Look at the output and check if your APB is listed. If it is, that means the OpenShift Ansible Broker knows about your APB and it is part of it's list of bootstrapped APB specs. If it does not show up then try running:
```
apb bootstrap
```

If the bootstrap is successful, and you now see your APB but you do not see it in the web console then try typing:
```
apb relist
```

This will trigger the service catalog to get full list of bootstrapped APB specs.

# Conclusion

At the end of this tutorial, you should be able to fully bind and provision the MediaWiki/PostgreSQL APBs from the web console. I hope this gives a better understanding of building an APB from the ground up and using the APB tooling to make it easier for a developer to create and test APBs.
