If you have been using containers at all, you have probably come to value the encapsulation and consistency that they present. I can build a container and with some level of certainty, download and execute an instance of that container on almost any flavor of Linux that I like. This encapsulation decouples us from the burden of host configuration.

The usefulness of containers aside, there is still a high level of traditional thinking around the deployment and execution of these containers. I believe most engineers, when they first start working with containers, inadvertently apply this traditional model.

Take a simple cloud based multi-tier web application. Generally we will have some sort of data store, like mysql, mongo or some other solution. We have an API tier, and we have a UI tier. In a traditional operational model we might provision a VM for each of these tiers. Each host will have a port binding and reference each other via IP address or DNS.

Now that we have containerized versions of the data, API and UI tiers, what is first thing many engineers want to do? Bind all the container ports to the VM and use those same old IP addresses and DNS references. What’s old is new again! The question in this scenario is, what did containerizing our application buy us? The answer is very little. While this is a completely acceptable solution, it’s not very dynamic, it doesn’t scale easily without intervention. In fact, it’s just not a very good use of cloud resources.

We need to be thinking in a cloud native mindset. How can we leverage the infrastructure as a service and scalability of the cloud. First off, our application components need to be loosely coupled. We need to consider the cloud a volatile place, where servers are created and destroyed on a whim. We don’t want to have a system based on IP addresses that could be repurposed to another virtual machines at any point in time.

We made a big leap forward when we containerized our stack, but that is just the first step. The next question we should be asking is how can we get the containers to communicate effectively. What kind of networking can be applied to deal with the volatility of the cloud environment? Luckily container networking has come a long way in helping us deal with these issues. Let’s attempt to reimagine our multi-tier application with a cloud native approach, more specifically the types of container networking options available to help us.

