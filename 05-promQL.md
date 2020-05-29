# promethus数据查询语言

prometheus提供了基于HTTP API的函数表达式语言，用于时序数据的聚合和查询。

## HTTP API

当前prometheus的包"web/api/"下有v1和v2两个包，v1包为稳定版本。'api/v1'的响应体为json格式，请求成功的响应码为2xx，非2xx的响应码表示请求错误，响应体也是json格式。常见的请求失败响应码包括：

- 400 bad request， 参数错误或者丢失。
- 422 Unprocessable，请求的表达式不能被执行
- 503 Service Unavailable，查询超时或者中断

返回的json格式如下：

```
{
  "status" : "success"|"error",
  "data" : <data>,
  // if status == error
  "errorType" : "<string>",
  "error" : "<string>"
}
```

此外，请求中的输入时间戳，必须是RFC3339格式或者Unix时间戳格式，单位是秒，可选提供亚秒的精度。输出的时间戳为精确到秒的Unix时间戳格式。



## 表达式查询


查询表达式可以在瞬时量或者范围向量中执行。

### 即时查询


即时查询的http restful api：

`GET /api/v1/query/ `
参数：

- query=<string>  prometheus查询表达式
- time=<rfc3339 | unix_timestamp> 执行时间戳，可选
- timeout=<duration> 请求超时时间，可选

示例：

```
GET http://localhost:9090/api/v1/query?query=rate(pg_exporter_scrapes_total[10m] offset 2d)

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Fri, 29 Jun 2018 10:18:11 GMT

{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "instance": "localhost:9292",
          "job": "postgres"
        },
        "value": [
          1530267491.909,
          "0.06666666666666667"
        ]
      }
    ]
  }
}

Response code: 200 (OK); Time: 20ms; Content length: 167 bytes
```

### 范围查询

查询一段时间范围的数据

`GET /api/v1/query_range`
URL参数说明如下：

- query=<string>  prometheus查询表达式
- start=<rfc3339 | unix_timestamp> 
- end=<rfc3339 | unix_timestamp> 
- step=<duration> 返回值的步长、时间间隔
- timeout=<duration> 超时时间可选

示例

```
GET http://localhost:9090/api/v1/query_range?query=go_goroutines{job="prometheus"}&start=1530086038.225&end=1530086338.225&step=15

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Fri, 29 Jun 2018 10:22:51 GMT

{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
        "metric": {
          "__name__": "go_goroutines",
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "values": [
          [
            1530086038.224,
            "52"
          ],
          [
            1530086053.224,
            "53"
          ],
          [
            1530086068.224,
            "53"
          ],
          [
            1530086083.224,
            "53"
          ],
          [
            1530086098.224,
            "54"
          ],
          [
            1530086113.224,
            "54"
          ],
          [
            1530086128.224,
            "55"
          ],
          [
            1530086143.224,
            "54"
          ],
          [
            1530086158.224,
            "54"
          ],
          [
            1530086173.224,
            "54"
          ],
          [
            1530086188.224,
            "54"
          ],
          [
            1530086203.224,
            "50"
          ],
          [
            1530086218.224,
            "48"
          ],
          [
            1530086233.224,
            "47"
          ],
          [
            1530086248.224,
            "47"
          ],
          [
            1530086263.224,
            "47"
          ],
          [
            1530086278.224,
            "46"
          ],
          [
            1530086293.224,
            "46"
          ],
          [
            1530086308.224,
            "46"
          ],
          [
            1530086323.224,
            "46"
          ],
          [
            1530086338.224,
            "46"
          ]
        ]
      }
    ]
  }
}

Response code: 200 (OK); Time: 20ms; Content length: 622 bytes 
```

### 查询元数据

查询元数据的api：

`GET /api/v1/series`
查询元数据的URL参数：

- match[]=<series_selector>  选择器，一个请求至少有一个match[]参数
- start=<rfc3339 | unix_timestamp>  可选
- end=<rfc3339 | unix_timestamp>  可选

示例：

```
GET http://10.62.127.88:9090/api/v1/series?match[]=go_goroutines

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Fri, 29 Jun 2018 10:24:22 GMT

{
  "status": "success",
  "data": [
    {
      "__name__": "go_goroutines",
      "instance": "localhost:9090",
      "job": "prometheus"
    },
    {
      "__name__": "go_goroutines",
      "instance": "localhost:9100",
      "job": "node_apm_server"
    },
    {
      "__name__": "go_goroutines",
      "instance": "localhost:9104",
      "job": "pg"
    },
    {
      "__name__": "go_goroutines",
      "instance": "localhost:9292",
      "job": "postgres"
    }
  ]
}

Response code: 200 (OK); Time: 16ms; Content length: 328 bytes
```

### 查询标签值

查询标签值得api

`GET  /api/v1/label/<label_name>/values`
lable_name是要查询的标签的string，label_name如job、instance、__name__等

示例：

