# ⚡ Spark Cluster using Raspberry Pi

![Apache Spark](https://img.shields.io/badge/Apache_Spark-Cluster_Deploy-orange?logo=apachespark)
![Docker](https://img.shields.io/badge/Docker-Orchestration-blue?logo=docker)
![Raspberry Pi](https://img.shields.io/badge/Hardware-Raspberry_Pi-red?logo=raspberrypi)
![JupyterLab](https://img.shields.io/badge/Notebook-JupyterLab-yellow?logo=jupyter)
![License](https://img.shields.io/badge/License-MIT-green)

A complete hands-on guide to deploy an **Apache Spark Cluster** on multiple **Raspberry Pi** devices using **Docker Swarm** and **GlusterFS** for distributed persistence — and to run **PySpark analytics** via **JupyterLab**.

---

## 📘 Objective

- Build a distributed big data cluster with Raspberry Pis.  
- Set up Docker Swarm for container orchestration.  
- Configure GlusterFS as a shared persistent volume for Spark.  
- Run Spark Master & Workers for distributed computation.  
- Connect and analyze data interactively using JupyterLab.  
- Use the **MovieLens dataset** for demonstration.

---

## ⚙️ Hardware & Software Requirements

| Component | Description |
|------------|-------------|
| Raspberry Pi | 4 nodes (Model 3B+ or above recommended) |
| OS | Raspberry Pi OS 64-bit |
| Network | Common LAN/Wi-Fi |
| Storage | ≥16 GB microSD each |
| Tools | SSH enabled, Docker, Python3, Spark ARM image |

---

## 🧩 Step 1: Install OS on Raspberry Pi

1. Install **Raspberry Pi Imager** and flash Raspberry Pi OS 64-bit.  
2. Boot and SSH into each Pi:
   ```bash
   ssh pi@<raspberry_pi_ip>
   ```
3. Configure Pi:
   ```bash
   sudo raspi-config
   ```
   - Enable **VNC** in Interface Options  
   - Expand Filesystem → Advanced Options → A1 → Enable  
   - Save and reboot:
     ```bash
     sudo reboot
     ```
4. Update system:
   ```bash
   sudo apt-get update && sudo apt-get upgrade -y
   ```

---

## 🐳 Step 2: Install Docker Engine

Remove old packages:
```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
  sudo apt-get remove $pkg;
done
```

Install Docker Engine:
```bash
sudo apt-get install ca-certificates curl
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```

Verify setup:
```bash
docker run hello-world
```

---

## 🕸️ Step 3: Initialize Docker Swarm

Initialize swarm (on master node):
```bash
docker swarm init
```

Add worker nodes:
```bash
docker swarm join --token <JOIN_TOKEN> <MASTER_IP>:2377
```

Check nodes:
```bash
docker node ls
```

Launch swarm visualizer:
```bash
docker service create --name viz --publish 8080:8080 --constraint=node.role==manager --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock alexellis2/visualizer-arm:latest
```

Visualizer UI → http://master_node_ip:8080

---

## 🔗 Step 4: Set Up GlusterFS (Persistent Storage)

Install on all nodes:
```bash
sudo apt-get install glusterfs-server -y
sudo systemctl enable --now glusterd
```

Create directory & probe peers:
```bash
sudo mkdir -p /glusterfs/distributed
sudo gluster peer probe <worker_ip>
sudo gluster pool list
```

Create replicated volume:
```bash
sudo gluster volume create gfs replica 4 node1:/glusterfs/distributed node2:/glusterfs/distributed node3:/glusterfs/distributed node4:/glusterfs/distributed force
sudo gluster volume start gfs
```

Install Docker GlusterFS Volume Plugin:
```bash
docker plugin install --alias glusterfs jmb12686/glusterfs-volume-plugin:latest --grant-all-permissions
docker plugin disable glusterfs
docker plugin set glusterfs SERVERS=<node1>,<node2>,<node3>,<node4>
docker plugin enable glusterfs
docker volume create --driver glusterfs gfs
```

---

## 🌐 Step 5: Create Docker Overlay Network

```bash
docker network create -d overlay --attachable spark
```

---

## 🔥 Step 6: Deploy Spark Cluster

**Spark Master:**
```bash
docker service create --name spark-master --network spark --constraint=node.role==manager --publish 8080:8080 --publish 7077:7077 --mount source=gfs,destination=/gfs pgiger/uzh-spark-arm bin/spark-class org.apache.spark.deploy.master.Master
```

**Spark Workers:**
```bash
docker service create --replicas 4 --name spark-worker --network spark --publish 8081:8081 --mount source=gfs,destination=/gfs pgiger/uzh-spark-arm bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077
```

---

## 🧠 Step 7: Deploy JupyterLab for PySpark

```bash
docker service create --name jupyterlab --network spark --publish 8888:8888 --mount source=gfs,destination=/gfs pgiger/uzh-spark-arm jupyter lab --ip=0.0.0.0 --allow-root --NotebookApp.token='' --NotebookApp.password='' --notebook-dir=/gfs
```

Open **JupyterLab** → http://master_node_ip:8888

---

## 🎬 Step 8: Test with MovieLens Dataset

```bash
cd /gfs
sudo apt install wget -y
wget http://files.grouplens.org/datasets/movielens/ml-20m.zip
unzip ml-20m.zip
```

---

## 💻 Step 9: Run PySpark Script

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.master("spark://spark-master:7077").appName("MovieLens").getOrCreate()

ratings = spark.read.csv("ml-20m/ratings.csv", header=True)
ratings.show(5)
ratings.printSchema()
```

---

## 🧩 Step 10: Explore RDD Operations

```python
data = spark.sparkContext.textFile("ml-20m/movies.csv")
moviesRDD = data.map(lambda l: l.split(","))
genres = moviesRDD.flatMap(lambda m: m[2].split("|"))
print(genres.take(10))
```

---

## 📊 Web Interfaces

| Service | URL |
|----------|-----|
| Spark Master | http://master_node_ip:8080 |
| Spark Worker | http://worker_node_ip:8081 |
| JupyterLab | http://master_node_ip:8888 |
| Swarm Visualizer | http://master_node_ip:8080 |

---

## 🧼 Troubleshooting

- Check Docker status:  
  ```bash
  systemctl status docker
  ```
- Check Gluster volume info:
  ```bash
  sudo gluster volume info
  ```
- Inspect overlay network:
  ```bash
  docker network inspect spark
  ```

---

## 🧾 License

This repository is distributed under the **MIT License**.  
Use and share freely for educational and research activities.

---

## 👨‍💻 Author

**Prashant Kushwaha**  
*Indian Institute of Information Technology, Lucknow*

---

### 🌟 Contributions Welcome

Submit issues or pull requests to improve performance, stability, or documentation.
