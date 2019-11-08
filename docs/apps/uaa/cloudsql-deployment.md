# Deploy UAA using CloudSQL

These instructions will walk you through deploying [Cloud Foundry UAA][uaa] as a container.
A [CloudSQL][cloudsql] MySQL database will be used.

## Assumptions

This guide assumes the UAA instance and the CloudSQL database will not be
publicly accessible. VPC networks are required, and the GKE cluster used needs
to be deployed to a VPC subnet. The master node can have a private IP address,
but if it does not, do not forget to add the `--internal-ip` flag when running
`gcloud container cluster get-credentials`.

## 1. Build a Container Image from UAA

1. Download the UAA source:

    ```sh
    git clone https://github.com/cloudfoundry/uaa
    cd uaa
    ```

1. Add a default servlet mapping for Tomcat (this is necessary as the
   Tomcat Cloud Native Buildpack does not include a global web.xml with this
   mapping.):

    ```sh
    cat >uaa/src/main/webapp/WEB-INF/tomcat-web.xml <<EOF
    <?xml version="1.0" encoding="ISO-8859-1"?>
    <web-app xmlns="http://java.sun.com/xml/ns/javaee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd" version="3.0"
      metadata-complete="true">
      <servlet>
            <servlet-name>default</servlet-name>
            <servlet-class>
              org.apache.catalina.servlets.DefaultServlet
            </servlet-class>
            <init-param>
                <param-name>debug</param-name>
                <param-value>0</param-value>
            </init-param>
            <init-param>
                <param-name>listings</param-name>
                <param-value>false</param-value>
            </init-param>
            <load-on-startup>1</load-on-startup>
        </servlet>
    </web-app>
    EOF
    ```

1. Use the following Dockerfile to create a container image for uaa:

    ```sh
    cat >Dockerfile <<EOF
    FROM tomcat:9.0.27-jdk11-openjdk-slim
    RUN rm -fr /usr/local/tomcat/webapps
    ADD uaa/build/libs/cloudfoundry-identity-uaa-0.0.0.war /usr/local/tomcat/webapps/ROOT/uaa.war
    RUN cd /usr/local/tomcat/webapps/ROOT && jar -xvf uaa.war && rm uaa.war
    EOF

    docker build -t gcr.io/[your-project]/uaa .
    docker push gcr.io/[your-project]/uaa
    ```
## 2. Create keys and infrastructure with Terraform

Terraform is used as an example configuration for creating the CloudSQL database
on a private VPC. Keys and passwords are also generated from the Terraform.
The Kf manifest file will be an output from the Terraform.

1. Change directory

Change to the `./kf/docs/apps/uaa` directory

1. Create a service account for applying Terraform

Terraform requires a service account json for authenticating to Google Cloud. The service account will need adequate permissions for creating the resources in the Terraform configuration, which consists of networking resources and CloudSQL resources.

The configuration for this tutorial assumes such a file exists and is named `account.json`.
You will have to either update the Terraform configuration to point to a real service account json or create a symbolic link from `account.json` to a real service account json on your machine.

1. Initialize Terraform

    ```sh
    terraform init
    ```

1. Configure Terraform variables

Update the values in `terraform.tfvars`.

1. Apply the Terraform

    ```sh
    terraform apply
    ```

1. Save the kf manifest file from the Terraform output

    ```sh
    terraform output uaa_manifest > manifest.yml
    ```

## 3. Kf push UAA

1. Deploy (this assumes you've already `kf target`'d a space; see [these
   docs][create-space] for more detail):

    ```sh
    kf push
    ```

1. Use the proxy feature to access the deployed app:

    ```sh
    kf proxy uaa
    ```

## 4. Validate installation

1. Download and install `uaac` via [these instructions](uaac-install)

1. Target `uaa`:

    ```sh
    uaac target http://localhost:8080 --skip-ssl-validation
    ```

1. Get a token for the admin user:

    ```sh
    uaac token client get admin -s [look-in-manifest]
    ```

1. Create a new user:

    ```sh
    uaac user add test -p changeme --emails test@example.com
    ```

1. Validate the user exists:

    ```sh
    uaac users
    ```

## Note: Keys and Certs

All keys and certificates were generated by the Terraform configuration.
However, the service provider certificate is self-signed.
For a production system, you may want to replace this with a certificate pair
which is signed by a trusted certificate authority.
This requires careful configuration of your Kf domains alongside your
configuration with your domain name provider.

[uaa]: https://github.com/cloudfoundry/uaa
[uaac-install]: https://github.com/cloudfoundry/cf-uaac#installation
[create-space]: /docs/install.md#create-and-target-a-space
[create-keys-and-certs]: #create-keys-and-certs
[cloudsql]: https://cloud.google.com/sql/docs/mysql/