```
GET http://10.62.127.88:9090/api/v1/label/__name__/values

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Fri, 29 Jun 2018 10:25:27 GMT
Transfer-Encoding: chunked

{
  "status": "success",
  "data": [
    "exporter_exporter_last_scrape_duration_seconds",
    "exporter_exporter_last_scrape_error",
    "exporter_exporter_scrapes_total",
    "go_gc_duration_seconds",
    "go_gc_duration_seconds_count",
    "go_gc_duration_seconds_sum",
    "go_goroutines",
    "go_info",
    "go_memstats_alloc_bytes",
    "go_memstats_alloc_bytes_total",
    "go_memstats_buck_hash_sys_bytes",
    "go_memstats_frees_total",
    "go_memstats_gc_cpu_fraction",
    "go_memstats_gc_sys_bytes",
    "go_memstats_heap_alloc_bytes",
    "go_memstats_heap_idle_bytes",
    "go_memstats_heap_inuse_bytes",
    "go_memstats_heap_objects",
    "go_memstats_heap_released_bytes",
    "go_memstats_heap_sys_bytes",
    "go_memstats_last_gc_time_seconds",
    "go_memstats_lookups_total",
    "go_memstats_mallocs_total",
    "go_memstats_mcache_inuse_bytes",
    "go_memstats_mcache_sys_bytes",
    "go_memstats_mspan_inuse_bytes",
    "go_memstats_mspan_sys_bytes",
    "go_memstats_next_gc_bytes",
    "go_memstats_other_sys_bytes",
    "go_memstats_stack_inuse_bytes",
    "go_memstats_stack_sys_bytes",
    "go_memstats_sys_bytes",
    "go_threads",
    "http_request_duration_microseconds",
    "http_request_duration_microseconds_count",
    "http_request_duration_microseconds_sum",
    "http_request_size_bytes",
    "http_request_size_bytes_count",
    "http_request_size_bytes_sum",
    "http_requests_total",
    "http_response_size_bytes",
    "http_response_size_bytes_count",
    "http_response_size_bytes_sum",
    "net_conntrack_dialer_conn_attempted_total",
    "net_conntrack_dialer_conn_closed_total",
    "net_conntrack_dialer_conn_established_total",
    "net_conntrack_dialer_conn_failed_total",
    "net_conntrack_listener_conn_accepted_total",
    "net_conntrack_listener_conn_closed_total",
    "node_arp_entries",
    "node_boot_time_seconds",
    "node_context_switches_total",
    "node_cpu_guest_seconds_total",
    "node_cpu_seconds_total",
    "node_disk_io_now",
    "node_disk_io_time_seconds_total",
    "node_disk_io_time_weighted_seconds_total",
    "node_disk_read_bytes_total",
    "node_disk_read_time_seconds_total",
    "node_disk_reads_completed_total",
    "node_disk_reads_merged_total",
    "node_disk_write_time_seconds_total",
    "node_disk_writes_completed_total",
    "node_disk_writes_merged_total",
    "node_disk_written_bytes_total",
    "node_entropy_available_bits",
    "node_exporter_build_info",
    "node_filefd_allocated",
    "node_filefd_maximum",
    "node_filesystem_avail_bytes",
    "node_filesystem_device_error",
    "node_filesystem_files",
    "node_filesystem_files_free",
    "node_filesystem_free_bytes",
    "node_filesystem_readonly",
    "node_filesystem_size_bytes",
    "node_forks_total",
    "node_intr_total",
    "node_load1",
    "node_load15",
    "node_load5",
    "node_memory_Active_anon_bytes",
    "node_memory_Active_bytes",
    "node_memory_Active_file_bytes",
    "node_memory_AnonHugePages_bytes",
    "node_memory_AnonPages_bytes",
    "node_memory_Bounce_bytes",
    "node_memory_Buffers_bytes",
    "node_memory_Cached_bytes",
    "node_memory_CmaFree_bytes",
    "node_memory_CmaTotal_bytes",
    "node_memory_CommitLimit_bytes",
    "node_memory_Committed_AS_bytes",
    "node_memory_DirectMap1G_bytes",
    "node_memory_DirectMap2M_bytes",
    "node_memory_DirectMap4k_bytes",
    "node_memory_Dirty_bytes",
    "node_memory_HardwareCorrupted_bytes",
    "node_memory_HugePages_Free",
    "node_memory_HugePages_Rsvd",
    "node_memory_HugePages_Surp",
    "node_memory_HugePages_Total",
    "node_memory_Hugepagesize_bytes",
    "node_memory_Inactive_anon_bytes",
    "node_memory_Inactive_bytes",
    "node_memory_Inactive_file_bytes",
    "node_memory_KernelStack_bytes",
    "node_memory_Mapped_bytes",
    "node_memory_MemAvailable_bytes",
    "node_memory_MemFree_bytes",
    "node_memory_MemTotal_bytes",
    "node_memory_Mlocked_bytes",
    "node_memory_NFS_Unstable_bytes",
    "node_memory_PageTables_bytes",
    "node_memory_SReclaimable_bytes",
    "node_memory_SUnreclaim_bytes",
    "node_memory_Shmem_bytes",
    "node_memory_Slab_bytes",
    "node_memory_SwapCached_bytes",
    "node_memory_SwapFree_bytes",
    "node_memory_SwapTotal_bytes",
    "node_memory_Unevictable_bytes",
    "node_memory_VmallocChunk_bytes",
    "node_memory_VmallocTotal_bytes",
    "node_memory_VmallocUsed_bytes",
    "node_memory_WritebackTmp_bytes",
    "node_memory_Writeback_bytes",
    "node_netstat_Icmp6_InErrors",
    "node_netstat_Icmp6_InMsgs",
    "node_netstat_Icmp6_OutMsgs",
    "node_netstat_Icmp_InErrors",
    "node_netstat_Icmp_InMsgs",
    "node_netstat_Icmp_OutMsgs",
    "node_netstat_Ip6_InOctets",
    "node_netstat_Ip6_OutOctets",
    "node_netstat_IpExt_InOctets",
    "node_netstat_IpExt_OutOctets",
    "node_netstat_Ip_Forwarding",
    "node_netstat_TcpExt_ListenDrops",
    "node_netstat_TcpExt_ListenOverflows",
    "node_netstat_TcpExt_SyncookiesFailed",
    "node_netstat_TcpExt_SyncookiesRecv",
    "node_netstat_TcpExt_SyncookiesSent",
    "node_netstat_Tcp_ActiveOpens",
    "node_netstat_Tcp_CurrEstab",
    "node_netstat_Tcp_InErrs",
    "node_netstat_Tcp_PassiveOpens",
    "node_netstat_Tcp_RetransSegs",
    "node_netstat_Udp6_InDatagrams",
    "node_netstat_Udp6_InErrors",
    "node_netstat_Udp6_NoPorts",
    "node_netstat_Udp6_OutDatagrams",
    "node_netstat_UdpLite6_InErrors",
    "node_netstat_UdpLite_InErrors",
    "node_netstat_Udp_InDatagrams",
    "node_netstat_Udp_InErrors",
    "node_netstat_Udp_NoPorts",
    "node_netstat_Udp_OutDatagrams",
    "node_network_receive_bytes_total",
    "node_network_receive_compressed_total",
    "node_network_receive_drop_total",
    "node_network_receive_errs_total",
    "node_network_receive_fifo_total",
    "node_network_receive_frame_total",
    "node_network_receive_multicast_total",
    "node_network_receive_packets_total",
    "node_network_transmit_bytes_total",
    "node_network_transmit_carrier_total",
    "node_network_transmit_colls_total",
    "node_network_transmit_compressed_total",
    "node_network_transmit_drop_total",
    "node_network_transmit_errs_total",
    "node_network_transmit_fifo_total",
    "node_network_transmit_packets_total",
    "node_nf_conntrack_entries",
    "node_nf_conntrack_entries_limit",
    "node_nfs_connections_total",
    "node_nfs_packets_total",
    "node_nfs_requests_total",
    "node_nfs_rpc_authentication_refreshes_total",
    "node_nfs_rpc_retransmissions_total",
    "node_nfs_rpcs_total",
    "node_nfsd_connections_total",
    "node_nfsd_disk_bytes_read_total",
    "node_nfsd_disk_bytes_written_total",
    "node_nfsd_file_handles_stale_total",
    "node_nfsd_packets_total",
    "node_nfsd_read_ahead_cache_not_found_total",
    "node_nfsd_read_ahead_cache_size_blocks",
    "node_nfsd_reply_cache_hits_total",
    "node_nfsd_reply_cache_misses_total",
    "node_nfsd_reply_cache_nocache_total",
    "node_nfsd_requests_total",
    "node_nfsd_rpc_errors_total",
    "node_nfsd_server_rpcs_total",
    "node_nfsd_server_threads",
    "node_procs_blocked",
    "node_procs_running",
    "node_scrape_collector_duration_seconds",
    "node_scrape_collector_success",
    "node_sockstat_FRAG_inuse",
    "node_sockstat_FRAG_memory",
    "node_sockstat_RAW_inuse",
    "node_sockstat_TCP_alloc",
    "node_sockstat_TCP_inuse",
    "node_sockstat_TCP_mem",
    "node_sockstat_TCP_mem_bytes",
    "node_sockstat_TCP_orphan",
    "node_sockstat_TCP_tw",
    "node_sockstat_UDPLITE_inuse",
    "node_sockstat_UDP_inuse",
    "node_sockstat_UDP_mem",
    "node_sockstat_UDP_mem_bytes",
    "node_sockstat_sockets_used",
    "node_textfile_scrape_error",
    "node_time_seconds",
    "node_timex_estimated_error_seconds",
    "node_timex_frequency_adjustment_ratio",
    "node_timex_loop_time_constant",
    "node_timex_maxerror_seconds",
    "node_timex_offset_seconds",
    "node_timex_pps_calibration_total",
    "node_timex_pps_error_total",
    "node_timex_pps_frequency_hertz",
    "node_timex_pps_jitter_seconds",
    "node_timex_pps_jitter_total",
    "node_timex_pps_shift_seconds",
    "node_timex_pps_stability_exceeded_total",
    "node_timex_pps_stability_hertz",
    "node_timex_status",
    "node_timex_sync_status",
    "node_timex_tai_offset_seconds",
    "node_timex_tick_seconds",
    "node_uname_info",
    "node_vmstat_pgfault",
    "node_vmstat_pgmajfault",
    "node_vmstat_pgpgin",
    "node_vmstat_pgpgout",
    "node_vmstat_pswpin",
    "node_vmstat_pswpout",
    "pg_active_connections",
    "pg_all_databases_size_bytes",
    "pg_connections_active_count",
    "pg_connections_current_count",
    "pg_connections_max_count",
    "pg_current_connections",
    "pg_database_size_bytes",
    "pg_exporter_last_scrape_duration_seconds",
    "pg_exporter_last_scrape_error",
    "pg_exporter_last_scrape_error_total",
    "pg_exporter_scrapes_total",
    "pg_maximum_connections",
    "pg_table_total",
    "pg_table_total_count",
    "pg_total_database_size_bytes",
    "pg_total_tables",
    "postgresql_buffers_alloc_total",
    "postgresql_buffers_backend_fsync_total",
    "postgresql_buffers_backend_total",
    "postgresql_buffers_checkpoint_total",
    "postgresql_buffers_clean_total",
    "postgresql_buffers_maxwritten_clean_total",
    "postgresql_databases_cache_hit_ratio_percents",
    "postgresql_databases_deadlocks_total",
    "postgresql_databases_numbackends_total",
    "postgresql_databases_size_bytes",
    "postgresql_databases_temp_bytes",
    "postgresql_databases_temp_files_total",
    "postgresql_databases_tup_deleted_total",
    "postgresql_databases_tup_fetched_total",
    "postgresql_databases_tup_inserted_total",
    "postgresql_databases_tup_returned_total",
    "postgresql_databases_tup_updated_total",
    "postgresql_databases_xact_commit_total",
    "postgresql_databases_xact_rollback_total",
    "postgresql_exporter_last_scrape_duration_seconds",
    "postgresql_exporter_last_scrape_error",
    "postgresql_exporter_scrapes_total",
    "postgresql_slow_dml_queries_total",
    "postgresql_slow_queries_total",
    "postgresql_slow_select_queries_total",
    "process_cpu_seconds_total",
    "process_max_fds",
    "process_open_fds",
    "process_resident_memory_bytes",
    "process_start_time_seconds",
    "process_virtual_memory_bytes",
    "prometheus_build_info",
    "prometheus_config_last_reload_success_timestamp_seconds",
    "prometheus_config_last_reload_successful",
    "prometheus_engine_queries",
    "prometheus_engine_queries_concurrent_max",
    "prometheus_engine_query_duration_seconds",
    "prometheus_engine_query_duration_seconds_count",
    "prometheus_engine_query_duration_seconds_sum",
    "prometheus_http_request_duration_seconds_bucket",
    "prometheus_http_request_duration_seconds_count",
    "prometheus_http_request_duration_seconds_sum",
    "prometheus_http_response_size_bytes_bucket",
    "prometheus_http_response_size_bytes_count",
    "prometheus_http_response_size_bytes_sum",
    "prometheus_notifications_alertmanagers_discovered",
    "prometheus_notifications_dropped_total",
    "prometheus_notifications_queue_capacity",
    "prometheus_notifications_queue_length",
    "prometheus_rule_evaluation_duration_seconds",
    "prometheus_rule_evaluation_duration_seconds_count",
    "prometheus_rule_evaluation_duration_seconds_sum",
    "prometheus_rule_evaluation_failures_total",
    "prometheus_rule_group_duration_seconds",
    "prometheus_rule_group_duration_seconds_count",
    "prometheus_rule_group_duration_seconds_sum",
    "prometheus_rule_group_iterations_missed_total",
    "prometheus_rule_group_iterations_total",
    "prometheus_sd_azure_refresh_duration_seconds",
    "prometheus_sd_azure_refresh_duration_seconds_count",
    "prometheus_sd_azure_refresh_duration_seconds_sum",
    "prometheus_sd_azure_refresh_failures_total",
    "prometheus_sd_consul_rpc_duration_seconds",
    "prometheus_sd_consul_rpc_duration_seconds_count",
    "prometheus_sd_consul_rpc_duration_seconds_sum",
    "prometheus_sd_consul_rpc_failures_total",
    "prometheus_sd_dns_lookup_failures_total",
    "prometheus_sd_dns_lookups_total",
    "prometheus_sd_ec2_refresh_duration_seconds",
    "prometheus_sd_ec2_refresh_duration_seconds_count",
    "prometheus_sd_ec2_refresh_duration_seconds_sum",
    "prometheus_sd_ec2_refresh_failures_total",
    "prometheus_sd_file_read_errors_total",
    "prometheus_sd_file_scan_duration_seconds",
    "prometheus_sd_file_scan_duration_seconds_count",
    "prometheus_sd_file_scan_duration_seconds_sum",
    "prometheus_sd_gce_refresh_duration",
    "prometheus_sd_gce_refresh_duration_count",
    "prometheus_sd_gce_refresh_duration_sum",
    "prometheus_sd_gce_refresh_failures_total",
    "prometheus_sd_kubernetes_events_total",
    "prometheus_sd_marathon_refresh_duration_seconds",
    "prometheus_sd_marathon_refresh_duration_seconds_count",
    "prometheus_sd_marathon_refresh_duration_seconds_sum",
    "prometheus_sd_marathon_refresh_failures_total",
    "prometheus_sd_openstack_refresh_duration_seconds",
    "prometheus_sd_openstack_refresh_duration_seconds_count",
    "prometheus_sd_openstack_refresh_duration_seconds_sum",
    "prometheus_sd_openstack_refresh_failures_total",
    "prometheus_sd_triton_refresh_duration_seconds",
    "prometheus_sd_triton_refresh_duration_seconds_count",
    "prometheus_sd_triton_refresh_duration_seconds_sum",
    "prometheus_sd_triton_refresh_failures_total",
    "prometheus_target_interval_length_seconds",
    "prometheus_target_interval_length_seconds_count",
    "prometheus_target_interval_length_seconds_sum",
    "prometheus_target_scrape_pool_sync_total",
    "prometheus_target_scrapes_exceeded_sample_limit_total",
    "prometheus_target_scrapes_sample_duplicate_timestamp_total",
    "prometheus_target_scrapes_sample_out_of_bounds_total",
    "prometheus_target_scrapes_sample_out_of_order_total",
    "prometheus_target_sync_length_seconds",
    "prometheus_target_sync_length_seconds_count",
    "prometheus_target_sync_length_seconds_sum",
    "prometheus_treecache_watcher_goroutines",
    "prometheus_treecache_zookeeper_failures_total",
    "prometheus_tsdb_blocks_loaded",
    "prometheus_tsdb_compaction_chunk_range_bucket",
    "prometheus_tsdb_compaction_chunk_range_count",
    "prometheus_tsdb_compaction_chunk_range_sum",
    "prometheus_tsdb_compaction_chunk_samples_bucket",
    "prometheus_tsdb_compaction_chunk_samples_count",
    "prometheus_tsdb_compaction_chunk_samples_sum",
    "prometheus_tsdb_compaction_chunk_size_bucket",
    "prometheus_tsdb_compaction_chunk_size_count",
    "prometheus_tsdb_compaction_chunk_size_sum",
    "prometheus_tsdb_compaction_duration_seconds_bucket",
    "prometheus_tsdb_compaction_duration_seconds_count",
    "prometheus_tsdb_compaction_duration_seconds_sum",
    "prometheus_tsdb_compactions_failed_total",
    "prometheus_tsdb_compactions_total",
    "prometheus_tsdb_compactions_triggered_total",
    "prometheus_tsdb_head_active_appenders",
    "prometheus_tsdb_head_chunks",
    "prometheus_tsdb_head_chunks_created_total",
    "prometheus_tsdb_head_chunks_removed_total",
    "prometheus_tsdb_head_gc_duration_seconds",
    "prometheus_tsdb_head_gc_duration_seconds_count",
    "prometheus_tsdb_head_gc_duration_seconds_sum",
    "prometheus_tsdb_head_max_time",
    "prometheus_tsdb_head_min_time",
    "prometheus_tsdb_head_samples_appended_total",
    "prometheus_tsdb_head_series",
    "prometheus_tsdb_head_series_created_total",
    "prometheus_tsdb_head_series_not_found",
    "prometheus_tsdb_head_series_removed_total",
    "prometheus_tsdb_reloads_failures_total",
    "prometheus_tsdb_reloads_total",
    "prometheus_tsdb_retention_cutoffs_failures_total",
    "prometheus_tsdb_retention_cutoffs_total",
    "prometheus_tsdb_tombstone_cleanup_seconds_bucket",
    "prometheus_tsdb_tombstone_cleanup_seconds_count",
    "prometheus_tsdb_tombstone_cleanup_seconds_sum",
    "prometheus_tsdb_wal_corruptions_total",
    "prometheus_tsdb_wal_fsync_duration_seconds",
    "prometheus_tsdb_wal_fsync_duration_seconds_count",
    "prometheus_tsdb_wal_fsync_duration_seconds_sum",
    "prometheus_tsdb_wal_truncate_duration_seconds",
    "prometheus_tsdb_wal_truncate_duration_seconds_count",
    "prometheus_tsdb_wal_truncate_duration_seconds_sum",
    "promhttp_metric_handler_requests_in_flight",
    "promhttp_metric_handler_requests_total",
    "scrape_duration_seconds",
    "scrape_samples_post_metric_relabeling",
    "scrape_samples_scraped",
    "up"
  ]
}

Response code: 200 (OK); Time: 18ms; Content length: 14538 bytes
```

