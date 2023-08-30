COMMENT 2 :

## Mapping config notification channel sử dụng consul key-value và consul template 

mapping config notification channel người dùng --> key/value trên consul --> config alertmanager

### Mô hình triển khai



<div align="center">
  <img src="https://github.com/LittleHawk03/markdown/assets/76674885/395412d3-7f2c-4d78-a958-32829adf0383">
</div>


### Thử nghiệm triển khai consul server để chạy consul key-value service

có thể deploy được consul server trên môi trường container bằng docker theo 2 cách sau 

docker run :

```sh
  docker run \                                        
    -d \
    -p 8500:8500 \
    -p 8600:8600/udp \
    --name=badger \
    consul:1.15.4 agent -server -ui -node=server-1 -bootstrap-expect=1 -client=0.0.0.0
```

docker-compose:


```yaml
    consul:
      image: consul:1.15.4
      container_name: consul_alert
      volumes:
        - ./keyvalue:/etc/tmp/
      command: 
        - "agent -server -ui -node=server-1 -bootstrap-expect=1 -client=0.0.0.0"
        - "./etc/tmp/consul_keyvalue.sh"
      ports:
        - 8500:8500
        - 8600:8600
      restart: unless-stopped
```

truy cập vào web ui của consul qua ip ``http://localhost:8500`` hoăc địa chỉ sau ``http://116.103.226.93:8500/`` (em built để thử nghiệm thôi ạ).

<div align="center">
  <img src="https://github.com/LittleHawk03/markdown/assets/76674885/a6cd12a0-769e-48f0-a8da-41230fc2c634">
</div>

**ssh** vào docker container của consul bằng lệnh `docker exec -it container_name sh` rồi chạy các lệnh sau để thêm các key-value:

```sh
  #webhook
  consul kv put alertmanager/config/webhook/alertname "tele_1"
  consul kv put alertmanager/config/webhook/url "https://telepush.dev/api/inlets/alertmanager/008a95"

  #slackconfiguration
  consul kv put alertmanager/config/slack/alertname "slack_1"
  consul kv put alertmanager/config/slack/api_url "https://hooks.slack.com/services/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"
  consul kv put alertmanager/config/slack/channel "#test"

  #mail
  consul kv put alertmanager/config/email/alertname "mail_1"
  consul kv put alertmanager/config/email/mail "manhduc030402@gmail.com"
```

sử dụng lệnh ``consul kv get alertmanager/config/email/alertname`` hoặc sử dụng web ui đển xem các key-value đã thêm.

<div align="center">
  <img src="https://github.com/LittleHawk03/markdown/assets/76674885/d3be911b-43d4-4db3-ac8c-322a298a81c5">
</div>

### Thử nghiệm dùng consul-template dynamic configuration của alertmanager

file config .hcl

```go
  consul{
    address = "116.103.226.93:8500"
  }

  exec {
    command       = "/bin/alertmanager --config.file=/etc/alertmanager/config.yml --web.listen-address=:9093 --log.level=info"

    reload_signal = "SIGHUP"
    kill_signal   = "SIGTERM"
    kill_timeout  = "15s"
  }

  template {
    source      = "/etc/alertmanager/config.yml.ctpl"
    destination = "/etc/alertmanager/config.yml"
    perms       = 0640
  }
```

file template config.yml.ctpl

```yaml
route:
  receiver: "all"
  group_interval: 30s
  repeat_interval: 30s
  routes:
    - match:
        alertname: "{{ key "alertmanager/config/webhook/alertname" }}"
      receiver: "telegram"
      repeat_interval: 1m
      continue: true

    - match:
        alertname: "{{ key "alertmanager/config/slack/alertname" }}"
      receiver: "slack"
      repeat_interval: 1m
      continue: true

    - match:
        alertname: "{{ key "alertmanager/config/email/alertname" }}"
      receiver: "email"
      repeat_interval: 1m
      continue: true
receivers:
  - name: "all"
    webhook_configs:
      - url: '{{ key "alertmanager/config/webhook/url/" }}'
        http_config:

  - name: "slack"
    slack_configs:
      - send_resolved: true
        text: "{{ .CommonAnnotations.description }}"
        username: "Prometheus"
        channel: "{{ key "alertmanager/config/slack/channel" }}"
        api_url: "{{ key "alertmanager/config/slack/api_url" }}"

  - name: "telegram"
    webhook_configs:
      - url: '{{ key "alertmanager/config/webhook/url" }}'
        http_config:

  - name: "email"
    email_configs:
      - to: '{{ key "alertmanager/config/email/mail" }}'
        from: 'goldf55f@gmail.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'goldf55f@gmail.com'
        auth_password: ''
```

chạy lại file Dockerfile đã build sẵn như trên rồi truy cập vào trong container để xem file config.yaml

<div align="center">
  <img src="https://github.com/LittleHawk03/markdown/assets/76674885/d187b57e-e415-4875-bd1f-a563e6742cf1">
</div>

``cat config.yml``

<div align="center">
  <img src="https://github.com/LittleHawk03/markdown/assets/76674885/b03445e4-0259-42bc-9e61-f8113b8d12ff">
</div>

như ta thấy các field đã được dynamic đầy đủ 


Vấn đề em đang gặp phải em muốn thực hiện các biểu thức logic trong file template tuy nhiên thì nó đang bế tắc anh xem qua hộ em với ạ
