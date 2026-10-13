# Wazuh 5 to Graylog: Forward Manager Events with rsyslog

Send decoded Wazuh 5 events from the manager node to Graylog over Syslog TCP using rsyslog. No agents on the indexer, no Python.

## Key points (read first)

> From the Wazuh team (Slack, Wazuh 5.0.0-rc1):
>
> **The manager is responsible for decoding events. This is why you can forward decoded events from the manager.**
>
> **The indexer is responsible for matching rules and generating the findings. So the findings only exist on the indexer.**
>
> **It's not possible to forward the findings from the manager since they are never stored there.**

What this means:

| | Wazuh 4 | Wazuh 5 (RC) |
|---|---|---|
| Rule matching | Manager | Indexer |
| Alerts file on the manager (`alerts.json`) | Yes | No |
| Forward events from the manager | Yes | Yes (decoded events, this repo) |
| Forward alerts (findings) from the manager | Yes (`syslog_output`) | **Not possible** |
| Findings available at | Manager | Indexer only (`wazuh-findings-v5-*`) |

So this repo covers **events and manager logs from the manager node**. Alerts need a different route: see [Getting alerts into Graylog](#getting-alerts-into-graylog).

## What you get

- Full decoded Wazuh events in Graylog, with the original Windows or Sysmon event inside.
- Wazuh manager service logs (`wazuh-manager.log`, `api.log`, `cluster.log`) in Graylog.
- Disk-backed rsyslog queue, so events are kept if Graylog is briefly down.

## Tested on

- Wazuh 5.0.0-rc1, single node, Ubuntu with rsyslog 8.2312 and AppArmor
- Graylog 7.1 with a Syslog TCP input
- Windows agent with Sysmon

Not tested: multi-node clusters, other distributions, other Graylog versions. Items marked **Verify** below are not confirmed.

## How it works

```
Agents -> Wazuh manager (decodes) -> events file -> rsyslog imfile -> Syslog TCP -> Graylog
                                  -> indexer (matches rules, creates findings)  [not forwarded]
```

## Requirements

- Wazuh 5 manager node with root access
- Graylog with a **Syslog TCP** input (note its port, for example 5069) reachable from the manager
- `rsyslog` and `acl` packages

Test the connection first:

```bash
nc -vz GRAYLOG_IP GRAYLOG_PORT
```

## Step 1: Create the Graylog input

In Graylog: System, Inputs, choose **Syslog TCP**, Launch new input. Bind address `0.0.0.0`, set a port, save, and start it. Use that port in the rsyslog config.

## Step 2: Enable the Wazuh events file

Edit `/var/wazuh-manager/etc/outputs/default/file-output-integrations.yml`, set `enabled: true`, then restart the manager:

```bash
sudo systemctl restart wazuh-manager
sleep 20
sudo ls -la /var/wazuh-manager/logs/standard-wazuh-events-v5/
```

You should see `standard-wazuh-events-v5.json`.

This file output comes from a Wazuh engineer's answer in Slack and is not confirmed in the official documentation. **Verify** it on your version.

## Step 3: Let rsyslog read the files

rsyslog runs as user `syslog`, but the Wazuh folders belong to `wazuh-manager`. Do **not** run rsyslog as root. Give it read access with ACLs:

```bash
sudo apt install -y rsyslog acl
sudo setfacl -m u:syslog:x /var/wazuh-manager /var/wazuh-manager/logs
sudo setfacl -R -m u:syslog:rX /var/wazuh-manager/logs/standard-wazuh-events-v5
sudo setfacl -d -m u:syslog:rX /var/wazuh-manager/logs/standard-wazuh-events-v5
```

Check it:

```bash
sudo -u syslog head -c 200 /var/wazuh-manager/logs/standard-wazuh-events-v5/standard-wazuh-events-v5.json
```

## Step 4: Allow it in AppArmor (Ubuntu)

If rsyslog still reports "Permission denied" after the ACLs, AppArmor is blocking it:

```bash
sudo tee -a /etc/apparmor.d/local/usr.sbin.rsyslogd <<'EOT'
/var/wazuh-manager/logs/standard-wazuh-events-v5/ r,
/var/wazuh-manager/logs/standard-wazuh-events-v5/** rk,
/var/wazuh-manager/logs/ r,
/var/wazuh-manager/ r,
EOT
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.rsyslogd
```

## Step 5: Configure rsyslog

Copy `configs/60-wazuh5-graylog.conf` to `/etc/rsyslog.d/`, then replace `GRAYLOG_IP` and `GRAYLOG_PORT`:

```bash
sudo cp configs/60-wazuh5-graylog.conf /etc/rsyslog.d/
sudo nano /etc/rsyslog.d/60-wazuh5-graylog.conf
sudo rsyslogd -N1
sudo systemctl restart rsyslog
```

## Step 6: Check

```bash
sudo ss -tnp | grep GRAYLOG_PORT           # expect ESTAB
sudo journalctl -u rsyslog -n 20 --no-pager # expect no "Permission denied"
```

In Graylog, search the last 5 minutes. Every message holds the whole event as JSON in the `message` field. The timestamp shown is when Graylog received it, not when the event happened. The real event time is inside `event.original` (`SystemTime`).

## Step 7 (optional): Manager service logs

```bash
sudo setfacl -m u:syslog:r /var/wazuh-manager/logs/wazuh-manager.log /var/wazuh-manager/logs/api.log /var/wazuh-manager/logs/cluster.log
sudo setfacl -d -m u:syslog:rX /var/wazuh-manager/logs
sudo setfacl -R -m u:syslog:rX /var/wazuh-manager/logs/cluster

sudo tee -a /etc/apparmor.d/local/usr.sbin.rsyslogd <<'EOT'
/var/wazuh-manager/logs/*.log rk,
/var/wazuh-manager/logs/cluster/ r,
/var/wazuh-manager/logs/cluster/** rk,
EOT
sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.rsyslogd

sudo cp configs/61-wazuh5-manager-logs.conf /etc/rsyslog.d/
sudo rsyslogd -N1 && sudo systemctl restart rsyslog
```

rsyslog sends only lines written after it starts. In Graylog, search for text from the logs, for example `[Master]` for the cluster log or `wazuh-manager-analysisd` for the manager log.

## Getting alerts into Graylog

The manager never holds findings, so there is no manager-side way. The options, best first:

1. **Poll the indexer** with the script in [`optional-findings-forwarder/`](optional-findings-forwarder). It reads `wazuh-findings-v5-*` and sends each finding to Graylog. It reads the indexer, so use it only if that is acceptable. Experimental: the query was verified with curl, the script itself has not been run against a live indexer.
2. **Alerting webhook** from the Wazuh dashboard to a Graylog HTTP input. Not verified in 5.0 RC.
3. **Build detections in Graylog** from the decoded fields (`event.code`, `process.name`, `registry.path`, `dns.question.name`). The alerts then live in Graylog, not in the Wazuh dashboard. You must parse the JSON in the `message` field first.
4. **Use Wazuh 4** if you need alerts straight from the manager.

Check findings from the command line without putting the password in your shell history:

```bash
read -s -p "Indexer admin password: " PW; echo
curl -sk -u "admin:$PW" 'https://127.0.0.1:9200/wazuh-findings-v5-*/_count?pretty'
unset PW
```

## Multi-node clusters

Not tested. The idea: repeat Steps 2 to 6 on every manager node, and run the findings forwarder on **one** host only, so findings are not duplicated. **Verify** whether the events file setting syncs from the master to the workers, and whether each worker writes its own events file.

## Troubleshooting

| Symptom | Fix |
|---|---|
| Nothing arrives in Graylog | Use the port shown on the Graylog Inputs page, not 514 or 1514. Test with `nc -vz` |
| `Permission denied` on the file | Add the ACLs (Step 3) |
| Still denied after ACLs | Add the AppArmor rules (Step 4) |
| File not found | Enable the file output and restart the manager (Step 2) |
| Rows look identical in Graylog | They share the same agent and host prefix. Compare `wazuh.id` and `SystemTime` |
| Manager log shows `Missing root decoder` for space `custom` | A custom policy has no root decoder. Check your custom space. Not caused by this setup |

## Security notes

- Do not paste the indexer `admin` password in chat, tickets or commits. If it leaks, change it.
- Keep the findings forwarder env file at `chmod 600`.
- Keep port 9200 closed to the network unless you need it.

## License

Add the license of your choice.
