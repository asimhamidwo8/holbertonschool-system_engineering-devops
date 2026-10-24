# Web Infrastructure Design

## 0. Simple web stack
Diagram: https://i.imgur.com/FHJrhzA.png

### Request flow
A user types `www.foobar.com` in the browser. DNS returns the IP `8.8.8.8`. The browser sends an HTTP/HTTPS request to the server. Nginx receives it, serves static files, and forwards dynamic requests to the application server. The application server runs the code and reads/writes data in MySQL. The response goes back to the user.

### Specifics
- **Server:** A physical or virtual computer that provides services to other computers (clients) over a network.
- **Domain name:** A human-readable name (`foobar.com`) that DNS translates to an IP address.
- **`www` record type:** An A record. It maps `www.foobar.com` to the IPv4 address `8.8.8.8`.
- **Web server (Nginx):** Receives HTTP requests, serves static content, and forwards dynamic requests to the application server.
- **Application server:** Runs the application code (business logic) and generates dynamic content.
- **Database (MySQL):** Stores and manages the website data.
- **Communication:** TCP/IP over the internet, using HTTP (port 80) or HTTPS (port 443).

### Issues
- **SPOF:** Only one server. If it fails, the website is down.
- **Downtime during maintenance:** Deploying code or restarting the web server makes the site unavailable.
- **Cannot scale:** One server has limited resources and cannot handle too much traffic.

## 1. Distributed web infrastructure
Diagram: https://i.imgur.com/3Zb6eRE.png

### Why each element
- **Load balancer (HAProxy):** Distributes traffic between two servers and keeps the site online if one server fails.
- **Second server:** Removes the single server SPOF and increases capacity.
- **Primary-Replica database:** Keeps a copy of the data for redundancy and shares the read load.

### Specifics
- **Distribution algorithm:** Round Robin. Each new request goes to the next server in order, so each server gets an equal share.
- **Active-Active vs Active-Passive:** In Active-Active, all servers handle traffic at the same time (our setup). In Active-Passive, one server handles traffic and the other waits on standby and takes over only if the active one fails.
- **Primary-Replica cluster:** The Primary receives all writes. Every change is copied automatically to the Replica, so it always has the same data.
- **Primary vs Replica for the application:** The Primary accepts reads and writes. The Replica is read-only.

### Issues
- **SPOF:** The load balancer (only one) and the Primary database (only one for writes).
- **Security:** No firewall (all ports are exposed) and no HTTPS (traffic is plain text).
- **No monitoring:** We cannot detect if a server is down, slow, or overloaded.

## 2. Secured and monitored web infrastructure
Diagram: https://i.imgur.com/BFweTVc.png

### Why each element
- **3 firewalls:** Protect each server by allowing only the needed traffic.
- **SSL certificate:** Serves `www.foobar.com` over HTTPS with encrypted traffic.
- **3 monitoring clients:** Collect metrics and logs from each server and send them to Sumo Logic.

### Specifics
- **Firewalls:** Filter network traffic using rules. For example, only ports 80/443 are open to the internet, and MySQL (3306) accepts only the application servers.
- **HTTPS:** Encrypts data between the browser and the server, so attackers cannot read or change it. It also proves the site identity.
- **Monitoring:** Tracks health, performance, and availability, and sends alerts when something goes wrong.
- **How data is collected:** An agent on each server reads system metrics and log files and sends them to the monitoring service.
- **Monitoring QPS:** Read the Nginx access log or enable the Nginx `stub_status` page. The tool counts requests over time, calculates requests per second, and shows it on a dashboard with alerts.

### Issues
- **SSL terminated at the load balancer:** Traffic between the load balancer and the servers is not encrypted, so it can be read inside the network.
- **Only one MySQL server accepts writes:** It is a SPOF for writes and limits write performance.
- **Same components on every server:** Components compete for CPU, RAM, and disk, and cannot be scaled separately.

## 3. Scale up
Diagram: https://i.imgur.com/fPwJMmJ.png

### Why each element
- **Second load balancer (cluster):** Removes the load balancer SPOF. The two HAProxy servers check each other with a heartbeat, and one takes over if the other fails.
- **Split components on their own servers:**
  - **Web server (Nginx):** Handles HTTP requests and static content.
  - **Application server:** Runs the code and business logic.
  - **Database server (MySQL):** Stores and manages the data.
- **Benefits of splitting:** No resource competition, independent scaling, easier maintenance, and better security (the database is isolated from the internet).