## 查询表达式

### 表达式语言的数据类型

表达式语言可以由一下数据类型的数据：

- instant vector 瞬时向量      同一时刻的metrics数据的集合，它们具有相同的时间戳
- range vector 范围向量       同一段时间metrics的集合
- scalar 标量                        简单的float浮点数值  
- string 字符串                      简单的字符串值，未使用

一次请求的结果应该只是上面四种类型中几种，例如只有instant vector类型的查询结果才可以直接用于图表呈现。

### 时序选择器

#### instant vector选择器

瞬时向量选择器允许对一组时间序列数据或者同一个时间戳下的一组采样值进行筛选，最简单的情况就只一个metric名称(表示该metric的所有时序数据)。例如

```
http_request_total{job="prometheus", group="cannary"}
```

上面的选择器指定了label为job="prometheus", group="cannary"，名称http_request_total的metric的所有时序数据。

除了"="(完全匹配)，也可以采用其他匹配操作：

- =  选择label精确地匹配给定的值
- != 选择label不等于给定的值
- =~ 正则表达式匹配给定的值
- !~ 给定的值不正则表达式匹配标签值

例如：下面的表达式筛选了时序序列http_request_total中，environment标签为staging、testing或者development，而且http方法不是GET的时间序列数据。

```
http_request_total{environment=~"staging"|"testing"|"development",method!="GET"}
```

