# GCSFuse Metrics
GCSFuse supports exporting custom metrics to Google cloud monitoring. 
Metrics are collected using OpenCensus and exported via Stackdriver exporter.

As of today, GCSFuse exports following metrics related to filesystem and
gcs calls.

## File system metrics:
* **fs/ops_count:** Cumulative number of operations processed by file system. It allows
grouping by op_type to get counts for individual operations. 
* **fs/ops_error_count:** Cumulative number of errors generated by file system operations.
This metric can be grouped by op_type and error_category.
Each error is mapped to an error_category in a many-to-one relationship.
* **fs/ops_latency:** Cumulative distribution of file system operation latencies. We 
can group by op_type.

## GCS metrics
* **gcs/download_bytes_count:** Cumulative number of bytes downloaded from GCS along
with read type. Read type specifies sequential or random or parallel read.
* **gcs/read_bytes_count:** Cumulative number of bytes read from GCS objects. This
is different from download_bytes_count. For eg: we might download x number of
bytes from GCS but read only <x bytes.
* **gcs/reader_count:** Cumulative number of GCS object readers opened or closed. We 
can group the data by IO Method type i.e., opened or closed. 
* **gcs/request_count:** Cumulative number of GCS requests processed. 
* **gcs/request_latencies:** Cumulative distribution of the GCS request latencies. 
* **gcs/read_count:** Specifies the count of gcs reads made along with read type. 
Read type specifies sequential or random read.

Note: Both request_count and request_latencies allows grouping by gcs method type.

## File cache metrics
* **file_cache/read_bytes_count:** The cumulative number of bytes read from file 
cache along with read type - Sequential/Random.
* **file_cache/read_latencies:** The cumulative distribution of the file cache read 
latencies along with cache hit - true/false.
* **file_cache/read_count:** Specifies the number of read requests made via file cache 
along with type - Sequential/Random and cache hit - true/false.


# Usage

## Stackdriver exporter

1. We need to set **stackdriver-export-interval** flag to enable exporting metrics to 
Google cloud monitoring. The value of this flag represents the interval with 
which data will be exported.

   Example command for exporting metrics every 60sec:
