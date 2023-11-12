# Logging with Loki, Promtail, and Grafana Setup Guide

This guide provides instructions for setting up a logging system using Docker, Loki, Promtail, and Grafana.

## 1. Create Docker Logging Driver Configuration

Docker logs are stored in JSON files by default. However, to ensure that logs are rotated and maintained properly, configure Docker's `daemon.json`.

Create or edit the `/etc/docker/daemon.json` file:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

This configuration sets the Docker log driver to json-file and specifies that each log file should be a maximum of 10MB, and only the last three log files should be kept.

Restart Docker to apply these settings:
```
sudo systemctl restart docker
```

## 2. Install Loki in a Docker Container

Run Loki using Docker:

```bash
sudo docker run -d --name=loki -p 3100:3100 grafana/loki:latest
```

This command will download the latest Loki image and run it as a daemon, binding port 3100 to the host.

## 3. Install Promtail on the Virtual Machine

Before running Promtail, create a configuration file named promtail-config.yml. Below is a basic example that collects logs from Docker containers:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    static_configs:
    - targets:
        - localhost
      labels:
        job: docker
        __path__: /var/lib/docker/containers/*/*.log
```

Now, run Promtail in a Docker container, ensuring to mount the appropriate volumes:

```bash
sudo docker run -d --name=promtail --restart always \
  --volume="/path/to/your/promtail-config.yml:/etc/promtail/config.yml" \
  --volume="/var/log:/var/log" \
  --volume="/var/lib/docker/containers:/var/lib/docker/containers:ro" \
  grafana/promtail:latest

```

Replace `/path/to/your/promtail-config.yml` with the actual path to your Promtail configuration file.

## 4. Configure Grafana

Finally, set up Grafana to visualize logs:

1. Install Grafana on your machine or run it inside a Docker container:

```bash
sudo docker run -d --name=grafana -p 3000:3000 grafana/grafana:latest
```

2. Access Grafana by navigating to `http://<your-vm-ip>:3000` in a web browser.

3. Log in with the default username `admin` and the password `admin`.

4. Add Loki as a data source:
    - Navigate to **Configuration > Data Sources**.
    - Click **Add data source**.
    - Choose **Loki** from the list.
    - Set the URL to `http://<loki-container-ip>:3100`, replacing `<loki-container-ip>` with the actual IP address of your Loki container.
    - Click Save & Test to ensure Grafana can communicate with Loki.

5. Create a dashboard to view logs:
    - Navigate to **+ > Create > Dashboard**.
    - Click **Add new panel**.
    - From the **Query dropdown**, select the Loki data source.
    - Construct your query to display logs as desired.
    - Click **Apply** to save the panel.

You should now be able to see your Docker logs in Grafana, sourced from Loki and collected by Promtail.

Make sure to replace placeholder values like `<your-vm-ip>` and `<loki-container-ip>` with the actual IP addresses in your setup. Adjust any file paths to match the locations on your system where you have saved your configuration files or where you wish to store logs and other data.