`__name__`是prometheus内置的标签，用来标识metric的名称，`{__name__="http_request_total"}`和`http_request_total`完全等价。

#### range vector选择器

和瞬时向量选择器不同，范围向量选择器选择当前metric的一段时间时序序列。语法上使用在metric_name尾部添加[]来指定时间范围，时间单位有：

- s -> seconds
- m -> minutes
- h -> hours
- d -> days
- w -> weeks
- y -> years

例如：

```
http_request_total[5m]
```

#### 偏移修饰符

offset允许查询中修改单个的瞬时向量或者范围向量的时间偏移，例如

```
GET http://10.62.127.88:9090/api/v1/query?query=go_goroutines offset 5m

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Thu, 28 Jun 2018 12:39:00 GMT

{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "__name__": "go_goroutines",
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1530189540.478,
          "45"
        ]
      },
      {
        "metric": {
          "__name__": "go_goroutines",
          "instance": "localhost:9100",
          "job": "node_apm_server"
        },
        "value": [
          1530189540.478,
          "7"
        ]
      }
    ]
  }
}

Response code: 200 (OK); Time: 12ms; Content length: 300 bytes
```

或者

```
GET http://10.62.127.88:9090/api/v1/query?query=go_goroutines[1m] offset 5m

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Thu, 28 Jun 2018 12:40:35 GMT

{
  "status": "success",
  "data": {
    "resultType": "matrix",
    "result": [
      {
        "metric": {
          "__name__": "go_goroutines",
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "values": [
          [
            1530189277.556,
            "46"
          ],
          [
            1530189292.556,
            "46"
          ],
          [
            1530189307.556,
            "46"
          ],
          [
            1530189322.556,
            "46"
          ]
        ]
      },
      {
        "metric": {
          "__name__": "go_goroutines",
          "instance": "localhost:9100",
          "job": "node_apm_server"
        },
        "values": [
          [
            1530189277.939,
            "7"
          ],
          [
            1530189292.939,
            "7"
          ],
          [
            1530189307.939,
            "7"
          ],
          [
            1530189322.939,
            "7"
          ]
        ]
      }
    ]
  }
}

Response code: 200 (OK); Time: 24ms; Content length: 435 bytes
```