```angular2html
 gcsfuse --stackdriver-export-interval=60s <bucket_name> <directory_name>
```
2. Cloud monitoring api has to be [enabled](https://cloud.google.com/monitoring/api/enable-api) 
on the Google cloud project.
3. Service account with which the GCSFuse is running should have 
**monitoring.metricsDescriptors.create** permission.
4. Install the [Ops agent](https://cloud.google.com/monitoring/agent/ops-agent/install-index) on the VM.
5. For viewing the metrics:
    1. In the Google cloud console, go to **Metrics Explorer** page within **Monitoring**.
    2. In the toolbar, select the **Explorer** tab.
    3. Select the **configuration** tab.
    4. Expand the **Select a metric** menu. All the GCSFuse metrics will be under
   **Global > Custom > metric name**
    5. Example graph for fs/ops_count
![fs/ops_count](https://user-images.githubusercontent.com/101323867/188802087-6423f4f1-2aa6-4501-8db6-3d1997986f68.png)

## Prometheus metrics

1. Specify Prometheus port via field `metrics:prometheus-port` in configuration file, or `--prometheus-port` cli flag. The Prometheus metrics endpoint will be exposed on this port and a path of `/metrics`.

For example, use the following configuration file:

```yaml
metrics:
  prometheus-port: 8080
```

Or, use the cli flag:

```bash
gcsfuse --prometheus-port 8080 <bucket_name> <directory_name>
```

2. Run the following command to validate the Prometheus metrics endpoint is available.

```bash
curl http://localhost:8080/metrics
```

The output is similar to the following:

```text
# HELP file_cache_read_bytes_count The cumulative number of bytes read from file cache along with read type - Sequential/Random
# TYPE file_cache_read_bytes_count counter
file_cache_read_bytes_count{read_type="Random"} 0
file_cache_read_bytes_count{read_type="Sequential"} 80
# HELP file_cache_read_count Specifies the number of read requests made via file cache along with type - Sequential/Random and cache hit - true/false
# TYPE file_cache_read_count counter
file_cache_read_count{cache_hit="false",read_type="Random"} 215
file_cache_read_count{cache_hit="false",read_type="Sequential"} 5
# HELP file_cache_read_latencies The cumulative distribution of the file cache read latencies along with cache hit - true/false
# TYPE file_cache_read_latencies histogram
file_cache_read_latencies_bucket{cache_hit="false",le="1"} 215
file_cache_read_latencies_bucket{cache_hit="false",le="2"} 216
file_cache_read_latencies_bucket{cache_hit="false",le="3"} 216
file_cache_read_latencies_bucket{cache_hit="false",le="4"} 216
file_cache_read_latencies_bucket{cache_hit="false",le="5"} 216
...
file_cache_read_latencies_sum{cache_hit="false"} 483.62783500000023
file_cache_read_latencies_count{cache_hit="false"} 220
# HELP fs_ops_count The cumulative number of ops processed by the file system.
# TYPE fs_ops_count counter
fs_ops_count{fs_op="FlushFile"} 9
fs_ops_count{fs_op="GetInodeAttributes"} 91
fs_ops_count{fs_op="LookUpInode"} 584
fs_ops_count{fs_op="OpenDir"} 122
fs_ops_count{fs_op="OpenFile"} 9
fs_ops_count{fs_op="ReadDir"} 184
fs_ops_count{fs_op="ReadFile"} 220
fs_ops_count{fs_op="ReleaseDirHandle"} 122
fs_ops_count{fs_op="ReleaseFileHandle"} 9
fs_ops_count{fs_op="StatFS"} 10
# HELP fs_ops_error_count The cumulative number of errors generated by file system operations
# TYPE fs_ops_error_count counter
fs_ops_error_count{fs_error="function not implemented",fs_error_category="function not implemented",fs_op="GetXattr"} 1
fs_ops_error_count{fs_error="function not implemented",fs_error_category="function not implemented",fs_op="ListXattr"} 1
fs_ops_error_count{fs_error="interrupted system call",fs_error_category="interrupt errors",fs_op="LookUpInode"} 58
fs_ops_error_count{fs_error="no such file or directory",fs_error_category="no such file or directory",fs_op="LookUpInode"} 6
# HELP fs_ops_latency The cumulative distribution of file system operation latencies
# TYPE fs_ops_latency histogram
fs_ops_latency_bucket{fs_op="FlushFile",le="1"} 9
fs_ops_latency_bucket{fs_op="FlushFile",le="2"} 9
fs_ops_latency_bucket{fs_op="FlushFile",le="3"} 9
fs_ops_latency_bucket{fs_op="FlushFile",le="4"} 9
fs_ops_latency_bucket{fs_op="FlushFile",le="5"} 9
...
fs_ops_latency_sum{fs_op="FlushFile"} 0.28800000000000003
fs_ops_latency_count{fs_op="FlushFile"} 9
# HELP gcs_download_bytes_count The cumulative number of bytes downloaded from GCS along with type - Sequential/Random
# TYPE gcs_download_bytes_count counter
gcs_download_bytes_count{read_type="Sequential"} 2.0971528e+08
# HELP gcs_read_count Specifies the number of gcs reads made along with type - Sequential/Random
# TYPE gcs_read_count counter
gcs_read_count{read_type="Sequential"} 5
```

3. Follow [Prometheus documentation](https://prometheus.io/docs/introduction/first_steps/#configuring-prometheus)
to specify the target Prometheus metric endpoint under the `scrape_configs` section in the Prometheus configuration file.

## References:
* More details around adding custom metrics using OpenCensus can be found [here](https://cloud.google.com/monitoring/custom-metrics/open-census)