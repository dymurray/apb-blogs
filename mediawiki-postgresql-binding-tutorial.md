# Introduction

This tutorial will give a user a walk-through on developing an APB for both MediaWiki and PostgreSQL and make them capable of binding to each other so that MediaWiki is backed by a database. In order to do this we must first make some assumptions. We are assuming that MediaWiki and PostgreSQL are already containerized applications which can run under the `restricted` scc in OpenShift. These images exist on Dockerhub ([mediawiki123](https://hub.docker.com/r/ansibleplaybookbundle/mediawiki123/)) and the Red Hat Container Catalog ([PostgreSQL 9.5](https://registry.access.redhat.com/rhscl/postgresql-95-rhel7)).

# Creating MediaWiki APB
To get started we will need to install the APB tooling. Please go [here](https://github.com/ansibleplaybookbundle/ansible-playbook-bundle/blob/master/docs/apb_cli.md#installing-the-apb-tool) to learn how to install the tool. Once installed, we will run 

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

## APB Spec
Looking at apb.yml, we are free to edit the default plan that is created by the tooling. Go ahead and edit the file to match below.
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
The important thing to note here are the `parameters`. Our Mediawiki container image will expect these parameters as environment variables to be used and configured on startup. Now that we have defined our APBs parameters, we can reference them as environment variables to be injected into the container from the `deploymentConfig`. This will be made obvious in the next section.

## Provisioning
The first thing we want to do is to edit the provision role for the APB. Thankfully `apb init` makes this easy by leaving us commented blocks of code we can simply uncomment. We want to uncomment the three standard resources that should be created by an APB. A `service`, a `deploymentConfig`, and a `route`. The first thing we will edit is the `deploymentConfig`. Open up `roles/provision-mediawiki-apb/tasks/main.yml` and uncomment the `deploymentConfig` resource so it looks like the following:

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

In the `deploymentConfig` you'll see that we have set the value of the environment variables that the app container expects to the parameter fields that we defined in `apb.yml`. The only value we have not set yet is the `MEDIAWIKI_SITE_SERVER` environment variable. To set this, we need to get the fully qualified route of where the application will exist on OpenShift. To get this, lets uncomment the `route` and `service` resources that were generated and move them before the `deploymentConfig` resource so that we can store the route and use it. Your provision role should now look like:

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

What's important to note here is that we used the `register` portion of the Ansible module when we created the route to store the output of the `route` creation. This output is then used as the value for the `MEDIAWIKI_SITE_SERVER` environment variable as `{{ route.route.spec.host }}`.

## Deprovisioning
By default we recommend that an APB author provides a basic deprovisioning role. This way a user can delete the service from the WebUI and all of the APBs corresponding resources are properly deleted. `apb init` provides a skeleton deprovision role with commented resources just like the provision role. Uncomment the `service`, `route`, and `deploymentconfig` resources in the deprovision task like so:

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
Now that our APB is ready to be tested, we can build and push the image so that it can be deployed by the OpenShift Ansible Broker. For this tutorial I am assuming your Ansible Broker is configured to source APBs from the internal OpenShift Registry. Please see [here](https://github.com/openshift/ansible-service-broker/blob/master/docs/config.md#local-openshift-registry) to configure the broker with the `local_openshift` registry adapter.

To push your image onto the OpenShift registry, type:
```
apb push --openshift
```

This command will automatically tag & build your image and push it to the internal OpenShift registry. It will then bootstrap the broker and relist the service catalog. Refreshing the webUI will display your new APB in the console. You can now deploy your application by clicking on it and filling out the respective parameters to deploy Mediawiki.

# Binding to PostgreSQL
Now that we have a basic Mediawiki APB developed. We want to be able to bind it to a database. To do this lets create a basic PostgreSQL APB. I will not rehash all of the previous steps here but instead simply paste the APB spec and provision role.

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
Note how we set `bindable:` to `true`. This means that the broker will know that our application can be bound to and we will need to provide the binding credentials at the end of our provision role.

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
The important things to note in the provision role is the final task `asb_encode_binding`. This is used on bindable applications to expose the secret fields to the broker. These variables will be injected as environment variables to the Mediawiki application container once bound. Also note that we have no need for a route in this instance, so we only create a service resource.

We will now push this image to the internal OpenShift registry:
```
apb push --openshift
```

You can now successfully bind PostgreSQL to Mediawiki in the web console.

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

At the end of this tutorial, you should be able to fully bind and provision the Mediawiki/PostgreSQL APBs from the web console. I hope this gives a better understanding of building an APB from the ground up and using the APB tooling to make it easier for a developer to create and test APBs.