再或者

```
GET http://10.62.127.88:9090/api/v1/query?query=sum(go_goroutines offset 5m)

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Thu, 28 Jun 2018 12:41:47 GMT

{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {},
        "value": [
          1530189707.218,
          "53"
        ]
      }
    ]
  }
}

Response code: 200 (OK); Time: 21ms; Content length: 106 bytes
```

### 表达式查询函数

一些函数有默认的参数，例如：year(v=vector(time()) instant-vector)。v是参数值，instant-vector是参数类型。vector(time())是默认值。

#### `abs()`

`abs(v instant-vector)`返回输入向量的所有样本的绝对值。

#### `absent()`

`absent(v instant-vector)`，如果赋值给它的向量具有样本数据，则返回空向量；如果传递的瞬时向量参数没有样本数据，则返回不带度量指标名称且带有标签的样本值为1的结果

当监控度量指标时，如果获取到的样本数据是空的， 使用absent方法对告警是非常有用的

```
absent(nonexistent{job="myjob"}) 
# => key: value = {job="myjob"}: 1

absent(nonexistent{job="myjob", instance=~".*"}) 
# => {job="myjob"} 1 so smart !

absent(sum(nonexistent{job="myjob"})) 
# => key:value {}: 0
```

#### `ceil()`

`ceil(v instant-vector)`是一个向上舍入为最接近的整数。

#### `changes()`

`changes(v range-vector)`输入一个范围向量， 返回这个范围向量内每个样本数据值变化的次数。

#### `clamp_max()`

`clamp_max(v instant-vector, max scalar)`函数，输入一个瞬时向量和最大值，样本数据值若大于max，则改为max，否则不变