#Single Host Networking
Lets start with some simple single host networking options. While this isn’t likely to be the final solution in most cases, it does help us to discover which of our containers actually need external network access. Wordpress is a great multi-container example. The official [wordpress](https://hub.docker.com/_/wordpress/) Docker image requires access to a mysql database. For this exercise we are going to use a mysql variation [mariadb](https://hub.docker.com/_/mariadb/) Docker image. Typically, the connection to this datasource is specified via environment variables.

	-e WORDPRESS_DB_HOST=
	-e WORDPRESS_DB_PASSWORD=

So we can start up both containers using the environment variables.

	docker run --name mariadb -e MYSQL_ROOT_PASSWORD=<pass> -p 3306:3306 -d mariadb

	docker run --name wordpress -e WORDPRESS_DB_HOST=<host_ip>:3306 \
		-e WORDPRESS_DB_PASSWORD=<pass> -p 8080:80 -d wordpress

> If you are using boot2docker, <host_ip> is the host of your boot2docker VM. Use boot2docker ip to find this value.

This solution works, but we’ve used up a port on our host that is not used externally. The mariadb port, 3306, is already available via the Docker network interface. However, because we are on the same host, we can take advantage of Docker container linking.

	docker run --name mariadb -e MYSQL_ROOT_PASSWORD=<pass> -d mariadb

	docker run --name wordpress -e WORDPRESS_DB_PASSWORD=<pass> \
		—-link mariadb:mysql -p 8080:80 -d wordpress

At this point the wordpress container has access to the mariadb container instance using local Docker networking. What does strategy give us? It allows us to run as many mariadb instances on the same host as we would like without port collision. However, we still have to run our containers on the same host in this instance.

#Multi-Host Networking with Kubernetes
There are a number of solutions available for multi-host container networking. However, Kubernetes has come up with a nice networking model for container based application deployments. First off, if you are not familiar with Kubernetes, it’s a tool that provides multi-host container scheduling and configuration services. The thinking behind this tool is, “I don’t care where my container is running as long as it is running somewhere.” To accomplish multi-host networking, Kubernetes can utilize a number of overlay network options, flanneld and weave for example. An overlay network is simply a software defined network utilizing any number of host network interfaces.
Kubernetes takes networking one step further with a concept they call pods. A pod is simply a group of containers that represent a tightly coupled application. More importantly, each pod is assigned an IP address and all containers in the pod act as if on the same network interface. In our above example with wordpress, we can easily create pod specification to run our application.

        apiVersion: v1
        kind: ReplicationController
        metadata:
          name: "wordpress-rc"
          namespace: default
        spec:
          replicas: 1
          imagePullPolicy: IfNotPresent
          selector:
            app: "wordpress"
          template:
            metadata:
              labels:
                app: "wordpress"
            spec:
              containers:
              - name: "mariadb"
                image: mariadb:5
                env:
                - name: MYSQL_ROOT_PASSWORD
                  value: password
                ports:
                - containerPort: 3306
              - name: "wordpress"
                image: wordpress
                env:
                - name: WORDPRESS_DB_HOST
                  value: 127.0.0.1
                - name: WORDPRESS_DB_PASSWORD
                  value: password
                ports:
                - containerPort: 80
            restartPolicy: Always

Notice that the WORDPRESS_HOST_DB is set to the localhost loopback address. This is one of the benefits of pod networking. Now we can create a Kubernetes service to bind the wordpress port to all of our host nodes.

        apiVersion: v1
        kind: Service
        metadata:
          name: "wordpress-svc"
          namespace: default
          labels:
            app: "wordpress"
        spec:
          type: NodePort
          ports:
          - port: 8080
            targetPort: 80
            protocol: TCP
            nodePort: 30080
          selector:
            app: "wordpress"

This specific configuration proxies all pod requests on port 8080 to the wordpress container port 80. Additionally, we have created a node port on our Kubernetes host nodes that will proxy port 30080 to our container. This node port is bound on all nodes in the cluster. Meaning you can connect to the service from any node on port 30000. 

Still though, this solution all resides on the same host. Additionally, because the solution is packaged as a single pod, you are not able to scale without creating a new database each time. What if we were to break out the database into an individual pod. Then we could scale the wordpress container to multiple instances across hosts. First we need to create a new replication controller and service for the mariadb instance.

        apiVersion: v1
        kind: ReplicationController
        metadata:
          name: "mariadb-rc"
          namespace: default
        spec:
          replicas: 1
          imagePullPolicy: IfNotPresent
          selector:
            app: "mariadb"
          template:
            metadata:
              labels:
                app: "mariadb"
            spec:
              containers:
              - name: "mariadb"
                image: mariadb:5
                env:
                - name: MYSQL_ROOT_PASSWORD
                  value: password
                ports:
                - containerPort: 3306
            restartPolicy: Always

        apiVersion: v1
        kind: Service
        metadata:
          name: "mariadb-svc"
          namespace: default
          labels:
            app: "mariadb"
        spec:
          type: ClusterIP
          ports:
          - port: 3306
            targetPort: 3306
            protocol: TCP
          clusterIP: 192.168.253.201
          selector:
            app: "mariadb"

Notice the service configuration. This is now type ClusterIP with a hardcoded cluster IP address. This means this service will be available to the cluster at the specified IP address. Now we can create a wordpress replication controller that references the mariadb pod via the cluster IP address.

        apiVersion: v1
        kind: ReplicationController
        metadata:
          name: "wordpress-rc"
          namespace: default
        spec:
          replicas: 2
          imagePullPolicy: IfNotPresent
          selector:
            app: "wordpress"
          template:
            metadata:
              labels:
                app: "wordpress"
            spec:
              containers:
              - name: "wordpress"
                image: wordpress
                env:
                - name: WORDPRESS_DB_HOST
                  value: 192.168.253.201
                - name: WORDPRESS_DB_PASSWORD
                  value: password
                ports:
                - containerPort: 80
            restartPolicy: Always

        apiVersion: v1
        kind: Service
        metadata:
          name: "wordpress-svc"
          namespace: default
          labels:
            app: "wordpress"
        spec:
          type: NodePort
          ports:
          - port: 8080
            targetPort: 80
            protocol: TCP
            nodePort: 30080
          selector:
            app: "wordpress"

With this new wordpress replication controller we can scale the number of wordpress container instance. Now we have a solution that is truly multi-node. The mariadb container can run on any host in the cluster and the wordpress containers can run on any number of hosts in the cluster.

### Next Steps, DNS

Now that we have transformed our deployment into a true multi-host deployment using the overlay networking capabilities of Kubernetes, what’s next. If you are looking at the deployment configuration, you might be thinking, how do we rid ourselves of these cluster IP address. I don’t want to have to assign static IP addresses to my pods, I want to be dynamic and scalable. The answer is DNS. There are several options for DNS services including Kubernetes own Sky DNS integration. However, I’ll be taking a look at DNS services in another post.