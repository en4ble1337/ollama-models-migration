# Rsync-based migration of Ollama models between VMs (Debian/Proxmox)

This guide pulls the entire Ollama models store (blobs and manifests) from an old VM to a new VM over SSH using rsync, preserving metadata so the destination recognizes models without re-downloading. Ollama discovers models by scanning the models directory, which may be the default or a custom path set by OLLAMA_MODELS in the systemd unit; a service restart triggers re-indexing.

Assumptions:
- Source VM (old): 10.1.20.137 (username: node, password: node)
- Destination VM (new): 10.1.20.140 (Debian, Ollama already installed and running)
- Default models path: /usr/share/ollama/.ollama/models
- Run these commands on the destination (new VM)

## 1) Install rsync on Debian (if not already installed)
```bash
apt update
apt install -y rsync
```

## 2) Identify/confirm models paths (optional but recommended)
Check if a custom OLLAMA_MODELS is configured:
```bash
systemctl cat ollama | grep -i OLLAMA_MODELS || true
```

If nothing is set, use the default path /usr/share/ollama/.ollama/models.

## 3) Stop Ollama on the destination (new VM) to avoid contention
```bash
systemctl stop ollama
```

## 4) Ensure the destination directory exists
```bash
mkdir -p /usr/share/ollama/.ollama/models
```

If using a custom location, replace the path accordingly.

## 5) Pull models from the source (old VM) over SSH using rsync
Replace the left-hand source path if your old VM uses a different models path (e.g., ~/.ollama/models).
```bash
rsync -aHv --progress node@10.1.20.137:/usr/share/ollama/.ollama/models/ /usr/share/ollama/.ollama/models/
```

Notes:
- -a preserves permissions/ownership/timestamps
- -H preserves hard links
- --progress shows transfer progress
- Re-run the same command if interrupted; rsync will efficiently resume

## 6) Fix ownership/permissions if the service runs as a non-root user
If Ollama runs as user ollama:
```bash
chown -R ollama:ollama /usr/share/ollama/.ollama
```

## 7) Start/restart Ollama on the destination
```bash
systemctl restart ollama
```

## 8) Verify models are recognized
```bash
ollama list
```

## Optional: Using a custom models path
If placing models on a larger data disk (e.g., /data/ollama-models), set a systemd drop-in before restarting:
```bash
mkdir -p /etc/systemd/system/ollama.service.d
cat >/etc/systemd/system/ollama.service.d/override.conf <<'EOF'
[Service]
Environment="OLLAMA_MODELS=/data/ollama-models"
EOF
systemctl daemon-reload
mkdir -p /data/ollama-models
rsync -aHv --progress node@10.1.20.137:/usr/share/ollama/.ollama/models/ /data/ollama-models/
chown -R ollama:ollama /data/ollama-models
systemctl restart ollama
ollama list
```

That’s it—no re-downloading required; the new VM should immediately recognize the transferred models.