#### `clamp_min()`

`clamp_min(v instant-vector)`函数，输入一个瞬时向量和最大值，样本数据值小于min，则改为min。否则不变

#### `day_of_month()`

`day_of_month(v=vector(time()) instant-vector)`函数，返回被给定UTC时间所在月的第几天。返回值范围：1~31。

#### `day_of_week()`

`day_of_week(v=vector(time()) instant-vector)`函数，返回被给定UTC时间所在周的第几天。返回值范围：0~6. 0表示星期天。

#### `days_in_month()`

`days_in_month(v=vector(time()) instant-vector)`函数，返回当月一共有多少天。返回值范围：28~31.

#### `delta()`

delta(v range-vector)函数，计算一个范围向量v的第一个元素和最后一个元素之间的差值。返回值：key:value=度量指标：差值

下面这个表达式例子，返回过去两小时的CPU温度差：

```
delta(cpu_temp_celsius{host="zeus"}[2h])
```

delta函数返回值类型只能是gauges。

#### `deriv()`

`deriv(v range-vector)`函数，计算一个范围向量v中各个时间序列二阶导数，使用简单线性回归

deriv二阶导数返回值类型只能是gauges。

#### `exp()`

`exp(v instant-vector)`函数，输入一个瞬时向量, 返回各个样本值的e指数值，即为e^N次方。特殊情况如下所示：

```
Exp(+inf) = +Inf Exp(NaN) = NaN
```

#### `floor()`

`floor(v instant-vector)`函数，与ceil()函数相反。 4.3 为 4 。

#### `histogram_quantile()`

`histogram_quatile(φ float, b instant-vector)`函数计算histogram指标b向量的`φ-`直方图`(0 ≤ φ ≤ 1)`。

使用rate()函数为分位数计算指定时间窗口。例如：对于一个叫做`http_request_durations_seconds`的histogram指标，计算该metric过去10m内90分位数的表达式：

```
GET http://localhost:9090/api/v1/query?query=histogram_quantile%280.9%2Crate%28prometheus_http_request_duration_seconds_bucket%5B10m%5D%29%29

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Fri, 29 Jun 2018 02:59:01 GMT

{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "handler": "/graph",
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1530241141.701,
          "0.09000000000000001"
        ]
      },
      {
        "metric": {
          "handler": "/query",
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1530241141.701,
          "0.09000000000000001"
        ]
      },
      {
        "metric": {
          "handler": "/metrics",
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1530241141.701,
          "0.09000000000000001"
        ]
      },
      {
        "metric": {
          "handler": "/",
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1530241141.701,
          "0.09000000000000001"
        ]
      }
    ]
  }
}

Response code: 200 (OK); Time: 18ms; Content length: 1518 bytes
```

上面例子中，不同label的90分位数分开计算，为了聚合，可以在`rate()`外包装`sum()`函数。

```
GET http://localhost:9090/api/v1/query?query=histogram_quantile%280.9%2Csum%28rate%28prometheus_http_request_duration_seconds_bucket%5B10m%5D%29%29+by%28le%29%29

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Fri, 29 Jun 2018 03:05:29 GMT

{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {},
        "value": [
          1530241529.026,
          "0.09000000000000001"
        ]
      }
    ]
  }
}

Response code: 200 (OK); Time: 29ms; Content length: 123 bytes
```

#### `holt_winters()`

`holt_winters(v range-vector, sf scalar, tf scalar)`函数基于范围向量v，生成事件序列数据平滑值。平滑因子sf越低, 对老数据越重要。趋势因子tf越高，越多的数据趋势应该被重视。0< sf, tf <=1。 holt_winters仅用于gauges

#### `hour()`

·hour(v=vector(time()) instant-vector)·函数返回被给定UTC时间的当前第几个小时，时间范围：0~23。

#### `idelta()`

`idelta(v range-vector`函数，输入一个范围向量，返回key: value = 度量指标： 每最后两个样本值差值。

#### `increase()`

`increase(v range-vector)`函数， 输入一个范围向量，返回：key:value = 度量指标：last值-first值，自动调整单调性，如：服务实例重启，则计数器重置。与delta()不同之处在于delta是求差值，而increase返回最后一个减第一个值,可为正为负。

下面的表达式例子，返回过去5分钟，连续两个时间序列数据样本值的http请求增加值。

```
increase(http_requests_total{job="api-server"}[5m])
```

increase的返回值类型只能是counters，主要作用是增加图表和数据的可读性，使用rate记录规则的使用率，以便持续跟踪数据样本值的变化。

#### `irate`

`irate(v range-vector)`函数, 输入：范围向量，输出：key: value = 度量指标： (last值-last前一个值)/时间戳差值。它是基于最后两个数据点，自动调整单调性， 如：服务实例重启，则计数器重置。

下面表达式针对范围向量中的每个时间序列数据，返回两个最新数据点过去5分钟的HTTP请求速率。

```
GET http://localhost:9090/api/v1/query?query=irate%28net_conntrack_dialer_conn_attempted_total%5B10m%5D%29

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Fri, 29 Jun 2018 03:20:03 GMT

{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "dialer_name": "alertmanager",
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1530242403.301,
          "0"
        ]
      },
      {
        "metric": {
          "dialer_name": "postgres",
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1530242403.301,
          "0.06666666666666667"
        ]
      },
      {
        "metric": {
          "dialer_name": "prometheus",
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1530242403.301,
          "0"
        ]
      }
    ]
  }
}

Response code: 200 (OK); Time: 26ms; Content length: 662 bytes
```

irate只能用于绘制快速移动的计数器。因为速率的简单更改可以重置FOR子句，利用警报和缓慢移动的计数器，完全由罕见的尖峰组成的图形很难阅读。

#### `label_join()`

对于时序序列v中的每个时间戳，`label_join(v instant-vector, dst_label string,separator string, src_label_1 string,src_label_2 string,......)`函数将src_labels通过separator合并组成新的dst_label，然后返回带有新的label的时序序列。

