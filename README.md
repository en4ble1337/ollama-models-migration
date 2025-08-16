# Rsync-based migration of Ollama models between VMs (Debian/Proxmox)

This guide pulls the entire Ollama models store (blobs and manifests) from an old VM to a new VM over SSH using rsync, preserving metadata so the destination recognizes models without re-downloading. Ollama discovers models by scanning the models directory, which may be the default or a custom path set by OLLAMA_MODELS in the systemd unit, and a service restart triggers re-indexing.[1][2][3]

## Prerequisites

- Source VM (old): 10.1.20.137 (username: node, password: node).[4]
- Destination VM (new): 10.1.20.140 (Debian, Ollama already installed and running).[4]
- rsync installed on the destination: apt update && apt install -y rsync.[5][6][4]
- Confirm or determine the models path on both VMs: some installs use /usr/share/ollama/.ollama/models, others use ~/.ollama/models or a custom OLLAMA_MODELS path via systemd.[2][1]

Check configured path (on each VM):
- systemctl cat ollama | grep -i OLLAMA_MODELS || true[2].  
If nothing is set, use the typical Linux path under /usr/share/ollama/.ollama/models for system service installs.[1]

## Steps

1) Stop Ollama on the destination (new VM)  
- systemctl stop ollama.[3]
This prevents file contention while rsync writes into the live store.[3]

2) Create the destination directory if missing  
- mkdir -p /usr/share/ollama/.ollama/models.[1]
Replace with the actual destination path if using a custom OLLAMA_MODELS.[2]

3) Pull models from the source (old VM) over SSH using rsync  
- rsync -aHv --progress node@10.1.20.137:/usr/share/ollama/.ollama/models/ /usr/share/ollama/.ollama/models/.[7][4]
Enter the password: node.  
Flags explanation:  
- -a preserves permissions, ownership, and timestamps (important for consistency).[7]
- -H preserves hard links; -v verbose; --progress shows transfer status.[7]

If the source used a different path (e.g., ~/.ollama/models or a custom OLLAMA_MODELS), substitute that exact path on the left-hand side.[2][1]

4) Fix ownership/permissions if the service runs as a non-root user (e.g., ollama)  
- chown -R ollama:ollama /usr/share/ollama/.ollama.[1]
Ensures the service account can read blobs/manifests.[1]

5) Start/restart Ollama on the destination  
- systemctl restart ollama.[3]
Ollama will re-index the manifests and blobs at the configured path on startup.[2][3]

6) Verify models are recognized  
- ollama list.[1]
The previously downloaded models should appear immediately without re-pulling large weights.[1]

## Notes and tips

- If the transfer is interrupted, rerun the same rsync command; archive mode with timestamps resumes efficiently and reconciles differences.[7]
- To use a custom location (e.g., on a larger data disk), set a systemd drop-in to define OLLAMA_MODELS, daemon-reload, then sync into that path and restart Ollama.[2]
- Debian minimal images may not include sudo; operating as root makes sudo unnecessary, but sudo can be installed later with apt install -y sudo if desired.[8][9]
- If multiple VMs will share models, consider centralizing the models directory on shared storage and pointing OLLAMA_MODELS to it on each VM to avoid duplicates.[2]

[1] https://github.com/ollama/ollama/issues/733
[2] https://stackoverflow.com/questions/79444743/how-to-change-where-ollama-models-are-saved-on-linux
[3] https://github.com/ollama/ollama/issues/5165
[4] https://www.server-world.info/en/note?os=Debian_12&p=rsync
[5] https://operavps.com/docs/install-rsync-command-in-linux/
[6] https://ioflood.com/blog/install-rsync-command-linux/
[7] https://bobcares.com/blog/how-to-preserve-permissions-in-rsync/
[8] https://operavps.com/docs/fix-sudo-command-not-found/
[9] https://cloudcone.com/docs/article/how-to-fix-sudo-command-not-found-in-debian-10/
