# NVIDIA DCGM exporter for Prometheus

Simple script to export metrics from [NVIDIA Data Center GPU Manager (DCGM)](https://developer.nvidia.com/data-center-gpu-manager-dcgm) to [Prometheus](https://prometheus.io/).

### Prerequisites
* NVIDIA Tesla drivers = R384+ (download from [NVIDIA Driver Downloads page](http://www.nvidia.com/drivers))
* nvidia-docker version > 2.0 (see how to [install](https://github.com/NVIDIA/nvidia-docker) and it's [prerequisites](https://github.com/nvidia/nvidia-docker/wiki/Installation-\(version-2.0\)#prerequisites))
* Optionally configure docker to set your [default runtime](https://github.com/NVIDIA/nvidia-container-runtime#daemon-configuration-file) to nvidia

### DCGM supported GPUs

Make sure all GPUs on the system are DCGM supported, otherwise the script will fail.
```
$ docker run --runtime=nvidia --rm --name=nvidia-dcgm-exporter nvidia/dcgm-exporter

# The output of dcgmi discovery and nvidia-smi should be same.

$ docker exec nvidia-dcgm-exporter dcgmi discovery -i a -v | grep -c 'GPU ID:'
$ nvidia-smi -L | wc -l
```

## Bare-metal install
```sh
# Download and install DCGM, then

$ sudo make install
$ sudo systemctl start prometheus-dcgm
```

## Container install
```sh
$ docker-compose up
```

## Deploy on kubernetes cluster
```sh
# First, set the default runtime to nvidia by editing /etc/docker/daemon.json.

# Deploy custom node-exporter with dcgm-exporter, to expose GPU metrics to Prometheus and Grafana
# Note that nodeSelector field is added to the pod spec to restrict deploying node-exporter only on GPU nodes

# Make sure to attach matching label to the GPU node
$ kubectl label nodes <gpu-node-name> hardware-type=NVIDIAGPU

# Check if the label is added
$ kubectl get nodes --show-labels

# node-exporter collecting GPU and its default metrics
$ kubectl create -f node-exporter-daemonset.yaml

# Check if node-exporter is collecting the GPU metrics successfully
$ curl -s localhost:9100/metrics | grep dcgm

# node-exporter collecting only GPU metrics
$ kubectl create -f dcgm-exporter-daemonset.yaml

# Check GPU metrics
$ curl -s localhost:9101/metrics
```

### Helm Charts

Another way to gather and visualize GPU metrics in kubernetes cluster is to use our helm charts. Find install and run instrcutions from [here](https://nvidia.github.io/gpu-monitoring-tools/).

## node-exporter

Add GPU metrics directly to node-exporter.
```sh
$ docker run -d --runtime=nvidia --rm --name=nvidia-dcgm-exporter nvidia/dcgm-exporter
$ docker run -d --rm --net="host" --pid="host" --volumes-from nvidia-dcgm-exporter:ro quay.io/prometheus/node-exporter --collector.textfile.directory="/run/prometheus"

$ curl localhost:9100/metrics

# Sample output

# HELP dcgm_gpu_temp GPU temperature (in C).
# TYPE dcgm_gpu_temp gauge
dcgm_gpu_temp{gpu="0",uuid="GPU-8f640a3c-7e9a-608d-02a3-f4372d72b323"} 34
# HELP dcgm_gpu_utilization GPU utilization (in %).
# TYPE dcgm_gpu_utilization gauge
dcgm_gpu_utilization{gpu="0",uuid="GPU-8f640a3c-7e9a-608d-02a3-f4372d72b323"} 0
# HELP dcgm_power_usage Power draw (in W).
# TYPE dcgm_power_usage gauge
dcgm_power_usage{gpu="0",uuid="GPU-8f640a3c-7e9a-608d-02a3-f4372d72b323"} 31.737
# HELP dcgm_sm_clock SM clock frequency (in MHz).
# TYPE dcgm_sm_clock gauge
dcgm_sm_clock{gpu="0",uuid="GPU-8f640a3c-7e9a-608d-02a3-f4372d72b323"} 135
# HELP dcgm_total_energy_consumption Total energy consumption since boot (in mJ).
# TYPE dcgm_total_energy_consumption counter
dcgm_total_energy_consumption{gpu="0",uuid="GPU-8f640a3c-7e9a-608d-02a3-f4372d72b323"} 7.824041e+06
# HELP dcgm_xid_errors Value of the last XID error encountered.
# TYPE dcgm_xid_errors gauge
dcgm_xid_errors{gpu="0",uuid="GPU-8f640a3c-7e9a-608d-02a3-f4372d72b323"} 0
```