```
label_join(up{job="api-server", src1="a",src2="b",src3="c"},"foo",",","src1","src2","src3")
```

#### `label_replace()`

对于v中的每个时间序列，label_replace(v instant-vector, dst_label string, replacement string, src_label string, regex string) 将正则表达式与标签值src_label匹配。如果匹配，则返回时间序列，标签值dst_label被替换的扩展替换。$1替换为第一个匹配子组，$2替换为第二个等。如果正则表达式不匹配，则时间序列不会更改。

另一种更容易的理解是：label_replace函数，输入：瞬时向量，输出：key: value = 度量指标： 值（要替换的内容：首先，针对src_label标签，对该标签值进行regex正则表达式匹配。如果不能匹配的度量指标，则不发生任何改变；否则，如果匹配，则把dst_label标签的标签纸替换为replacement 下面这个例子返回一个向量值a带有foo标签： label_replace(up{job="api-server", serice="a:c"}, "foo", "$1", "service", "(.*):.*")

#### `ln()`

`ln(v instance-vector)`计算瞬时向量v中所有样本数据的自然对数。特殊例子：

```
ln(+Inf) = +Inf 
ln(0) = -Inf 
ln(x<0) = NaN 
ln(NaN) = NaN
```

#### `log2()`

`log2(v instant-vector)`函数计算瞬时向量v中所有样本数据的二进制对数。

#### `log10()`

`log10(v instant-vector)`函数计算瞬时向量v中所有样本数据的10进制对数。相当于ln()

#### `minute()`

`minute(v=vector(time()) instant-vector)`函数返回给定UTC时间当前小时的第多少分钟。结果范围：0~59。

#### `month()`

`month(v=vector(time()) instant-vector)`函数返回给定UTC时间当前属于第几个月，结果范围：0~12。

#### `predict_linear()`

`predict_linear(v range-vector, t scalar)`预测函数，输入：范围向量和从现在起t秒后，输出：不带有度量指标，只有标签列表的结果值。

例如：

```
GET http://localhost:9090/api/v1/query?query=predict_linear%28go_memstats_lookups_total%7Bjob%3D%22prometheus%22%7D%5B5m%5D%2C60%29

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Fri, 29 Jun 2018 08:09:08 GMT

{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1530259748.14,
          "910408.5251059623"
        ]
      }
    ]
  }
}

Response code: 200 (OK); Time: 16ms; Content length: 166 bytes
```

#### `rate()`

`rate(v range-vector)`函数, 输入：范{}围向量，输出：key: value = 不带有度量指标，且只有标签列表：(last值-first值)/时间差s

`rate()`函数返回值类型只能用counters， 当用图表显示增长缓慢的样本数据时，这个函数是非常合适的。

注意：当rate函数和聚合方式联合使用时，一般先使用rate函数，再使用聚合操作, 否则，当服务实例重启后，rate无法检测到counter重置。

例如：

```
GET http://localhost:9090/api/v1/query?query=rate%28go_memstats_lookups_total%7Bjob%3D%22prometheus%22%7D%5B10m%5D%29

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Fri, 29 Jun 2018 08:13:45 GMT

{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1530260025.764,
          "1.6119658119658118"
        ]
      }
    ]
  }
}

Response code: 200 (OK); Time: 18ms; Content length: 168 bytes
```

#### `resets()`

`resets()`函数, 输入：一个范围向量，输出：key-value=没有度量指标，且有标签列表[在这个范围向量中每个度量指标被重置的次数]。在两个连续样本数据值下降，也可以理解为counter被重置。 示例：

```
GET http://localhost:9090/api/v1/query?query=resets%28go_memstats_lookups_total%7Bjob%3D%22prometheus%22%7D%5B5m%5D%29

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Fri, 29 Jun 2018 08:17:00 GMT

{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1530260220.604,
          "0"
        ]
      }
    ]
  }
}

Response code: 200 (OK); Time: 17ms; Content length: 151 bytes
```

resets只能和counters一起使用。

#### `round()`

`round(v instant-vector, to_nearest 1= scalar)`函数，与ceil和floor函数类似，输入：瞬时向量，输出：指定整数级的四舍五入值, 如果不指定，则是1以内的四舍五入。

#### `scalar()`

`scalar(v instant-vector)`函数, 输入：瞬时向量，输出：key: value = "scalar", 样本值[如果度量指标样本数量大于1或者等于0, 则样本值为NaN, 否则，样本值本身]

#### `sort()`
`sort(v instant-vector)`函数，输入：瞬时向量，输出：key: value = 度量指标：样本值[升序排列]

#### `sort_desc()`

`sort(v instant-vector`函数，输入：瞬时向量，输出：key: value = 度量指标：样本值[降序排列]

#### `sqrt()`

`sqrt(v instant-vector)`函数，输入：瞬时向量，输出：key: value = 度量指标： 样本值的平方根

#### `time()`

`time()`函数，返回从1970-01-01到现在的秒数，注意：它不是直接返回当前时间，而是时间戳

```
GET http://localhost:9090/api/v1/query?query=time%28%29

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Fri, 29 Jun 2018 08:25:58 GMT

{
  "status": "success",
  "data": {
    "resultType": "scalar",
    "result": [
      1530260758.366,
      "1530260758.366"
    ]
  }
}

Response code: 200 (OK); Time: 18ms; Content length: 94 bytes
```

#### `timestamp()`

`timestamp(v instant-vector)`返回给定的时序序列的每个采样值的时间戳。

#### `vector()`

`vector(s scalar)`函数，返回：key: value= {}, 传入参数值

#### `year()`
`year(v=vector(time()) instant-vector)`, 返回年份。

#### `_over_time()`

下面的函数列表允许传入一个范围向量，返回一个带有聚合的瞬时向量:

