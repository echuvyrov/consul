## Modern Service Discovery with Consul on Azure - Part 1

This two-part article introduces you to Consul, a modern service discovery system from HashiCorp. In the first part, we will frame the problem that Consul was designed to solve and will go over basic architectural principles underlying the system. Then, we will show you how you can deploy Consul on Azure and conclude with enterprise features of Consul and pointers to further technical information about the product. In the second part, we will dive into features introduced in Consul v1.2, which turned Consul into a full-blown service mesh solution.

### Why Consul: Modern Applications Discovery Challenges

Continuing evolution towards widely distributed software systems and microservices-based architectures brings with it multiple interesting challenges, with one of the main ones being “how do you keep track of all your deployed services?” For example, how does Service A deployed into the cloud (web service) knows how to find Service B (a database)?  In the pre-digital transformation world, you would hard-code the address of Service B into Service A’s configuration, and would need to redeploy Service A if Service B moves to a different server or datacenter. However, the downtime associated with identifying and resolving issues associated with service relocation is not acceptable in today’s rapidly changing business landscape. This is where Consul deployed onto a modern cloud platform like Microsoft Azure can help.

### Intro to Consul
Consul is a distributed, highly available service discovery and service mesh system that can run atop of your infrastructure in Azure, including in multiple regions around the world. In addition to allowing services to discover each other, Consul allows you to monitor cluster health with builtin health checks, as well as serves as a distributed Key-Value store that you can utilize for a number of purposes. The basic building blocks of Consul are <i>agents</i> and <i>servers</i>. Agents allow services to announce their existence, while Consul servers collect the information shared by the agents and serve as a central repository of information about the services. Agents run on the nodes providing services and are responsible for health checking the service, as well as the node itself. To discover services, you can query either the agent or the server, with agents forwarding queries to servers automatically. You use DNS or HTTP for discovery.

