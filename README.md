0. **Create a GCP VM with 100+GB storage instead of uisng cloudshell (5GB)**:
1. **On GCP VM, Clone the repository**: 
   ```
   git clone https://github.com/fengchenLBL/gcp-gpu-dws-nersc25.git
   cd gcp-gpu-dws-nersc25
   ```

2. **Install Packer**: If Packer isn't already installed on your system, download and install it from the official HashiCorp site. Follow the instructions here: https://developer.hashicorp.com/packer/install. For example, on Ubuntu/Debian:
   ```
   wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
   echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
   sudo apt update && sudo apt install packer
   ```
   Verify installation: `packer --version`.

3. **Install the Google Compute Packer plugin**: As per the `README.md`, run:
   ```
   packer plugins install github.com/hashicorp/googlecompute
   ```

4. **Navigate to the image folder and customize the Packer template**: 
   ```
   cd image
   ```
   Edit `ubuntu2204.json` (use a text editor like nano or vim) to replace the placeholders:
   - Set `"project_id": "your-gcp-project-id"` (e.g., "my-project-123").
   - Set `"ssh_username": "your-oslogin-username"` (e.g., "your_username" – this should be your Google Cloud OS Login username).
   - The `"zone": "us-central1-b"` is already set, which matches the default zone for T4 gpus.
   If you want to pre-install additional software in the image (e.g., specific tools for the NERSC workshop), edit `install-custom.sh` in the same folder and add your bash commands there (e.g., `apt install emacs -y`).

5. **Build the custom machine image**: Run the Packer build command:
   ```
   packer build ubuntu2204.json
   ```
   This will create the custom image named "demo-u2204" in your GCP project (under Storage > Images). Monitor the output for errors; it may take 10-20 minutes. 

6. **Navigate to the blueprint folder and customize the YAML**: Go back to the root and into the blueprint folder (assuming that's where the YAMLs are):
   ```
   cd ../blueprint
   ```
   Edit `demo-u2204.yaml` to replace the placeholder:
   - Set `project_id: your-gcp-project-id` (same as above).
   - Update `zone: us-central1-a` to `zone: us-central1-b` under `vars` for alignment with your default.
   - If deployment failed: comment out the line with `zones`
   No other changes are needed unless you want to tweak node counts, GPU types, or other settings.

7. **Install Google HPC Toolkit (if not already installed)**: The blueprint uses the Google HPC Toolkit (formerly Cluster Toolkit) to deploy. 

8. **Deploy the Slurm cluster**: Use the HPC Toolkit CLI to create the deployment from the blueprint:
   ```
   cd to_where_gcluster_installed
   ./gcluster create gcp-gpu-dws-nersc25/blueprint/demo-u2204.yaml --vars project_id=your-gcp-project-id
   ./gcluster deploy demo-u2204
   ```
   (Replace with your project ID if not already set in the YAML.) This deploys the cluster, including the network, NFS homefs, GPU nodesets, partitions, and Slurm controller. It may take 10-20 minutes.

9. **Verify the deployment**: Once complete, find the controller instance in GCP Console (Compute Engine > VM instances, look for something like "demo-u2204-slurm-controller"). SSH into it.
   - Run `sinfo` to check partitions (e.g., l4dws, t4dws, v100dws) and node status.
   ```
   @demou2204-controller:~$ sinfo
   PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
   l4dws        up   infinite      4  idle~ demou2204-nsl4dws-[0-3]
   t4dws*       up   infinite      4  down# demou2204-nst4dws-[2-3,5,7]
   t4dws*       up   infinite     12  idle~ demou2204-nst4dws-[12-23]
   t4dws*       up   infinite      8   idle demou2204-nst4dws-[0-1,4,6,8-11]
   v100dws      up   infinite     16  idle~ demou2204-nsv100dws-[0-15]
   ```
   - Run `scontrol show node` to confirm nodes use the custom "demo-u2204" image.
   - Test GPUs: Submit a job like:
   ```
   @demou2204-controller:~$ srun -p t4dws -N 8 hostname
   demou2204-nst4dws-0
   demou2204-nst4dws-1
   demou2204-nst4dws-6
   demou2204-nst4dws-9
   demou2204-nst4dws-11
   demou2204-nst4dws-4
   demou2204-nst4dws-8
   demou2204-nst4dws-10
   ```
   
   ```
   @demou2204-controller:~$ srun -p t4dws -N 8 --gres=gpu:4 nvidia-smi
   ```


10. **Cleanup or iterate (optional)**: If the test works and you need to rebuild custom image for default SC25 workshop users, OOD login, Julia etc., you can destroy the deployment with `./gcluster destroy demo-u2204 --auto-approve`. For production, scale up node counts in the YAML and redeploy. If issues arise (e.g., GPU quotas), request quotas in GCP Console or try a different region/zone. If you want to switch to Ubuntu 24.04 later, repeat steps 4-9 with `ubuntu2404.json` and `demo-u2404.yaml`.

This should get test workshop cluster running...
