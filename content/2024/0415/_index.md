+++
title = "Kubernetes Windows and Active Directory"
description = "How to integrate Active Directory on Kubernetes Windows clusters"
date = "2024-04-11"
+++

![tainha](./images/post.png?width=200px "tainha")

## Kubernetes RBAC

### [Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)

Kubernetes clusters have two categories of users: human users who access the cluster (and the goal of this post), ServiceAccounts used for machines (or identities providers like SPIFFE), and Pods. 

The complicated part (when starting on Kubernetes) is the lack of a User object, so there’s no API to add one directly. Users can present (when accessing any api-server API) a certificate authenticated by the cluster CA. Kubernetes reads the subject from the cert (CN field). And uses the RBAC system to analyze its permission aspects. A few types of authentication strategies:

* x509 client certificates - If a client certificate is presented and verified, the common name of the subject is used as the user name for the request, is necessary to create a CSR, and have it accepted and validated manually.
* Service account tokens - are automatically enabled and created by the apiserver. They are associated with pods running in the cluster via the Service Account Admission Controller.
* openID connect tokens - a flavor of Oauth2 supported by OAtuh2 providers, using JWT tokens for authentication.
* Authentication  proxy - can be configured to identify users from request header values.


If you want to check how to create users on Kubernetes here is good example [article](https://cormachogan.com/2020/11/06/creating-developer-users-and-namespaces-scripted-in-tkg-guest-clusters/).

### RBAC Permissions

A Role sets permissions within a specific namespace, and its namespace must be specified. On the other hand, a ClusterRole is a non-namespaced resource with broader actions. 
These differences should be considered when using either. RoleBinding and ClusterRoleBinding grant the permissions defined in the role to a user or group of users. 
These subjects can be users, groups, or service accounts. Additionally, each role can hold one or more subjects.

Lets see in practice:

```shell
# Create a ServiceAccount called reader
$ kubectl create serviceaccount reader

# Create a role allowing Pods to be get into the namespace
$ kubectl create role podsreads --verb=get --resource=pods

# Create the Rolebinding between the Role and ServiceAccount
$ kubectl create rolebinding reader-binding --role=podsreads --serviceaccount=default:reader

# Check the Permission exists for the ServiceAccount
$ kubectl who-can get pods
ROLEBINDING     NAMESPACE  SUBJECT  TYPE            SA-NAMESPACE
reader-binding  default    reader   ServiceAccount  default

# Check using the default auth subcommand 
kubectl auth can-i --as=system:serviceaccount:default:reader get pods
yes
```


## Active Directory

Access management is crucial in various parts of the infrastructure to ensure security and verify the identities of system users. Modern access management, often called Identity and Access Management (IdM), uses the concept of identity to solve computer and user management problems. Identities can be assigned to services or individuals within an organization and are later used for authentication, confirming the service is as it claims. They are also used for authorization, determining if the service has permission to access the requested resource.

The systems implementing these concepts need to be:

1. A directory that stores user identity data
1. A set of tools to provision, modify, and delete user and privileges
1. A service to regular access to privileges using policies and workflows
1. A system for auditing and reporting

A well-known implementation solution is Microsoft Active Directory. Developed by Microsoft for Windows domain networks, this product has been around for about 24 years, first released in the Windows 2000 Server Edition. Although Active Directory is not a full Identity Access Management (IAM) system, it has strong IAM characteristics.

A directory is a hierarchical structure that holds information about objects on a network. Directory services store this data and make it accessible to network users and administrators. In Active Directory's logic structure concepts, we have "forests," which comprise one or more domains and domain trees. Each domain has its unique characteristics, boundaries, and allocated resources. The creation of the first domain initiates a forest, known as the forest root domain.

Domains contain the logical components to achieve administrative goals in the organization; by default, the domain becomes the security boundary for the objects inside it. The security rules defined also control objects in the domain. A domain tree is a collection of domains that reflects the organization’s structure. The child domain is also called a subdomain. One important concept is the domain controller, a server that responds to security authentication requests within the Windows domain.

Organizational Units (OU) are subgroups or sub-organizations within a domain, similar to a business unit. When deploying a domain controller, it creates a default OU structure to segment the most common object types, such as users, computers, and domain controllers. Globally unique identifiers such as GUID and security identifiers such as SID are set for each new object created.

Some other components are important, like the Distinguished names (DN) that are used to define a unique path for an object. 
In the AD schema, there are a lot of attributes to set into the tree in order to segment, organize, and model the actual organization; for example, 
the commonName attribute is used to represent a name,  domainComponent is the naming attribute for Domain and DNS objects. 

```shell
$ ldapsearch -x -b 'dc=hacktheplanet,dc=com' -H ldap://192.168.39.198 -W -D "cn=Administrator,cn=Users,dc=hacktheplanet,dc=com"

# hacktheplanet.com
dn: DC=hacktheplanet,DC=com
objectClass: top
objectClass: domain
objectClass: domainDNS
distinguishedName: DC=hacktheplanet,DC=com
instanceType: 5
whenCreated: 20240410001515.0Z
whenChanged: 20240411231546.0Z
subRefs: DC=ForestDnsZones,DC=hacktheplanet,DC=com
subRefs: DC=DomainDnsZones,DC=hacktheplanet,DC=com
subRefs: CN=Configuration,DC=hacktheplanet,DC=com
uSNCreated: 4099
dSASignature:: AQAAACgAAAAAAAAAAAAAAAAAAAAAAAAAuwQ0ugw8nUmd1H5VUPKyIg==
uSNChanged: 20486
name: hacktheplanet

PS> Get-UserAD Administrator


DistinguishedName : CN=Administrator,CN=Users,DC=hacktheplanet,DC=com
Enabled           : True
GivenName         :
Name              : Administrator
ObjectClass       : user
ObjectGUID        : 510d986d-4588-4cb3-b27d-cdee7dcd2aeb
SamAccountName    : Administrator
SID               : S-1-5-21-1587599777-1881890648-1557951787-500
Surname           :
UserPrincipalName :
```

![adexample](./images/adexample.png?width=800px "adexample")

These are the server roles attached to Active Directory that need to be installed:

* Active Directory Domain Services
* Active Directory Federation Services
* Active Directory Lightweight Directory Services
* Active Directory Rights Management Services
* Active Directory Certificate Services

![adinstall](./images/adinstall.png?width=600px "adinstall")

The AD is a very long topic, and this post does not aim to provide an extensive introduction to it. Check this [book](https://www.amazon.com/Mastering-Active-Directory-protect-Services/dp/1801070393) for full coverage.

### Designing our domain

add a bunch of users for tests
add computers
defining a group managed Services


### Pinniped for authentication in the cluster.

Pinniped installation

Define the settings and LDAP installation 

Connect directly with integrator no DEX

### Using gMSA for Service Account

- gMSA and Active Directory — windows operational readiness

### Creating the Windows Operational Readiness

Command for swdt to run Win opsrdns

- Categories
- What is validated
- Results analysis


## Conclusion