Consul is an executable binary that you can install on your local machine - [find the appropriate package](https://www.consul.io/downloads.html) for your operating system, download it and run it.  The same Consul binary runs either in client or server mode, depending on the parameters passed in. Production-ready Consul would normally be deployed on top of global infrastructure, such as Microsoft Azure, and, in partnership with HashiCorp, we have created a set of scripts that automate the installation of Consul in either single or multiple data centers on Azure. We will go over the installation instructions on Azure shortly.

### Consul Architecture
Consul is built upon modern distributed systems principles backed by leading academic research. For discovery, Consul uses [Serf](http://serf.io) implementation of the <I>gossip</I> protocol, which allows nodes in the cluster to automatically discover servers, without resorting to manual configuration of individual clients. In addition, the work of discovering node failures is distributed instead of centralized around the servers. 

Not all servers in Consul world are equal - there’s a leader server that processes all service discovery queries. Consul implements leader server election using Raft protocol, which is a type of consensus protocol described in detail on the [Consul internals site](https://www.consul.io/docs/internals/consensus.html).

The diagram below shows how Consul is able to scale to modern-day workloads across multiple data centers. There are 3 core concepts, some of which we have briefly covered above, that enable service discovery in a distributed environment like Microsoft Azure. Those components are (1) LAN Gossip, (2) Leader Election and (3) WAN Gossip protocol. Let’s briefly examine how these concepts complement each other in Consul.

![Consul Architecture](https://github.com/echuvyrov/consul/blob/master/architectureazure.png)

The recommended pool of Consul servers should be between three and five nodes. After you deploy Consul servers inside a data center, LAN gossip protocol allows them to be automatically discovered by the clients, significantly simplifying cluster configuration. Next, as part of consensus protocol (Raft in Consul), a leader is elected among the servers deployed. The leader becomes responsible for fulfilling all the queries from the clients. Should a leader server go down or become unreachable, a new leader will be elected without disrupting service discovery.  Finally, WAN pool allows servers from different data centers (Azure regions) to process cross-datacenter requests. Unlike LAN pool, WAN pool is optimized for the higher latency of the internet and is expected to contain only other Consul server nodes.

By now, hopefully you are starting to understand how Consul is able to meet dynamic modern applications service discovery requirements. Now, let’s see how to install and run Consul on Azure.

### Consul on Azure
You can install either a single or a multi-datacenter Consul on Azure by executing a set of Terraform templates found in HashiCorp Guides repo on Github: 

  https://github.com/hashicorp-guides/azure-consul (https://github.com/hashicorp-guides/azure-consul)

If you’re not familiar with Terraform, it’s a very popular DevOps tool from HashiCorp that allows you to easily provision infrastructure into Azure. You can learn more about Terraform in Azure on [docs.microsoft.com](http://docs.microsoft.com/azure/virtual-machines/terraform-install-configure). You should install and configure Terraform for Azure before proceeding with the installation of Consul.

Go ahead and clone the repo with Consul provisioning scripts referenced above. After you create security credentials for Terraform in Azure ([as described here](http://docs.microsoft.com/azure/virtual-machines/terraform-install-configure)), you can follow the instructions in the repo [you just cloned](https://github.com/hashicorp-guides/azure-consul). These instructions can be broken down into two parts: the first part creates a read-only security mechanism that allows Consul agents to auto-join the Consul cluster; the second actually installs Consul. For convenience, a very abbreviated version of those steps have been copied below:

1. Create a read-only Azure Service Principal (using the Azure CLI) that will be used to perform the Consul [auto-join](https://www.consul.io/docs/agent/options.html#microsoft-azure) (make note of these values as you will use them later in this guide):

```
$ az ad sp create-for-rbac --role="Reader" --scopes="/subscriptions/[YOUR_SUBSCRIPTION_ID]"
```

2. Create `terraform.tfvars` file,  providing the following values obtained in Step 1 above: `auto_join_subscription_id`, `auto_join_client_id`, `auto_join_client_secret`, `auto_join_tenant_id`. Then, run `terraform init`, `terraform plan` and `terraform apply` to deploy Consul cluster.

3. Once your clusters have been provisioned you can ssh into one (or all) of them to check your cluster status:
◦	consul members [-wan]

If you see a list of consul member nodes as output, you have successfully provisioned Consul cluster in Azure.

### Automated Upgrades, Advanced Federation and Service Mesh
With Consul up and running, your services in Azure can now register themselves, as well as perform service lookup in a dynamic, highly resilient fashion. Sometimes, however, organizations may desire that not all of their data centers be connected to each other in Consul; instead, they might prefer a hub-and-spoke system of WAN gossip protocol, where hub data centers can talk to other hub data centers, but spoke data centers cannot talk to other data centers. Although beyond the scope of this blog post, [Advanced Federation](https://www.consul.io/docs/enterprise/federation/index.html) in Consul allows for such configuration. 

Additionally, [Automated Upgrades](https://www.consul.io/docs/enterprise/upgrades/index.html) allow Consul servers to upgrade to newer versions in an intelligent manner: their votes in consensus protocol count only if a number of servers with newer version of Consul is equal or greater to the number of servers running the older version of Consul. There are [other features](https://www.consul.io/docs/enterprise/index.html) that might be required by larger enterprises as well.

Finally, with the release of Consul 1.2, HashiCorp has introduced a feature called Connect, which turns Consul into a complete Service Mesh solution. We will take a look at the Connect feature of Consul and working with it in Azure in Part 2 of this article.

### Conclusion and Further Info
You now have the basic understanding of what Consul does, how it does it and how you can make it work for you in Azure. To learn more about Consul, visit the following links and also take a look at Part 2 of this article, which will explore Consul as an enterprise service mesh solution.

Consul Enterprise landing page: [https://www.consul.io/docs/enterprise/index.html](https://www.consul.io/docs/enterprise/index.html)

HashiCorp Consul Blog: [https://www.hashicorp.com/blog/category/consul](https://www.hashicorp.com/blog/category/consul)