- `avg_over_time(range-vector)`: 范围向量内每个度量指标的平均值。
- `min_over_time(range-vector)`: 范围向量内每个度量指标的最小值。
- `max_over_time(range-vector)`: 范围向量内每个度量指标的最大值。
- `sum_over_time(range-vector)`: 范围向量内每个度量指标的求和值。
- `count_over_time(range-vector)`: 范围向量内每个度量指标的样本数据个数。
- `quantile_over_time(scalar, range-vector)`: 范围向量内每个度量指标的样本数据值分位数，`φ-quantile (0 ≤ φ ≤ 1)`
- `stddev_over_time(range-vector)`: 范围向量内每个度量指标的总体标准偏差。
- `stdvar_over_time(range-vector)`: 范围向量内每个度量指标的总体标准方差。

举例：

```
GET http://localhost:9090/api/v1/query?query=avg_over_time(go_memstats_lookups_total{job="prometheus"}[10m])

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Fri, 29 Jun 2018 08:40:39 GMT

{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1530261639.748,
          "912854.975"
        ]
      }
    ]
  }
}

Response code: 200 (OK); Time: 23ms; Content length: 160 bytes
```

### 操作符

#### 二元操作符

prometheus的查询表达式支持基本的逻辑运算和算术运算。

##### 算术二元运算符

prometheus支持一下算术二元运算符：

- +  加法 
- -  减法给结果
- *  乘法
- /  除法
- % 模
- ^  幂等

二元运算符支持标量/标量、向量/标量、向量/向量之间的运算操作。其中标量之间的结果也是标量，向量对标量的运算，操作会应用到向量每个采样值上，结果是新的向量。

向量对向量的运算需要了解向量匹配，这里不再详述。

```
GET GET http://localhost:9090/api/v1/query?query=(prometheus_tsdb_head_chunks_created_total - prometheus_tsdb_head_chunks_removed_total)

HTTP/1.1 200 OK
Access-Control-Allow-Headers: Accept, Authorization, Content-Type, Origin
Access-Control-Allow-Methods: GET, OPTIONS
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: Date
Content-Security-Policy: frame-ancestors 'self'
Content-Type: application/json
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
X-Xss-Protection: 1; mode=block
Date: Fri, 29 Jun 2018 09:46:13 GMT

{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "value": [
          1530265573.239,
          "4936"
        ]
      }
    ]
  }
}

Response code: 200 (OK); Time: 15ms; Content length: 154 bytes
```

##### 比较运算符

prometheus比较运算符包括：

- ==  等于
- !=  不等于
- >  大于
- <  小于
- >=  大于等于
- <=  小于等于

比较二元操作符被应用于`scalar/scalar`（标量/标量）、`vector/scalar`(向量/标量)，和`vector/vector`（向量/向量）。比较操作符得到的结果是bool布尔类型值，返回1或者0值。

在两个标量之间的比较运算，bool结果写入到另一个结果标量中。

瞬时向量和标量之间的比较运算，这个操作符会应用到某个当前时刻的每个时间序列数据上，如果一个时间序列数据值与这个标量比较结果是`false`，则这个时间序列数据被丢弃掉，如果是`true`, 则这个时间序列数据被保留在结果中。

在两个瞬时向量之间的比较运算，左边度量指标数据中的每个时间序列数据，与右边度量指标中的每个时间序列数据匹配，没有匹配上的，或者计算结果为false的，都被丢弃，不在结果中显示。否则将保留左边的度量指标和标签的样本数据写入瞬时向量。

##### 集合运算符

集合逻辑运算符只能运用于向量，包括：

- and 交集
- or  并集
- unless 补集

`vector1 and vector2` 逻辑/集合二元操作符，规则：vector1瞬时向量中的每个样本数据与vector2向量中的所有样本数据进行标签匹配，不匹配的，全部丢弃。运算结果是保留左边的度量指标名称和值。

`vector1 or vector2`   逻辑/集合二元操作符，规则: 保留vector1向量中的每一个元素，对于vector2向量元素，则不匹配vector1向量的任何元素，则追加到结果元素中。

`vector1 unless vector2`   逻辑/集合二元操作符，又称差积。规则：包含在vector1中的元素，但是该元素不在vector2向量所有元素列表中，则写入到结果集中。

##### 聚合操作符

Prometheus支持下面的内置聚合操作符。这些聚合操作符被用于聚合单个即时向量的所有时间序列列表，把聚合的结果值存入到新的向量中。

- sum (在维度上求和)
- max (在维度上求最大值)
- min (在维度上求最小值)
- avg (在维度上求平均值)
- stddev (求标准差)
- stdvar (求方差)
- count (统计向量元素的个数)
- count_values (统计相同数据值的元素数量)
- bottomk (样本值第k个最小值)
- topk (样本值第k个最大值)
- quantile (统计分位数)

这些操作符被用于聚合所有标签维度，或者通过without或者by子句来保留不同的维度。

```
<aggr-op>([parameter,] <vector expr>) [without | by (<label list>)] [keep_common]
```

parameter只能用于count_values, quantile, topk和bottomk。without移除结果向量中的标签集合，其他标签被保留输出。by关键字的作用正好相反，即使它们的标签值在向量的所有元素之间。keep_common子句允许保留额外的标签（在元素之间相同，但不在by子句中的标签）

count_values对每个唯一的样本值输出一个时间序列。每个时间序列都附加一个标签。这个标签的名字有聚合参数指定，同时这个标签值是唯一的样本值。每一个时间序列值是结果样本值出现的次数。

topk和bottomk与其他输入样本子集聚合不同，返回的结果中包括原始标签。by和without仅仅用在输入向量的桶中

例如： 如果度量指标名称http_requests_total包含由group, application, instance的标签组成的时间序列数据，我们可以通过以下方式计算去除instance标签的http请求总数：

```
sum(http_requests_total) without (instance)
```

如果我们对所有应用程序的http请求总数：

```
sum(http_requests_total)
```

统计每个编译版本的二进制文件数量：

```
count_values("version", build_version)
```

通过所有实例，获取http请求第5个最大值，我们可以简单地写下：

```
topk(5, http_requests_total)
```

#### 二元运算符优先级

在Prometheus系统中，二元运算符优先级从高到低：

- ^
- *, /, %
- +, -
- ==, !=, <=, <, >=, >
- and, unless
- or
