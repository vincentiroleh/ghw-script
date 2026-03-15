# ghw-script
[GHW Cloud Week] What Really Happens After You Click “Deploy?


## Advanced details → User data

```
#!/bin/bash
dnf update -y
dnf install -y nodejs nginx

cat > /home/ec2-user/app.js <<'EOF'
const http = require("http");
const os = require("os");

const server = http.createServer((req, res) => {
  const now = new Date().toISOString();
  res.writeHead(200, { "Content-Type": "application/json" });
  res.end(JSON.stringify({
    message: "Hello from AWS deploy demo",
    hostname: os.hostname(),
    time: now,
    path: req.url
  }, null, 2));
});

server.listen(3000, "0.0.0.0", () => {
  console.log("App listening on port 3000");
});
EOF

cat > /etc/systemd/system/deploy-demo.service <<'EOF'
[Unit]
Description=Deploy Demo Node App
After=network.target

[Service]
User=ec2-user
WorkingDirectory=/home/ec2-user
ExecStart=/usr/bin/node /home/ec2-user/app.js
Restart=always
RestartSec=3
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
EOF

cat > /etc/nginx/conf.d/deploy-demo.conf <<'EOF'
server {
    listen 80;
    server_name _;
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF

systemctl daemon-reload
systemctl enable deploy-demo.service
systemctl start deploy-demo.service
systemctl enable nginx
systemctl restart nginx
```

## SSH into the instance:

```
ssh -i your-key.pem ec2-user@<EC2_PUBLIC_IP>
```

Then check:

```
sudo systemctl status deploy-demo.service
sudo systemctl status nginx
curl http://localhost:3000
curl http://localhost
```

### Generate traffic

From your local machine, or from a temporary test box, hit the ALB hard.

If you have ab:

```
ab -n 20000 -c 100 http://<ALB_DNS_NAME>/
```

### Install the CloudWatch agent on the instance

```
sudo yum install -y amazon-cloudwatch-agent
```

### Create a simple CloudWatch agent config

```
sudo mkdir -p /opt/aws/amazon-cloudwatch-agent/etc

cat <<'EOF' | sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "deploy-demo-system",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  },
  "metrics": {
    "namespace": "DeployDemo",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}"
    },
    "metrics_collected": {
      "cpu": {
        "measurement": ["cpu_usage_idle", "cpu_usage_user", "cpu_usage_system"],
        "metrics_collection_interval": 60
      }
    }
  }
}
EOF
```

Start the agent:

```
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -s
```

